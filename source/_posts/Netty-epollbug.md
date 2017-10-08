---
title: Netty源码分析 解决NIO的epoll死循环bug
date: 2017-03-20 20:00:00
categories: Netty
tags: [NIO, Netty, 源码]
---

> 本文使用`netty-4.1.5.Final`版本源码进行分析

JDK的NIO类库有一个epoll死循环bug，它会导致Selector空轮询，IO线程CPU达到100%，严重影响系统运行。netty从api使用层面对该bug进行了规避解决，下面看下netty的解决策略并从源码了解其具体实现。

### Netty的解决策略：

1. 对Selector的select操作周期进行统计。
2. 每完成一次空的select操作进行一次计数。
3. 在某个周期内如果连续N次空轮询，则说明触发了JDK NIO的epoll死循环bug。
4. 创建新的Selector，将出现bug的Selector上的channel重新注册到新的Selector上。
5. 关闭bug的Selector，使用新的Selector进行替换。


### 源码解析：

netty中的select操作，如果有被选择出来的channel，或者有task任务，或者有定时任务task，或者需要wakenUp，或者超时，将会退出当前select操作。该逻辑识别并规避解决了epoll死循环bug：

```java
    private void select(boolean oldWakenUp) throws IOException {
        Selector selector = this.selector;
        try {
            int selectCnt = 0;
            long currentTimeNanos = System.nanoTime();//当前时间
            long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);//当前时间+下一个定时task的延迟时间得到当前select操作的最晚时间点，如果没有定时task则取默认1秒
            for (;;) {
                long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;//转换成毫秒并增加0.5毫秒调整值处理精度损失
                if (timeoutMillis <= 0) {//到达超时时间则退出
                    if (selectCnt == 0) {//如果是第一次则需要执行一次selectNow操作再退出
                        selector.selectNow();
                        selectCnt = 1;
                    }
                    break;
                }

                // If a task was submitted when wakenUp value was true, the task didn't get a chance to call
                // Selector#wakeup. So we need to check task queue again before executing select operation.
                // If we don't, the task might be pended until select operation was timed out.
                // It might be pended until idle timeout if IdleStateHandler existed in pipeline.
                if (hasTasks() && wakenUp.compareAndSet(false, true)) {//检查是否有task任务并且wakenUp可以被设置为true，则需要执行一次selectNow操作再退出
                    selector.selectNow();
                    selectCnt = 1;
                    break;
                }

                int selectedKeys = selector.select(timeoutMillis);//进行一次select(timeoutMillis)操作并增加selectCnt计数
                selectCnt ++;

                if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {//如果有选择出来的selectedKeys，或者需要wakenUp，或者task队列有任务，或者定时task队列有任务，则可以退出
                    // - Selected something,
                    // - waken up by user, or
                    // - the task queue has a pending task.
                    // - a scheduled task is ready for processing
                    break;
                }
                if (Thread.interrupted()) {//如果当前线程被中断则退出
                    // Thread was interrupted so reset selected keys and break so we not run into a busy loop.
                    // As this is most likely a bug in the handler of the user or it's client library we will
                    // also log it.
                    //
                    // See https://github.com/netty/netty/issues/2426
                    if (logger.isDebugEnabled()) {
                        logger.debug("Selector.select() returned prematurely because " +
                                "Thread.currentThread().interrupt() was called. Use " +
                                "NioEventLoop.shutdownGracefully() to shutdown the NioEventLoop.");
                    }
                    selectCnt = 1;
                    break;
                }

                long time = System.nanoTime();
                if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {//超时则将selectCnt置为1
                    // timeoutMillis elapsed without anything selected.
                    selectCnt = 1;
                } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                        selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {//如果select操作周期计数值selectCnt超出io.netty.selectorAutoRebuildThreshold设置的阈值，默认512次，说明在该周期内发生了空轮询，触发了JDK NIO的epoll死循环bug。
                    // The selector returned prematurely many times in a row.
                    // Rebuild the selector to work around the problem.
                    logger.warn(
                            "Selector.select() returned prematurely {} times in a row; rebuilding Selector {}.",
                            selectCnt, selector);

                    rebuildSelector();//重建Selector
                    selector = this.selector;

                    // Select again to populate selectedKeys.
                    selector.selectNow();对新的selector执行一次selectNow操作然后退出选择操作
                    selectCnt = 1;
                    break;
                }

                currentTimeNanos = time;//一次for循环结束后重置currentTimeNanos
            }

            if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                            selectCnt - 1, selector);
                }
            }
        } catch (CancelledKeyException e) {
            if (logger.isDebugEnabled()) {
                logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                        selector, e);
            }
            // Harmless exception - log anyway
        }
    }
```

重建Selector。当发生epoll bug，则创建一个新的Selector，将出现bug的Selector上的channel重新注册到新的Selector上，关闭bug的Selector，使用新的Selector进行替换：

```java
    public void rebuildSelector() {
        if (!inEventLoop()) {//如果是非当前EventLoop执行rebuild，则将rebuild操作封装成task，交由对应的EventLoop线程执行rebuild操作
            execute(new Runnable() {
                @Override
                public void run() {
                    rebuildSelector();
                }
            });
            return;
        }

        final Selector oldSelector = selector;
        final Selector newSelector;

        if (oldSelector == null) {
            return;
        }

        try {
            newSelector = openSelector();//打开一个新的Selector
        } catch (Exception e) {
            logger.warn("Failed to create a new Selector.", e);
            return;
        }

        // Register all channels to the new Selector.
        int nChannels = 0;
        for (;;) {
            try {
                for (SelectionKey key: oldSelector.keys()) {//遍历所有老的Selector上的channel，重新注册到新的Selector上
                    Object a = key.attachment();
                    try {
                        if (!key.isValid() || key.channel().keyFor(newSelector) != null) {//SelectionKey无效或者已经注册上了则跳过
                            continue;
                        }

                        int interestOps = key.interestOps();
                        key.cancel();//取消SelectionKey
                        SelectionKey newKey = key.channel().register(newSelector, interestOps, a);//将SelectionKey中的channel注册到新的Selector上
                        if (a instanceof AbstractNioChannel) {
                            // Update SelectionKey
                            ((AbstractNioChannel) a).selectionKey = newKey;//将AbstractNioChannel的成员变量selectionKey赋新值
                        }
                        nChannels ++;
                    } catch (Exception e) {
                        logger.warn("Failed to re-register a Channel to the new Selector.", e);
                        if (a instanceof AbstractNioChannel) {
                            AbstractNioChannel ch = (AbstractNioChannel) a;
                            ch.unsafe().close(ch.unsafe().voidPromise());
                        } else {
                            @SuppressWarnings("unchecked")
                            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                            invokeChannelUnregistered(task, key, e);
                        }
                    }
                }
            } catch (ConcurrentModificationException e) {
                // Probably due to concurrent modification of the key set.
                continue;
            }

            break;
        }

        selector = newSelector;//用新的Selector替换老的Selector

        try {
            // time to close the old selector as everything else is registered to the new one
            oldSelector.close();//关闭老的Selector
        } catch (Throwable t) {
            if (logger.isWarnEnabled()) {
                logger.warn("Failed to close the old Selector.", t);
            }
        }

        logger.info("Migrated " + nChannels + " channel(s) to the new Selector.");
    }
```

### 总结

如果发生epoll死循环bug，那么当前I/O线程将停下来进行bug的修复，然后再继续进行逻辑处理，bug的修复是非阻塞操作，处理速度非常快；而且netty线程池中一般设置有多个I/O线程，其中某个I/O线程中的Selector触发bug并不会影响其他I/O线程运行，所以netty通过这种策略，在几乎不影响性能的情况下从api使用层面规避解决了该bug。


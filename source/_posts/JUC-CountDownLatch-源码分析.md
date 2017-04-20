---
title: JUC - CountDownLatch 源码分析
date: 2016-10-11 10:58:39
categories: Concurrent
tags: [Java,并发,源码]
---
CountDownLatch，一个同步辅助类，在完成一组正在其他线程中执行的操作之前，它允许一个或多个线程一直等待。

<!--more-->

# 源码分析

CountDownLatch的实现方式是在内部定义了一个实现**AbstractQueuedSynchronizer**（详见：[JUC - AbstractQueuedSynchronizer(AQS) 源码分析](https://kris-liu.github.io/2016/09/28/JUC-AbstractQueuedSynchronizer-AQS-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)）的**内部类Sync**，Sync主要实现了AbstractQueuedSynchronizer中共享模式的获取和释放方法tryAcquireShared和tryReleaseShared，在CountDownLatch中使用AQS的子类Sync，初始化的state表示一个计数器，每次countDown的时候计数器会减少1，直到减为0的时候或超时或中断，await方法从等待中返回。


```
	private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {//初始化计数器数量，不可重置
            setState(count);
        }

        int getCount() {//获取当前计数器数量
            return getState();
        }

        protected int tryAcquireShared(int acquires) {//await方法会调用，当前计数器为0是才返回，不为0时，会挂起当前线程，直到计数器减少为0的时候，才被countDown方法依次唤醒
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {//countDown方法会调用，每次调用都会使计数器减1，直到减少为0的时候，会返回true，依次唤醒等待中的线程，使从await方法返回。
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
```

```
	public void await() throws InterruptedException {//该方法等待计数器减少为0，await系列方法分别调用AQS的共享模式的acquire系列方法
        sync.acquireSharedInterruptibly(1);
    }
```

```
	public void countDown() {//该方法减少计数器，调用AQS的共享模式的release方法
        sync.releaseShared(1);
    }
```

# 使用方式

主线程等待子线程都执行完任务后才返回。

```
public class CountDownLatchTest {
    public static void main(String[] args) throws InterruptedException {
        final CountDownLatch c = new CountDownLatch(3);
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                c.countDown();
                System.out.println("countDown 1000 : " + c.getCount());
            }
        });
        thread.start();
        Thread thread2 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                c.countDown();
                System.out.println("countDown 2000 : " + c.getCount());
            }
        });
        thread2.start();
        Thread thread3 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                c.countDown();
                System.out.println("countDown 3000 : " + c.getCount());
            }
        });
        thread3.start();
        System.out.println("await before : " + c.getCount());
        c.await();
        System.out.println("await after : " + c.getCount());
    }
}
```
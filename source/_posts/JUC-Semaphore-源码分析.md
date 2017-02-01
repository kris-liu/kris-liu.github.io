---
title: JUC - Semaphore 源码分析
date: 2016-10-08 16:58:30
categories: Concurrent
tags: [Java,并发,源码]
---
**Semaphore，信号量。用于控制同时访问特定资源的线程数量，来保证合理的使用特定资源。**比如：有10个数据库连接，有30个线程都需要使用连接，Semaphore可以控制只有10个线程能够获取连接，其他线程需要排队等待，当已经获取到连接的线程释放连接，排队的线程才能够去申请获取。

<!--more-->

# 源码分析
Semaphore的实现方式是在内部定义了一个实现**AbstractQueuedSynchronizer**（详见：[JUC - AbstractQueuedSynchronizer(AQS) 源码分析](https://kris-liu.github.io/2016/09/28/JUC-AbstractQueuedSynchronizer-AQS-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)）的**内部类Sync**，Sync主要实现了AbstractQueuedSynchronizer中共享模式的获取和释放方法tryAcquireShared和tryReleaseShared，在Semaphore中使用AQS的子类Sync，初始化的state表示许可数，在每一次请求acquire()一个许可都会导致state减少1，同样每次释放一个许可release()都会导致state增加1。一旦达到了0，新的许可请求线程将被挂起，直到有许可被释放。

```
	abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1192457210091910933L;

        Sync(int permits) {//Sync的构造方法初始化可用许可数量
            setState(permits);
        }

        final int getPermits() {
            return getState();//state代表当前总共可用许可数量
        }

        final int nonfairTryAcquireShared(int acquires) {//非公平的获取许可，代表新插入的线程，可以和同步队列中的节点竞争获取锁
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }

        protected final boolean tryReleaseShared(int releases) {//释放许可，每次释放，通过循环和CAS保证并发时也可以安全的释放
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }

        final void reducePermits(int reductions) {//减少一定数量的许可
            for (;;) {
                int current = getState();
                int next = current - reductions;
                if (next > current) // underflow
                    throw new Error("Permit count underflow");
                if (compareAndSetState(current, next))
                    return;
            }
        }

        final int drainPermits() {//清空可用许可
            for (;;) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
    }
```

**公平与非公平**的实现，主要区别就是在获取许可时，公平实现会检查同步队列是否有线程处在等待，有则获取失败进入同步队列中去等待，非公平实现则不会检查，新插入的线程可以和队列中等待最久的线程一起竞争锁的使用，非公平是默认的实现，因为减少了线程挂起和释放，线程上下文切换的开销，性能好，缺点是有可能造成锁饥饿，队列中的线程迟迟无法获取到锁。

```
	static final class NonfairSync extends Sync {//非公平
        private static final long serialVersionUID = -2694183684443567898L;

        NonfairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
    }

	final int nonfairTryAcquireShared(int acquires) {
         for (;;) {
             int available = getState();
             int remaining = available - acquires;
             if (remaining < 0 ||
                 compareAndSetState(available, remaining))
                 return remaining;
         }
	}
```

```
	static final class FairSync extends Sync {//公平
        private static final long serialVersionUID = 2014338818796000944L;

        FairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())//检查同步队列中是否有等待的节点
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }
    
	public final boolean hasQueuedPredecessors() {
	    Node t = tail; 
	    Node h = head;
	    Node s;
	    return h != t &&
	        ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```

# 使用方式

```
public class SemaphoreTest {

    public static void main(String[] args) {
        final Semaphore semaphore = new Semaphore(2);
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        for (int i = 0; i < 10; i++) {
            executorService.execute(new Runnable() {
                public void run() {
                    try {
                        semaphore.acquire();
                        try {
                            System.out.println("线程:" + Thread.currentThread().getName() + "获得许可");
                            TimeUnit.SECONDS.sleep(1);//访问特定资源
                        } finally {
                            semaphore.release();
                            System.out.println("剩余许可：" + semaphore.availablePermits());
                        }
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
        executorService.shutdown();
    }
}

```





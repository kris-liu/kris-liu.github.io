---
title: JUC - ReentrantLock 源码分析
date: 2016-10-11 18:05:50
categories: Concurrent
tags: [Java,并发,锁,源码]
---

ReentrantLock，一个可重入的独占锁 Lock，它具有与使用 synchronized 方法和语句所访问的隐式监视器锁相同的一些基本行为和语义，但功能更强大。

<!--more-->


# 源码分析
ReentrantLock的实现方式是在内部定义了一个实现**AbstractQueuedSynchronizer**（详见：[JUC - AbstractQueuedSynchronizer(AQS) 源码分析](https://kris-liu.github.io/2016/09/28/JUC-AbstractQueuedSynchronizer-AQS-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)）的**内部类Sync**，Sync主要实现了AbstractQueuedSynchronizer中独占模式的获取和释放方法tryAcquire和tryRelease，在ReentrantLock中使用AQS的子类Sync，AQS的status代表锁是否被占用，为0代表没有被占用，大于0代表被当前线程占用的次数，每次占用必需要对应一次释放，ReentrantLock是可重入的锁，一个线程可以同时多次获取到锁。



```
	abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

        abstract void lock();//主要为了子类可以尝试快速非公平的获取锁

        final boolean nonfairTryAcquire(int acquires) {//非公平尝试获取锁，公平与非公平实现的trylock方法都会调用这个来尝试非公平的获取一次
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {//当前状态为0代表锁没有被获取，CAS设置状态，成功后setExclusiveOwnerThread设置持有锁的线程
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {//如果锁被占用，查看占用锁的线程是否是当前线程，当前线程可以再次获取锁
                int nextc = c + acquires;
                if (nextc < 0) //如果负值则快速失败
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

        protected final boolean tryRelease(int releases) {//释放锁
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())//释放锁的当前线程必须是持有锁的线程，否则无法释放
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {//因为可重入，所以状态为0时才需要setExclusiveOwnerThread(null)清空持有锁的线程，并且返回true，返回true会在release方法触发唤醒等待锁的线程。只有当前线程完全释放锁，其他线程才可以去得到锁
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }

        protected final boolean isHeldExclusively() {//判断当前线程是否是持有锁的线程
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        final ConditionObject newCondition() {//Condition是AQS的等待队列，类似Object的wait和notify，在下一节分析
            return new ConditionObject();
        }
        
        final Thread getOwner() {//获取当前持有锁的线程
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }

        final int getHoldCount() {//获取重入持有锁的次数
            return isHeldExclusively() ? getState() : 0;
        }

        final boolean isLocked() {//获取当前是否被锁
            return getState() != 0;
        }

        /**
         * Reconstitutes this lock instance from a stream.
         * @param s the stream
         */
        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {//反序列化需要重置锁状态为0
            s.defaultReadObject();
            setState(0); // reset to unlocked state
        }
    }
```

ReentrantLock的实现有公平和非公平两种，主要区别就是在获取锁时，公平实现会检查同步队列是否有线程处在等待，有则获取失败进入同步队列中去等待，非公平实现则不会检查，新插入的线程可以和队列中等待最久的线程一起竞争锁的使用，非公平是默认的实现，因为减少了线程挂起和释放，线程上下文切换的开销，性能好，缺点是有可能造成锁饥饿，队列中的线程迟迟无法获取到锁。

```
	static final class NonfairSync extends Sync {//非公平实现
        private static final long serialVersionUID = 7316153563782823691L;

        final void lock() {
            if (compareAndSetState(0, 1))//非公平实现会尝试快速获取一次，获取失败则调用acquire调用非公平的实现正常获取。
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);//非公平的去尝试获取锁
        }
    }
```

```
	static final class FairSync extends Sync {//公平实现
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {//调用AQS的acquire()方法
            acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {//公平尝试获取锁
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {//为0代表可以获取锁
                if (!hasQueuedPredecessors() && //公平和非公平的主要区别，检查是否队列中没有等待的线程
                    compareAndSetState(0, acquires)) {//队列中没有等待的线程才尝试CAS修改锁状态，目的是保证先来获取的一定先获取到。
                    setExclusiveOwnerThread(current);//成功则设置当前线程为锁持有线程
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {//如果锁被占用，查看占用锁的线程是否是当前线程，当前线程可以再次获取锁
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
```

# 使用方式
注意：unlock必需在finally块中，以保证锁的释放；lock必需在try{}finally外面，防止未获取到锁仍然做额外的释放。

```
public class ReentrantLockTest {

    public static void main(String[] args) {
        ReentrantLock lock = new ReentrantLock();
        lock.lock();
        try {
            Thread.sleep(1000l);//独占逻辑
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

}
```
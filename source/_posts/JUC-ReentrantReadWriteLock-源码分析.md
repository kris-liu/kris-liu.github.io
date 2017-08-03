---
title: JUC - ReentrantReadWriteLock 源码分析
date: 2016-10-16 02:09:04
categories: Concurrent
tags: [Java,并发,读写锁,源码]
---

ReentrantReadWriteLock，读写锁。维护了一对相关的锁，一个用于只读操作，另一个用于写入操作。只要没有 writer，读取锁可以由多个 reader 线程同时保持。写入锁是独占的。

与互斥锁相比，读-写锁允许对共享数据进行更高级别的并发访问。虽然一次只有一个线程（writer 线程）可以修改共享数据，但在许多情况下，任何数量的线程可以同时读取共享数据（reader 线程）。当访问读写比恰当的共享数据时，使用读-写锁所允许的并发性将带来更大的性能提高。

<!--more-->

# 源码分析
ReentrantReadWriteLock的实现方式是在内部定义了一个实现**AbstractQueuedSynchronizer**（详见：[JUC - AbstractQueuedSynchronizer(AQS) 源码分析](https://kris-liu.github.io/2016/09/28/JUC-AbstractQueuedSynchronizer-AQS-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)）的**内部类Sync**，Sync同时实现了AbstractQueuedSynchronizer中独占模式的获取和释放方法tryAcquire和tryRelease，和共享模式的获取和释放方法tryAcquireShared和tryReleaseShared，**写锁WriteLock**使用独占模式的方法控制锁状态，**读锁ReadLock**使用共享模式的方法控制锁状态，在WriteLock和ReadLock中使用同一个AQS的子类Sync，用AQS的status代表读写锁的状态计数，单个int值，通过位运算区分高低位，低16位代表写状态，高16位代表读状态。支持公平非公平实现，支持中断，支持重入，支持锁降级。

### 当并发读写时：

1. 当有线程获取了独占锁，那么后续所有其他线程的独占和共享锁请求会加入同步队列等待，后续当前线程的独占和共享锁可以再次获取；

2. 当有线程获取了共享锁，那么后续所有线程的独占锁请求会加入同步队列，后续所有线程的共享锁请求可以继续获取锁；

3. 当独占锁完全释放时，会唤醒后继节点，当唤醒的是共享节点时，会传播向后唤醒后继的共享节点；

4. 当共享锁完全释放时，且当前没有持有独占锁，会唤醒后继节点，当唤醒的是共享节点时，会传播向后唤醒后继的共享节点；

5. 当当前线程已经获取独占锁，那么当前线程可以继续获取共享锁，当独占锁退出时，锁降级为共享锁；

6. 一个线程可以同时进入多次共享锁或独占锁；


## Sync

```java
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 6317671515068378041L;

        static final int SHARED_SHIFT   = 16;//32位分高16和低16位
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);//0000000000000001|0000000000000000
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;//0000000000000000|1111111111111111
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;//0000000000000000|1111111111111111

        static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }//获取共享锁计数，高16位
        static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }//获取独占锁计数，低16位

        static final class HoldCounter {//每个线程持有的读锁计数。写锁独占所以写锁的低16位就代表独占线程重入计数，读锁共享，所以读锁的高16代表所有线程所有重入次数的计数，HoldCounter用ThreadLocal保存每个线程自己的重入计数。
            int count = 0;
            final long tid = Thread.currentThread().getId();
        }

        static final class ThreadLocalHoldCounter
            extends ThreadLocal<HoldCounter> {
            public HoldCounter initialValue() {//初始化
                return new HoldCounter();
            }
        }
        
        private transient ThreadLocalHoldCounter readHolds;//保存每个线程持有的读锁计数，不参与序列化。

        private transient HoldCounter cachedHoldCounter;//缓存最新获取共享锁的线程的HoldCounter

        private transient Thread firstReader = null;//缓存第一个获取共享锁的线程
        private transient int firstReaderHoldCount;//缓存第一个获取共享锁的线程的重入计数

        Sync() {
            readHolds = new ThreadLocalHoldCounter();
            setState(getState()); //用volatile的读写保证readHolds的可见性，保证readHolds对所有线程可见。
        }

        abstract boolean readerShouldBlock();//共享锁获取是否需要阻塞。控制共享模式的获取锁操作是否公平，子类实现。公平实现会看当前同步队列是否有有效的等待节点，有则返回true，没有则返回false，直接尝试获取锁；非公平实现查看队列中的下一个节点是否是独占模式，是独占模式，则需要阻塞，避免写锁的获取总是发生饥饿，否则，可以直接尝试获取锁。
        
        abstract boolean writerShouldBlock();//独占锁获取是否需要阻塞。控制独占模式的获取锁操作是否公平，子类实现。公平实现会看当前同步队列是否有有效的等待节点，有则返回true，需要加入同步队列，顺序获取，没有则返回false，直接尝试获取锁；非公平实现返回false，代表不需要等待可以直接尝试获取锁

        protected final boolean tryRelease(int releases) {//独占模式的释放锁
            if (!isHeldExclusively())//如果当前线程不是持有独占锁的线程，将抛出异常
                throw new IllegalMonitorStateException();
            int nextc = getState() - releases;
            boolean free = exclusiveCount(nextc) == 0;//exclusiveCount方法检查释放后的状态的低16位，是0则独占锁完全释放，设置当前持有独占锁的线程位null
            if (free)
                setExclusiveOwnerThread(null);
            setState(nextc);//因为是独占的释放，所以直接set不会有问题。写volatile，之前的所有内存操作会对所有线程可见，释放之后其他线程才能获取锁。
            return free;//完全释放独占锁才返回true，这里有可能是锁降级，或者是读写锁完全释放。
        }

        protected final boolean tryAcquire(int acquires) {//独占模式的获取锁
            Thread current = Thread.currentThread();//获取当前线程
            int c = getState();//获取当前状态
            int w = exclusiveCount(c);//获取写状态
            if (c != 0) {//状态不是0，则当前锁已被获取
                if (w == 0 || current != getExclusiveOwnerThread())//状态不为0，写状态为0，说明读状态不为0。即读锁被获取或者当前线程不是持有独占锁的线程，获取独占锁失败，只有重入的独占锁才能获取
                    return false;
                if (w + exclusiveCount(acquires) > MAX_COUNT)//写状态超出16位的最大值则失败抛出异常
                    throw new Error("Maximum lock count exceeded");
                setState(c + acquires);//执行到这里说明状态不为0且写状态不为0，说明独占锁已经被获取，且当前线程是持有独占锁的线程，该获取操作是重入获取独占锁
                return true;
            }
            //当前状态为0
            if (writerShouldBlock() //写锁是否需要等待，由子类实现，公平实现会看当前同步队列是否有有效的等待节点，有则返回true，需要加入同步队列，顺序获取，没有则返回false，直接尝试获取锁；非公平实现返回false，代表不需要等待可以直接尝试获取锁
	            || !compareAndSetState(c, c + acquires))//或CAS更新状态失败
                return false;//不可以直接获取或者CAS更新状态失败，返回false
            setExclusiveOwnerThread(current);
            return true;//可以获取锁且CAS更新状态成功，设置当前线程为持有独占锁的线程，返回true
        }

        protected final boolean tryReleaseShared(int unused) {//共享模式的释放锁
            Thread current = Thread.currentThread();
            if (firstReader == current) {
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != current.getId())
                    rh = readHolds.get();
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT;//状态减去1左移16位的数，即读状态减一
                if (compareAndSetState(c, nextc))//CAS更新读状态成功
                    return nextc == 0;//如果更新后的状态位为0，代表读写锁完全释放，返回true。返回true才会唤醒同步队列中的线程
            }
        }

        private IllegalMonitorStateException unmatchedUnlockException() {
            return new IllegalMonitorStateException(
                "attempt to unlock read lock, not locked by current thread");
        }

        protected final int tryAcquireShared(int unused) {//共享模式的获取锁
            Thread current = Thread.currentThread();
            int c = getState();
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)//独占锁被持有，且当前线程不是持有独占锁的线程，返回不可获取
                return -1;
            int r = sharedCount(c);//读状态
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {//如果读锁不需要阻塞，且读状态小于最大值，且读状态CAS增加成功，意味着获取共享锁成功
                if (r == 0) {//如果之前读状态是0，设置firstReader第一个读线程为当前线程，firstReaderHoldCount第一个读线程获取锁计数1
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {//firstReader第一个读线程为当前线程，说明这次是重入的获取共享锁，firstReaderHoldCount第一个读线程获取锁计数＋1
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;//缓存最后一个获取共享锁的线程状态
                    if (rh == null || rh.tid != current.getId())
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;//readHolds保存每个线程持有的读锁计数
                }
                return 1;//获取成功
            }
            //不可获取
            return fullTryAcquireShared(current);
        }
        
        final int fullTryAcquireShared(Thread current) {
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0) {//独占锁被持有
                    if (getExclusiveOwnerThread() != current)//当前线程不是持有独占锁的线程。返回-1
                        return -1;
                } else if (readerShouldBlock()) {//如果需要阻塞
                    if (firstReader == current) {
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != current.getId()) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT)//共享锁获取已经达到最大计数
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {//CAS增加读状态成功
                    if (sharedCount(c) == 0) {//如果之前读状态是0，设置firstReader第一个读线程为当前线程，firstReaderHoldCount第一个读线程获取锁计数1
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {//firstReader第一个读线程为当前线程，说明这次是重入的获取共享锁，firstReaderHoldCount第一个读线程获取锁计数＋1
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != current.getId())
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;//获取成功
                }
            }
        }
        
        final boolean tryWriteLock() {//仅当写入锁在调用期间未被另一个线程保持时获取该锁
            Thread current = Thread.currentThread();
            int c = getState();
            if (c != 0) {//状态不为0
                int w = exclusiveCount(c);
                if (w == 0 || current != getExclusiveOwnerThread())//写状态为0或者当前线程不是持有独占锁的线程
                    return false;
                if (w == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
            }
            //写状态不为0，且当前线程是持有独占锁的线程
            if (!compareAndSetState(c, c + 1))
                return false;
            setExclusiveOwnerThread(current);
            return true;//CAS更新成功，设置当前线程为持有独占锁的线程，返回成功获取
        }
        
        final boolean tryReadLock() {//仅当写入锁在调用期间未被另一个线程保持时获取读取锁。
            Thread current = Thread.currentThread();
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0 &&
                    getExclusiveOwnerThread() != current)
                    return false;//这种情况说明独占锁被持有，且不是当前线程持有，一定获取不到共享锁
                int r = sharedCount(c);
                if (r == MAX_COUNT)//最大值校验
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {//循环CAS直到更新状态成功，获取到共享锁
                    if (r == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        HoldCounter rh = cachedHoldCounter;
                        if (rh == null || rh.tid != current.getId())
                            cachedHoldCounter = rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                    }
                    return true;
                }
            }
        }

        protected final boolean isHeldExclusively() {//当前线程是否是持有独占锁的线程
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        // Methods relayed to outer class

        final ConditionObject newCondition() {//独占锁可以获取等待队列
            return new ConditionObject();
        }

        final Thread getOwner() {//获取当前持有独占锁的线程
            return ((exclusiveCount(getState()) == 0) ?
                    null :
                    getExclusiveOwnerThread());
        }

        final int getReadLockCount() {//共享锁计数
            return sharedCount(getState());
        }

        final boolean isWriteLocked() {//独占锁是否被持有
            return exclusiveCount(getState()) != 0;
        }

        final int getWriteHoldCount() {//获取当前线程持有的写锁计数
            return isHeldExclusively() ? exclusiveCount(getState()) : 0;
        }

        final int getReadHoldCount() {//获取当前线程获取共享锁的重入计数
            if (getReadLockCount() == 0)
                return 0;

            Thread current = Thread.currentThread();
            if (firstReader == current)
                return firstReaderHoldCount;

            HoldCounter rh = cachedHoldCounter;
            if (rh != null && rh.tid == current.getId())
                return rh.count;

            int count = readHolds.get().count;
            if (count == 0) readHolds.remove();
            return count;
        }

        /**
         * Reconstitute this lock instance from a stream
         * @param s the stream
         */
        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            s.defaultReadObject();
            readHolds = new ThreadLocalHoldCounter();
            setState(0); // reset to unlocked state
        }

        final int getCount() { return getState(); }//获取总读写状态
    }
```

## NonfairSync

```java
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -8159625535654395037L;
        final boolean writerShouldBlock() {
            return false; 
        }
        final boolean readerShouldBlock() {
            return apparentlyFirstQueuedIsExclusive();
        }
    }

    final boolean apparentlyFirstQueuedIsExclusive() {
        Node h, s;
        return (h = head) != null &&
            (s = h.next)  != null &&
            !s.isShared()         &&
            s.thread != null;
    }
```

## FairSync

```java
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -2274990926593161451L;
        final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }
        final boolean readerShouldBlock() {
            return hasQueuedPredecessors();
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

```java
public class ReadWriteLockTest {

    private final Map<String, String> m = new TreeMap<String, String>();
    private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    private final Lock r = rwl.readLock();
    private final Lock w = rwl.writeLock();

    public String get(String key) {
        r.lock();
        try {
            return m.get(key);
        } finally {
            r.unlock();
        }
    }

    public String put(String key, String value) {
        w.lock();
        try {
            return m.put(key, value);
        } finally {
            w.unlock();
        }
    }

    public void clear() {
        w.lock();
        try {
            m.clear();
        } finally {
            w.unlock();
        }
    }

}
```
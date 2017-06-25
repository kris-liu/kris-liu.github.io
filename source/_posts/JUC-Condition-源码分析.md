---
title: JUC - Condition 源码分析
date: 2016-10-12 15:38:57
categories: Concurrent
tags: [Java,并发,源码]
---

Condition将Object监视器方法（wait、notify 和 notifyAll）分解成截然不同的对象，以便通过将这些对象与任意Lock实现组合使用，为每个对象提供多个等待set（wait-set）。其中，Lock替代了synchronized方法和语句的使用，Condition替代了Object监视器方法的使用。Condition实例实质上被绑定到一个锁上。要为特定Lock实例获得Condition实例，请使用其newCondition()方法。

<!--more-->

# 源码分析

Condition属于**AbstractQueuedSynchronizer**（详见：[JUC - AbstractQueuedSynchronizer(AQS) 源码分析](https://kris-liu.github.io/2016/09/28/JUC-AbstractQueuedSynchronizer-AQS-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)）的一部分，AQS内部实现了一个实现Condition接口的内部类。
&emsp;&emsp;AQS维护了一个申请获取锁的同步队列(pred，next连接的双向队列)，Condition维护了一个等待队列(nextWaiter连接的单向队列)，一个AQS可以伴有多个Condition。当线程A获取到锁，然后调用await方法时，会将当前线程A加入到等待队列中并释放锁并挂起线程A；当线程B获取到锁，然后调用signal方法，将等待队列中的线程A转移到同步队列中，在同步队列中等待被唤醒，唤醒由释放锁的同时唤醒后继节点触发，线程A被唤醒后，将调用AQS的acquireQueued方法，直到获取到锁，线程A才从await方法返回。

### ＊void await() throws InterruptedException

造成当前线程在接到信号或被中断之前一直处于等待状态。

```java
        public final void await() throws InterruptedException {//调用者一定是获取了锁的线程
            if (Thread.interrupted())//中断检测
                throw new InterruptedException();
            Node node = addConditionWaiter();//将当前节点添加到等待队列中
            long savedState = fullyRelease(node);//释放锁，并返回当前锁重入的次数
            int interruptMode = 0;//中断模式，0不中断，-1抛出中断异常，1不抛异常只设置中断标志。
            while (!isOnSyncQueue(node)) {//检查当前线程节点是否在同步队列中，不在则继续挂起线程。signal方法会将等待队列中的节点转移到同步队列中。
                LockSupport.park(this);//挂起线程直到被唤醒，期望的唤醒由当其他线程释放锁然后唤醒同步队列的后继节点时触发。也可能是被中断或者意外被唤醒，所以需要在循环中检查是否仍需挂起。
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)//检查中断状态
                    break;
            }
            if (acquireQueued(node, savedState) //AQS的acquireQueued会一直尝试获取锁，获取失败则挂起线程，直到获取到锁，才返回。返回true代表acquireQueued过程中被中断过。
	            && interruptMode != THROW_IE)//如果if返回true代表获取锁的过程被中断过，且不需要抛异常，则interruptMode=REINTERRUPT会在接下来重新中断。在同步队列获取锁过程中被中断，只重新中断，不抛异常；在等待队列中中断才会抛出异常。
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) //如果节点被取消或者中断，则在等待队列中清除掉该节点
                unlinkCancelledWaiters();
            if (interruptMode != 0)//不为0代表需要重新设置中断或者抛出中断异常
                reportInterruptAfterWait(interruptMode);//根据interruptMode重新设置中断或者抛出中断异常，interruptMode代表中断模式，0不中断，-1抛出中断异常，1不抛异常只设置中断标志。
        }
```

Condition中的方法：


```java
        private Node addConditionWaiter() {//添加当前线程节点进入等待队列。此方法执行时，一定是持有锁的，不会有并发问题
            Node t = lastWaiter;
            if (t != null && t.waitStatus != Node.CONDITION) {//当等待队列中有节点且尾节点状态不是CONDITION，则移除等待队列中不是CONDITION状态的节点。
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            Node node = new Node(Thread.currentThread(), Node.CONDITION);//构造当前节点
            if (t == null)//没有尾部节点，则新节点就是first和last，否则将新节点添加到next。
                firstWaiter = node;
            else 
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
        
        private void unlinkCancelledWaiters() {//移除等待队列中不是CONDITION状态的节点。
            Node t = firstWaiter;
            Node trail = null;
            while (t != null) {//从前向后遍历
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {
                    t.nextWaiter = null;
                    if (trail == null)
                        firstWaiter = next;
                    else
                        trail.nextWaiter = next;
                    if (next == null)
                        lastWaiter = trail;
                }
                else
                    trail = t;
                t = next;
            }
        }

		private void reportInterruptAfterWait(int interruptMode)//根据中断模式interruptMode决定抛出异常还是重新中断还是不处理
            throws InterruptedException {
            if (interruptMode == THROW_IE)
                throw new InterruptedException();
            else if (interruptMode == REINTERRUPT)
                selfInterrupt();
        }

		private int checkInterruptWhileWaiting(Node node) {//检查是否被中断过，清空中断状态并返回相应中断模式
            return Thread.interrupted() ?
                (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                0;
        }
        
```

AQS中的方法：


```java
	final boolean isOnSyncQueue(Node node) {//检查节点是否被转移到同步队列中了
        if (node.waitStatus == Node.CONDITION || node.prev == null)//节点状态是CONDITION或者prev是空一定没有加入同步队列，因为加入同步队列一定会设置prev并且状态不为CONDITION
            return false;
        if (node.next != null) //next不为空也一定在同步队列
            return true;
        return findNodeFromTail(node);////遍历同步队列，查看该节点是否出现在同步队列中
    }

	private boolean findNodeFromTail(Node node) {//遍历同步队列，查看该节点是否出现在同步队列中
        Node t = tail;
        for (;;) {
            if (t == node)
                return true;
            if (t == null)
                return false;
            t = t.prev;
        }
    }

	final long fullyRelease(Node node) {//完全释放锁，并返回锁的重入次数，失败则取消节点
        boolean failed = true;
        try {
            long savedState = getState();
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }
    
	final boolean transferAfterCancelledWait(Node node) {//转移节点
        if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {//CAS操作将node的waitStatus从CONDITION设置为0，如果成功，说明当中断发生时，没有signal发生（signal的第一步是将node的waitStatus设置为0），再调用enq将线程放入同步队列后直接返回true，抛出中断异常
            enq(node);//添加到同步队列中
            return true;
        }//如果CAS返回false，说明CAS的时候 中断和signal都已经发生了
        while (!isOnSyncQueue(node))//直到加入同步队列，才返回
            Thread.yield();
        return false;//返回false代表上层会重新中断，这种不抛出异常
    }

	final boolean acquireQueued(final Node node, int arg) {//AQS的自旋阻塞获取锁方法，该方法只有获取到锁才能被返回，不响应超时和中断，用这个方法是因为await返回必须要有锁，即使await方法设置了超时，也只是设置了在等待队列中的超时时间，在同步队列中不响应超时设置。方法分析见[JUC 源码分析 － AbstractQueuedSynchronizer(AQS)](http://blog.csdn.net/xx_ytm/article/details/52690851)）
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
    
```

### ＊void awaitUninterruptibly()

造成当前线程在接到信号之前一直处于等待状态。

```
		public final void awaitUninterruptibly() {
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            boolean interrupted = false;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if (Thread.interrupted())
                    interrupted = true;
            }
            if (acquireQueued(node, savedState) || interrupted)
                selfInterrupt();//在同步队列或等待队列被中断，都忽略，然后重新中断，不抛异常
        }
```

### ＊long awaitNanos(long nanosTimeout) throws InterruptedException

造成当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态。

```java
		public final long awaitNanos(long nanosTimeout)
                throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            long lastTime = System.nanoTime();
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (nanosTimeout <= 0L) {//如果在等待队列超时,则转移节点到同步队列中，转移失败说明节点同时被signal了
                    transferAfterCancelledWait(node);
                    break;
                }
                LockSupport.parkNanos(this, nanosTimeout);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;

                long now = System.nanoTime();
                nanosTimeout -= now - lastTime;
                lastTime = now;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return nanosTimeout - (System.nanoTime() - lastTime);//这里计算超时会算上节点在同步队列中等待锁的时间。
        }
```

### ＊boolean await(long time, TimeUnit unit) throws InterruptedException

造成当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态。此方法在行为上等效于：awaitNanos(unit.toNanos(time)) > 0

```java
		public final boolean await(long time, TimeUnit unit)
                throws InterruptedException {
            if (unit == null)
                throw new NullPointerException();
            long nanosTimeout = unit.toNanos(time);
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            long lastTime = System.nanoTime();
            boolean timedout = false;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (nanosTimeout <= 0L) {
                    timedout = transferAfterCancelledWait(node);//返回true代表超时，false说明调用之前被signal了，不认为超时，这里超时时间只是控制在等待队列的超时，实际有可能在同步队列等了很久，导致总时间开销大于超时时间
                    break;
                }
                if (nanosTimeout >= spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
                long now = System.nanoTime();
                nanosTimeout -= now - lastTime;
                lastTime = now;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return !timedout;
        }
```

### ＊boolean awaitUntil(Date deadline) throws InterruptedException

造成当前线程在接到信号、被中断或到达指定最后期限之前一直处于等待状态。

```java
		public final boolean awaitUntil(Date deadline)
                throws InterruptedException {
            if (deadline == null)
                throw new NullPointerException();
            long abstime = deadline.getTime();
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            boolean timedout = false;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (System.currentTimeMillis() > abstime) {
                    timedout = transferAfterCancelledWait(node);//返回true代表超时，false说明调用之前被signal了，不认为超时，这里超时时间只是控制在等待队列的超时，实际有可能在同步队列等了很久，导致总时间开销大于超时时间
                    break;
                }
                LockSupport.parkUntil(this, abstime);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return !timedout;
        }
```

### ＊void signal()

唤醒一个等待线程。

```java
		public final void signal() {//将节点从等待队列转移到同步队列，持有锁的线程才能执行此操作
            if (!isHeldExclusively())//当前线程必须是持有锁的线程
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);//signal等待队列的首节点
        }
```

```java
		private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)//移除移除老的first，将firstWaiter置为signal节点的next
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) //
		             && (first = firstWaiter) != null);
        }
        
	final boolean transferForSignal(Node node) {
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))//将需要Signal的节点的状态重置
            return false;
        Node p = enq(node);//将需要signal的节点添加到同步队列中
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))//如果此时节点正好被中断或者取消，则unpark当前节点
            LockSupport.unpark(node.thread);
        return true;
    }
```

### ＊void signalAll()

唤醒所有等待线程。

```java
		public final void signalAll() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignalAll(first);
        }
```

```java
		private void doSignalAll(Node first) {
            lastWaiter = firstWaiter = null;
            do {
                Node next = first.nextWaiter;
                first.nextWaiter = null;
                transferForSignal(first);
                first = next;
            } while (first != null);//从前向后遍历signal等待队列中的所有的节点
        }
```

# 使用方式

BoundedBuffer是使用两个Condition维护的一个阻塞队列，队列空时，take方法会等待直到队列中有新元素加入；队列满时，put方法会等待直到队列中有元素移出。

```java
public class ConditionTest {

    static class BoundedBuffer {
        final Lock lock = new ReentrantLock();
        final Condition notFull  = lock.newCondition();
        final Condition notEmpty = lock.newCondition();

        final Object[] items = new Object[3];
        int putptr, takeptr, count;

        public void put(Object x) throws InterruptedException {
            lock.lock();
            try {
                while (count == items.length) {
                    System.out.println("队列满，put等待");
                    notFull.await();
                }
                items[putptr] = x;
                if (++putptr == items.length) putptr = 0;
                ++count;
                notEmpty.signal();
            } finally {
                lock.unlock();
            }
        }

        public Object take() throws InterruptedException {
            lock.lock();
            try {
                while (count == 0) {
                    System.out.println("队列空，take等待");
                    notEmpty.await();
                }
                Object x = items[takeptr];
                if (++takeptr == items.length) takeptr = 0;
                --count;
                notFull.signal();
                return x;
            } finally {
                lock.unlock();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        final BoundedBuffer buffer = new BoundedBuffer();
        final Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    buffer.put(new Object());
                    buffer.put(new Object());
                    buffer.put(new Object());
                    buffer.put(new Object());
                    Thread.sleep(1000l);
                    buffer.put(new Object());
                    buffer.put(new Object());
                    buffer.put(new Object());
                    buffer.put(new Object());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

            }
        });
        Thread thread2 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(1000l);
                    buffer.take();
                    buffer.take();
                    buffer.take();
                    buffer.take();
                    Thread.sleep(1000l);
                    buffer.take();
                    buffer.take();
                    buffer.take();
                    buffer.take();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        thread.start();
        thread2.start();

        thread.join();
        thread2.join();
        System.out.println("buffer 是否为空 ： " + (buffer.count == 0));
    }

}

```
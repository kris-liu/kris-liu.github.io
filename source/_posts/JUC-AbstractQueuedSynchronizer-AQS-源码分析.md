---
title: JUC - AbstractQueuedSynchronizer(AQS) 源码分析
date: 2016-09-28 15:24:00
categories: Concurrent
tags: [Java,并发,锁,源码]
---
AbstractQueuedSynchronizer，同步器，以下简称AQS。本文从源码分析AQS的核心方法和实现原理。

AQS内部有两组重要的成员变量：

1. int类型的status变量，通过CAS操作（详见：[CAS深度分析](http://blog.csdn.net/hsuxu/article/details/9467651)）改变status值来控制当前线程能否访问资源以及并发数量。

2. Node类型的head和tail两个变量，两个变量维护了一个FIFO的同步队列，将获取访问权限失败的线程构造成Node节点加入队列中，释放资源时再来唤醒队列中阻塞的线程。（Node类型主要包涵节点的状态，当前线程的引用，以及前驱节点和后置节点的引用）

<!--more-->

# 使用方式

AQS在使用时一般是作为自定义同步工具的内部类，实现AQS中可重写的方法，来自定义获取以及释放锁的方式，在自定义同步工具类中，调用AQS中提供给使用者的模版方法，来控制锁的获取和释放。

## 提供给使用者调用的模版方法：

独占式

```
void acquire(int arg)
以独占模式获取对象，忽略中断。

void acquireInterruptibly(int arg) throws InterruptedException
以独占模式获取对象，如果被中断则中止，抛出InterruptedException。

boolean tryAcquireNanos(int arg,long nanosTimeout) throws InterruptedException
试图以独占模式获取对象，如果被中断则中止，抛出InterruptedException，如果到了给定超时时间，则会返回失败。

boolean release(int arg)
以独占模式释放对象。
```

共享式

```
void acquireShared(int arg)
以共享模式获取对象，忽略中断。

void acquireSharedInterruptibly(int arg) throws InterruptedException
以共享模式获取对象，如果被中断则中止，抛出InterruptedException。

boolean tryAcquireSharedNanos(int arg,long nanosTimeout) throws InterruptedException
试图以共享模式获取对象，如果被中断则中止，抛出InterruptedException，如果到了给定超时时间，则会返回失败。

boolean releaseShared(int arg)
以共享模式释放对象。
```

## 同步器可重写的方法：

独占式

```
protected boolean tryAcquire(int arg)	
独占的获取这个状态。这个方法的实现需要查询当前状态是否允许获取，然后再进行获取（使用compareAndSetState来做）状态。

protected boolean tryRelease(int arg) 	
释放状态。
```

共享式

```
protected int tryAcquireShared(int arg)	
共享的模式下获取状态。

protected boolean tryReleaseShared(int arg)	
共享的模式下释放状态。
```

# 核心方法及源码分析

## 独占模式的获取和释放

### * void acquire(int arg)

以独占模式获取对象，忽略中断。

```
	public final void acquire(int arg) {
        if (!tryAcquire(arg) //通过CAS更新status尝试获取
	        && acquireQueued(addWaiter(Node.EXCLUSIVE) //获取锁失败后添加到同步队列
	        , arg)) //在队列中自旋直至成功获取锁才返回，线程可能被反复park、unpark，直到获取锁，返回值代表是否被中断过
            selfInterrupt(); //之前被中断过则还原中断状态
    }
```

```
	private Node addWaiter(Node mode) { //添加到同步队列
        Node node = new Node(Thread.currentThread(), mode); //把当前线程构造成Node节点
        Node pred = tail; //获取当前的tail节点
        if (pred != null) {
            node.prev = pred; //当前节点前驱节点设置为当前的tail节点
            if (compareAndSetTail(pred, node)) { //CAS操作，如果内存中当前tail节点的值是pred，则将tail节点指向当前新节点node，返回true代表新节点成功进入队列
                pred.next = node; //将前驱节点的next设置为node节点，这个节点无法和tail节点一起通过CAS操作设置，next节点仅仅是为了优化，当next为空时，始终可以通过tail节点的pred字段从后向前遍历所有节点。
                return node;
            }
        }
        enq(node); //失败时通过无线循环直至成功添加节点
        return node;
    }
    
	private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { //当前没有初始化过队列则创建初始化新节点，再通过循环添加当前线程节点
                if (compareAndSetHead(new Node())) //原子地设置头结点
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

```
	final boolean acquireQueued(final Node node, int arg) { //使节点阻塞自旋，直至获取到锁，才返回。
        boolean failed = true; //当前获取是否失败
        try {
            boolean interrupted = false; //当前获取是否被中断
            for (;;) {
                final Node p = node.predecessor(); //获取有效的前驱节点，前驱节点一定不为null
                if (p == head //head节点要么是初始化的节点，要么代表当前成功获取到锁的线程，所以前驱节点是head的时候，当前节点才应该尝试去获取锁 
                    && tryAcquire(arg) ) { //尝试获取锁
                    setHead(node); //设置当前节点为head
                    p.next = null; //之前老的head节点next引用置空 help GC 
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) //查看当前节点是否应该被park 
                    && parkAndCheckInterrupt()) //park当前线程，直到被其他线程unpark，如果被中断唤醒，则返回true，由于acquire忽略中断所以重新尝试获取锁，获取失败失败重新park。
                    interrupted = true; //线程被中断过
            }
        } finally {
            if (failed)
                cancelAcquire(node); //取消当前线程
        }
    }
    
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) { //查看当前节点是否应该被park 
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL) //前驱节点是signal状态, park当前线程
            return true;
        if (ws > 0) { //前驱结点已取消
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0); //前驱结点已取消，向前遍历直到找到一个非取消结点，同时将取消节点的前后节点相连
            pred.next = node;
        } else {
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL); //将当前节点设置为signal状态，表明后继节点需要unpark
        }
        return false;
    }
    
    private final boolean parkAndCheckInterrupt() { //park当前线程，并返回当前中断状态
        LockSupport.park(this); //挂起当前线程，当中断或者被其他线程unpark这个线程则返回，不区分park和unpark的先后
        return Thread.interrupted(); //清空并返回中断状态，保证后续仍然可以park，返回的值将决定完成获取后是否需要恢复中断状态
    }
```

```
    private void cancelAcquire(Node node) { //取消当前节点
        if (node == null)
            return;
        node.thread = null;
        Node pred = node.prev;
        while (pred.waitStatus > 0) //跳过已经取消的前驱节点
            node.prev = pred = pred.prev;
            
        Node predNext = pred.next; //predNext是需要移除的结点
        node.waitStatus = Node.CANCELLED; //无条件设置节点状态为取消
        if (node == tail && compareAndSetTail(node, pred)) { //如果处于链尾，直接移除，再修复前驱的连接关系
            compareAndSetNext(pred, predNext, null);
        } else {node有后继。用前驱的next指针指向他，这样他会得到正确的signal信号，否则唤醒他来传播信号。
            int ws;
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) { //如果true，说明node有一个需要signal的前驱，让这个前驱指向node的后继，完整节点的链接关系
                Node next = node.next;
                if (next != null && next.waitStatus <= 0) //node有后继
                    compareAndSetNext(pred, predNext, next);
            } else {
                unparkSuccessor(node); //在执行到上面一句将waitStatus置CANCELLED之前，锁被释放，该node线程被唤醒，则释放锁线程的unparkSuccessor不能起到预期作用，所以这里需要调用unparkSuccessor.即使此时持有锁的线程没有释放锁也不会有严重后果，被unpark的线程在获取锁失败后会继续park
            }
            node.next = node; // help GC
        }
    }
```

### * void acquireInterruptibly(int arg) throws InterruptedException

以独占模式获取对象，如果被中断则中止，抛出InterruptedException。

```
	public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted()) //被中断则清空中断状态并抛出异常
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }

	private void doAcquireInterruptibly(int arg)
        throws InterruptedException { //和acquireQueued的区别就是被中断则不再重新获取，直接结束
        final Node node = addWaiter(Node.EXCLUSIVE); //将节点添加进同步队列
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt()) //park当前线程，直到被其他线程unpark，如果被中断唤醒，则返回true
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

### * boolean tryAcquireNanos(int arg,long nanosTimeout) throws InterruptedException

试图以独占模式获取对象，如果被中断则中止，抛出InterruptedException，如果到了给定超时时间，则会返回失败。

```
	public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted()) //被中断则清空中断状态并抛出异常
            throw new InterruptedException();
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
    }

	private boolean doAcquireNanos(int arg, long nanosTimeout) //doAcquireInterruptibly
        throws InterruptedException {
        long lastTime = System.nanoTime(); //获取当前时间
        final Node node = addWaiter(Node.EXCLUSIVE); //将节点添加进同步队列
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
                if (nanosTimeout <= 0) //如果没有获取到，且超时时间小于等于0则返回失败
                    return false;
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold) //spinForTimeoutThreshold为1000纳秒，当休眠时间小于等于1000纳秒时，由于非常短的超时等待无法做到十分精确，也会带来额外的线程上下文切换，所以这里直接进入快速的自旋。否则park线程
                    LockSupport.parkNanos(this, nanosTimeout);
                long now = System.nanoTime(); //获取unpark后的时间
                nanosTimeout -= now - lastTime; //将超时时间减去获取锁以及线程park的时间，得到之后还需要park的时间。
                lastTime = now;
                if (Thread.interrupted()) //被中断则清空中断状态并抛出异常
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

### * boolean release(int arg)

以独占模式释放对象。

```
	public final boolean release(int arg) {
        if (tryRelease(arg)) { //如果修改status状态释放锁成功
            Node h = head; //head是初始化的节点或代表当前占有锁的线程，所以要unparkhead的有效后继节点
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h); //unparkhead节点的有效后继节点
            return true;
        }
        return false;
    }
    
	private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        if (ws < 0) //重置头节点的状态为0
            compareAndSetWaitStatus(node, ws, 0);
            
        Node s = node.next;
        if (s == null || s.waitStatus > 0) { //后继节点是null或者已取消
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev) //节点的next字段用于优化，当next链接没有时，仍然需要从tail节点向前遍历检查，获取队列最前面的有效节点作为需要unpark的节点
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);//unpark同步队列最前面的有效节点
    }
```

## 共享模式的获取和释放

### * void acquireShared(int arg)

以共享模式获取对象，忽略中断。

```
	public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0) //tryAcquireShared大于等于0则共享锁获取成功
            doAcquireShared(arg);
    }
    
    private void doAcquireShared(int arg) { //和独占式的acquireQueued方法区别就是获取成功的节点会继续unpark后继节点，将共享状态向后传播
        final Node node = addWaiter(Node.SHARED); //添加到同步队列
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) { //节点的前驱是head
                    int r = tryAcquireShared(arg);
                    if (r >= 0) { //获取锁成功
                        setHeadAndPropagate(node, r); //设置头节点并unpark后继节点，传播共享状态
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
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

	private void setHeadAndPropagate(Node node, int propagate) { //设置头节点并unpark后继节点，传播共享状态。参数propagate代表当前还有可以获取的锁数量
        Node h = head; 
        setHead(node); //设置当前节点为head节点
        if (propagate > 0 || h == null || h.waitStatus < 0) {//当前有资源可以获取，或当前节点可以唤醒后继节点 
            Node s = node.next;
            if (s == null || s.isShared()) //后继节点是共享的则unpark后继节点
                doReleaseShared(); //unpark后继节点
        }
    }
```

### * void acquireSharedInterruptibly(int arg) throws InterruptedException

以共享模式获取对象，如果被中断则中止，抛出InterruptedException。

```
	public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }

	private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

### * boolean tryAcquireSharedNanos(int arg,long nanosTimeout) throws InterruptedException

试图以共享模式获取对象，如果被中断则中止，抛出InterruptedException，如果到了给定超时时间，则会返回失败。

```
	public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquireShared(arg) >= 0 ||
            doAcquireSharedNanos(arg, nanosTimeout);
    }
    
    private boolean doAcquireSharedNanos(int arg, long nanosTimeout)
        throws InterruptedException {

        long lastTime = System.nanoTime();
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return true;
                    }
                }
                if (nanosTimeout <= 0)
                    return false;
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                long now = System.nanoTime();
                nanosTimeout -= now - lastTime;
                lastTime = now;
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

### * boolean releaseShared(int arg)

以共享模式释放对象。

```
	public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared(); //unpark后继节点，传播共享状态
            return true;
        }
        return false;
    }
    
    private void doReleaseShared() { //unpark后继节点，传播共享状态
        for (;;) {
            Node h = head;
            if (h != null && h != tail) { //如果队列中有节点
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) { //头节点状态是signal，则重置状态为0，并unpark后继节点
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE)) ／／如果状态为0，则将0设置为PROPAGATE状态
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```



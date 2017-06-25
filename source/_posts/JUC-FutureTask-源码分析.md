---
title: JUC - FutureTask 源码分析
date: 2016-11-29 23:43:59
categories: Concurrent
tags: [Java,并发,异步,源码]
---

FutureTask，可取消的异步计算任务。利用开始和取消计算的方法、查询计算是否完成的方法和获取计算结果的方法，此类提供了对Future的基本实现。仅在计算完成时才能获取结果；如果计算尚未完成，则阻塞get方法。一旦计算完成，就不能再重新开始或取消计算。可使用 FutureTask包装Callable或Runnable对象。因为FutureTask实现了Runnable，所以可将FutureTask提交给Executor执行。

<!--more-->

# 源码分析

## FutureTask类继承关系

FutureTask类实现了RunnableFuture接口，RunnableFuture接口继承自Future接口和Runnable接口，整合了一下Future接口和Runnable接口。
其中的方法在FutureTask中做了具体的实现。

 - Future接口表示异步计算的结果。它提供了检查计算是否完成的方法，以等待计算的完成，并获取计算的结果。计算完成后只能使用get方法来获取结果，如有必要，计算完成前可以阻塞此方法。取消则由cancel方法来执行。还提供了其他方法，以确定任务是正常完成还是被取消了。一旦计算完成，就不能再取消计算。

 - Runnable接口是为了方便把FutureTask提交给线程池，线程池中的工作线程将调用他的 run 方法。

```java
public interface Future<V> {

	//试图取消对此任务的执行
    boolean cancel(boolean mayInterruptIfRunning);

	//如果在任务正常完成前将其取消，则返回 true。
    boolean isCancelled();

	//如果任务已完成，则返回 true。
    boolean isDone();

	//如有必要，等待计算完成，然后获取其结果。
    V get() throws InterruptedException, ExecutionException;

	//如有必要，最多等待为使计算完成所给定的时间之后，获取其结果（如果结果可用）。
    V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException;
        
}
```

```java
public interface Runnable {

    public abstract void run();
    
}
```

## FutureTask核心属性

```java
    /**
     * 状态扭转
     * NEW -> COMPLETING -> NORMAL //正常完成 
     * NEW -> COMPLETING -> EXCEPTIONAL //异常 
     * NEW -> CANCELLED //取消 
     * NEW -> INTERRUPTING -> INTERRUPTED //中断
     */
    private volatile int state;//Future状态
    private static final int NEW          = 0;//初始化
    private static final int COMPLETING   = 1;//运行中
    private static final int NORMAL       = 2;//正常完成
    private static final int EXCEPTIONAL  = 3;//异常
    private static final int CANCELLED    = 4;//已取消
    private static final int INTERRUPTING = 5;//中断中
    private static final int INTERRUPTED  = 6;//中断完成

    //内部的callable，运行完成后设置为null
    private Callable<V> callable;
    
    //从get方法返回或者抛出异常
    private Object outcome; //没有volatile修饰, 通过状态字段state读写保证了可见性
    
    //执行内部callable的线程
    private volatile Thread runner;
    
    /** 
	 * Treiber stack 等待线程的非阻塞堆栈，存放所有等待的线程
     * 首先获取当前最顶的节点，创建一个新节点放在堆栈上，
     * 如果最顶端的节点在获取之后没有变化，那么就设置上新节点。
     * 如果 CAS 失败，意味着另一个线程已经修改了堆栈，
     * 那么会重新执行上述操作。
     */
    private volatile WaitNode waiters;

    static final class WaitNode {//包含了当前线程对象，并有指向下一个WaitNode的指针next，Treiber Stack就是由WaitNode组成的一个单向链表。
        volatile Thread thread;
        volatile WaitNode next;
        WaitNode() { thread = Thread.currentThread(); }
    }
```

## FutureTask核心方法源码分析

### 构造方法

```java
	public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;//volatile写保证callable的可见性
    }
    
    public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);//包装成一个Callable
        this.state = NEW;//volatile写保证callable的
    }
```

```java
    public static <T> Callable<T> callable(Runnable task, T result) {
        if (task == null)
            throw new NullPointerException();
        return new RunnableAdapter<T>(task, result);
    }
    
	static final class RunnableAdapter<T> implements Callable<T> {//把Runnable包装成一个Callable，在线程池中会执行FutureTask的run方法，在FutureTask的run方法中会执行Callable的call方法，在这个RunnableAdapter中会执行task的run方法。
        final Runnable task;
        final T result;
        RunnableAdapter(Runnable task, T result) {
            this.task = task;
            this.result = result;
        }
        public T call() {
            task.run();
            return result;
        }
    }
```

### 核心运行方法run()

```java
    public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset, null, Thread.currentThread()))//如果当前状态不是NEW，或者状态是NEW但是将执行线程runner用CAS从null更新为当前线程失败，则直接退出
            return;
        try {
            Callable<V> c = callable;//获取当前需要执行的callable
            if (c != null && state == NEW) {//callable不为null且状态是NEW，则执行业务逻辑
                V result;//call方法的返回值
                boolean ran;//是否正常执行完成
                try {
                    result = c.call();//调用call方法
                    ran = true;//是正常完成设置true
                } catch (Throwable ex) {//如果抛出异常
                    result = null;//设置结果为null
                    ran = false;//设置非正常完成false
                    setException(ex);//设置异常
                }
                if (ran)//正常完成
                    set(result);//设置返回结果
            }
        } finally {
            runner = null;//将执行线程设置为null
            int s = state;//重新读取状态
            if (s >= INTERRUPTING)//如果是中断的，处理中断
                handlePossibleCancellationInterrupt(s);
        }
    }

    protected void set(V v) {//设置结果
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {//首先将状态从NEW更改为COMPLETING
            outcome = v;//将结果负值给outcome
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); //设置结果为NORMAL，正常完成
            finishCompletion();//唤醒Treiber stack所有等待线程
        }
    }

    protected void setException(Throwable t) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {////首先将状态从NEW更改为COMPLETING
            outcome = t;//设置结果为异常t
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); //设置结果为EXCEPTIONAL，运行异常
            finishCompletion();//唤醒Treiber stack所有等待线程
        }
    }

    private void finishCompletion() {//唤醒所有等待线程
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {//如果有等待线程，设置为q
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {//将waiters用CAS从q设置为null，置空所有等待线程
                for (;;) {//从q开始遍历WaitNode栈，唤醒所有的等待线程
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();//执行扩展点done方法

        callable = null; //将需要执行的callable设置为null
    }

    private void handlePossibleCancellationInterrupt(int s) {
        if (s == INTERRUPTING)//如果状态是INTERRUPTING，则让出cpu等待状态变成INTERRUPTED才结束
            while (state == INTERRUPTING)
                Thread.yield(); 
    }
```

### V get() throws InterruptedException, ExecutionException

等待计算完成，然后获取其结果。

```java
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)//如果是初始化或者运行中状态，则调用awaitDone等待执行完成后唤醒返回。
            s = awaitDone(false, 0L);//返回结果是当前Future状态
        return report(s);//返回结果
    }

    private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {//timed标示是否超时等待，nano代表超时等待时间
        final long deadline = timed ? System.nanoTime() + nanos : 0L;//获取最长的等待时间，支持超时是当前时间加上等待时间得到未来需要等到的最长时间点，不支持超时是0。
        WaitNode q = null;//当前线程节点
        boolean queued = false;//当前q节点是否加入等待线程栈中
        for (;;) {
            if (Thread.interrupted()) {//如果当前线程被中断，移除当前等待节点q，然后抛出中断异常
                removeWaiter(q);//从等待线程中移除当前节点q
                throw new InterruptedException();
            }

            int s = state;
            if (s > COMPLETING) {//如果已经完成则置空当前节点，并返回完成状态
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING) //如果是COMPLETING，则意味着业务逻辑已经执行结束，让出cpu等待最终状态更新完成后返回
                Thread.yield();
            else if (q == null)//q==null时，初始化当前线程节点q
                q = new WaitNode();
            else if (!queued)//如果当前节点q不再等待线程栈中，则CAS将当前节点加入等待线程栈中，放在等待线程栈的顶部
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset, q.next = waiters, q);
            else if (timed) {//如果支持超时等待，则挂起线程直到超时
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {//如果已经超时，则从等待线程栈中移除当前节点q，并返回当前状态。
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
            else//如果不支持超时等待，则直接挂起线程
                LockSupport.park(this);
        }
    }

    private void removeWaiter(WaitNode node) {//移除当前等待节点node
        if (node != null) {
            node.thread = null;//下面会将node从等待队列中移除，以thread字段为null为依据，出现竞争则重试
            retry:
            for (;;) {          // restart on removeWaiter race
                for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                    s = q.next;
                    if (q.thread != null)
                        pred = q;
                    else if (pred != null) {
                        pred.next = s;
                        if (pred.thread == null) // check for race
                            continue retry;
                    }
                    else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                          q, s))
                        continue retry;
                }
                break;
            }
        }
    }

    private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)//状态是正常完成，则直接返回outcome
            return (V)x;
        if (s >= CANCELLED)//如果是取消后的状态，则直接返回CancellationException，中断也是属于取消的状态
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);//其他状态都返回ExecutionException
    }
```

### V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException

最多等待为使计算完成所给定的时间之后，获取其结果（如果结果可用）。

```java
    public V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (unit == null)
            throw new NullPointerException();
        int s = state;
        if (s <= COMPLETING &&
            (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)//超时等待，如果从awaitDone返回时，状态还是NEW或者COMPLETING，则意味着超时，抛出超时异常
            throw new TimeoutException();
        return report(s);
    }
```

### boolean cancel(boolean mayInterruptIfRunning)

试图取消对此任务的执行。如果任务已完成、或已取消，或者由于某些其他原因而无法取消，则此尝试将失败。当调用 cancel 时，如果调用成功，而此任务尚未启动，则此任务将永不运行。如果任务已经启动，则 mayInterruptIfRunning 参数确定是否应该以试图停止任务的方式来中断执行此任务的线程。如果此时业务方法在执行中且FutureTask状态还是NEW时，可以取消FutureTask，但是无法停止业务方法的执行，取消之后，即使业务方法执行完毕也无法获取执行结果，因为FutureTask状态是取消的。

```java
    public boolean cancel(boolean mayInterruptIfRunning) {
        if (state != NEW)//如果是NEW状态，则一定没取消返回false
            return false;
        if (mayInterruptIfRunning) {//如果强制取消则中断的方式取消任务
            if (!UNSAFE.compareAndSwapInt(this, stateOffset, NEW, INTERRUPTING))//用CAS将state从NEW更新到INTERRUPTING，失败则返回false取消失败，成功则中断运行任务的线程，然后将状态设置为INTERRUPTED，然后唤醒所有等待线程返回true取消成功
                return false;
            Thread t = runner;
            if (t != null)
                t.interrupt();//中断运行线程（如果在线程池中执行任务，该中断会中断线程池中的工作线程，线程池中工作线程的run方法中会清除线程的中断状态）
            UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED); // final state
        }
        else if (!UNSAFE.compareAndSwapInt(this, stateOffset, NEW, CANCELLED))//如果不是强制取消，用CAS将state从NEW更新到CANCELLED，失败则返回false取消失败，成功则唤醒所有等待线程返回true取消成功
            return false;
        finishCompletion();//唤醒所有等待线程
        return true;
    }
```

### boolean isCancelled()

如果在任务正常完成前将其取消，则返回 true。

```java
    public boolean isCancelled() {//大于等于CANCELLED都是取消，中断也是取消
        return state >= CANCELLED;
    }
```

### boolean isDone()

如果任务已完成，则返回 true。 可能由于正常终止、异常或取消而完成，在所有这些情况中，此方法都将返回 true。

```java
    public boolean isDone() {//不是NEW就代表已经执行完成任务，等待返回了
        return state != NEW;
    }
```

## FutureTask扩展点

任务执行完成后执行，可自定义处理逻辑，做监控或记录等等。

```java
    protected void done() { }
    
```
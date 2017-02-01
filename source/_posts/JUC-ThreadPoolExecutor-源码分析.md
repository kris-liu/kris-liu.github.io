---
title: JUC - ThreadPoolExecutor 源码分析
date: 2016-11-27 22:13:52
categories: Concurrent
tags: [Java,并发,线程池,源码]
---
ThreadPoolExecutor，Java线程池。使用线程池可以降低资源消耗，通过重复利用已创建的线程降低线程创建和销毁造成的消耗；可以提高响应速度，当任务到达时，任务可以不需要的等到线程创建就能立即执行；可以提高线程的可管理性，防止无限制的创建线程，消耗系统资源。

下面首先通过初始化参数介绍一下线程池：

<!--more-->

## 初始化参数

```
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

- corePoolSize：线程池的基本大小，核心线程数，当提交一个任务到线程池时，如果线程数小于核心线程数时，即使现有的线程空闲，线程池也会优先创建新线程来处理任务，而不是直接交给现有的线程处理，等到需要执行的任务数大于线程池基本大小时就不再创建。如果调用了线程池的prestartAllCoreThreads方法，线程池会提前创建并启动所有基本线程。核心线程会一直存活，即使没有任务需要处理，除非调用了allowCoreThreadTimeOut方法则允许核心线程超时终止。

- maximumPoolSize：线程池最大大小，当线程数大于或等于核心线程，当任务队列已满，且已创建的线程数小于最大线程数，线程池会创建新的工作线程，直到线程数量达到maxPoolSize。如果线程数已等于maxPoolSize，且任务队列已满，则已超出线程池的处理能力，线程池会拒绝处理任务而抛出异常。

- workQueue：任务队列，用于保存等待执行的任务的阻塞队列，当达到corePoolSize的时候，就向该等待队列放入线程信息 。可以选择以下几个阻塞队列。
 - ArrayBlockingQueue：是一个基于数组结构的有界阻塞队列，此队列按 FIFO（先进先出）原则对元素进行排序。
 - LinkedBlockingQueue：一个基于链表结构的阻塞队列，此队列按FIFO （先进先出） 排序元素，吞吐量通常要高于ArrayBlockingQueue。静态工厂方法Executors.newFixedThreadPool()使用了这个队列。
 - SynchronousQueue：一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedBlockingQueue，静态工厂方法Executors.newCachedThreadPool使用了这个队列。
 - PriorityBlockingQueue：一个具有优先级得无限阻塞队列。

- keepAliveTime：线程活动保持时间，线程池的工作线程空闲后，保持存活的时间。所以如果任务很多，并且每个任务执行的时间比较短，可以调大这个时间，提高线程的利用率。

- unit：线程活动保持时间keepAliveTime的单位，可选的单位有天（DAYS），小时（HOURS），分钟（MINUTES），毫秒(MILLISECONDS)，微秒(MICROSECONDS, 千分之一毫秒)和毫微秒(NANOSECONDS, 千分之一微秒)。

- threadFactory：用于设置创建线程的工厂，可以通过线程工厂给每个创建出来的线程设置更有意义的名字，Debug和定位问题时非常又帮助。

- handler：饱和策略，当队列和线程池都满了，说明线程池处于饱和状态，那么必须采取一种策略处理提交的新任务。以下是JDK提供的策略。
 - AbortPolicy：表示无法处理新任务时抛出异常
 - CallerRunsPolicy：使用调用者所在线程来运行任务。
 - DiscardOldestPolicy：丢弃队列里当前第一个任务，并重新提交当前任务。
 - DiscardPolicy：不处理。
当然也可以根据应用场景需要来实现RejectedExecutionHandler接口自定义策略。如监控，记录日志或持久化不能处理的任务。

## 线程池执行任务流程

线程池按以下行为执行任务：

1. 当线程数小于核心线程数时，创建线程并执行任务。
2. 当线程数大于等于核心线程数，且任务队列未满时，将任务放入任务队列。
3. 当线程数大于等于核心线程数，且任务队列已满。若线程数小于最大线程数，创建线程并执行任务；若线程数到达最大线程数，则抛出异常，拒绝任务。

# 源码分析

## ThreadPoolExecutor类继承关系

ThreadPoolExecutor类继承自AbstractExecutorService抽象类，AbstractExecutorService抽象类实现了ExecutorService接口，ExecutorService接口继承自Executor接口。核心执行方法是Executor接口的execute()方法，ExecutorService扩展了submit()，invokeAll()，invokeAny()方法，并在AbstractExecutorService中做了具体实现，这三个方法最终都会调用Executor接口的execute()方法，execute()方法在ThreadPoolExecutor中做了具体实现。

```
public interface Executor {

    void execute(Runnable command);//在将来某个时间执行给定任务。
    
}
```

```
public interface ExecutorService extends Executor {

    void shutdown();//启动一次顺序关闭，执行以前提交的任务，但不接受新任务。

    List<Runnable> shutdownNow();//试图停止所有正在执行的活动任务，暂停处理正在等待的任务，并返回等待执行的任务列表。

    boolean isShutdown();//如果此执行程序已关闭，则返回 true。

    boolean isTerminated();//如果关闭后所有任务都已完成，则返回 true。

    boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;//请求关闭、发生超时或者当前线程中断，无论哪一个首先发生之后，都将导致阻塞，直到所有任务完成执行。

    <T> Future<T> submit(Callable<T> task);//提交一个返回值的任务用于执行，返回一个表示任务的未决结果的 Future。

    <T> Future<T> submit(Runnable task, T result);//提交一个 Runnable 任务用于执行，并返回一个表示该任务的 Future。

    Future<?> submit(Runnable task);//提交一个 Runnable 任务用于执行，并返回一个表示该任务的 Future。

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException;//执行给定的任务，当所有任务完成时，返回保持任务状态和结果的 Future 列表。


    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException;//执行给定的任务，当所有任务完成或超时期满时（无论哪个首先发生），返回保持任务状态和结果的 Future 列表。

    <T> T invokeAny(Collection<? extends Callable<T>> tasks) throws InterruptedException, ExecutionException;//执行给定的任务，如果某个任务已成功完成（也就是未抛出异常），则返回其结果。

    <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException;//执行给定的任务，如果在给定的超时期满前某个任务已成功完成（也就是未抛出异常），则返回其结果。
    
}

```

## ThreadPoolExecutor核心属性

```
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));//原子的int类型计数器，高三位代表线程池状态，低29位代表线程数量
    private static final int COUNT_BITS = Integer.SIZE - 3;//代表线程数量，29位
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;//允许的最大线程数

	/**
     * 线程池状态的转换
     * RUNNING -> SHUTDOWN：调用shutdown()，隐含在finalize()方法也会调用shutdown()
     * (RUNNING or SHUTDOWN) -> STOP：调用shutdownnow()
     * SHUTDOWN -> TIDYING：当队列和池都是空的
     * STOP -> TIDYING：当线程池是空的
     * TIDYING -> TERMINATED：当terminated()方法完成了
	 **/
	//线程池状态
    private static final int RUNNING    = -1 << COUNT_BITS;//ctl高3位值是RUNNING的代表线程池状态是运行：接受新的任务和进程队列任务
    private static final int SHUTDOWN   =  0 << COUNT_BITS;//ctl高3位值是SHUTDOWN的代表线程池状态是关闭：不接受新的任务，但处理队列中的任务
    private static final int STOP       =  1 << COUNT_BITS;//ctl高3位值是STOP的代表线程池状态是停止：不接受新的任务，不处理队列中的任务，并中断正在进行的任务
    private static final int TIDYING    =  2 << COUNT_BITS;//ctl高3位值是TIDYING的代表线程池状态是整理中：所有任务已经终止，工作线程数是零，过渡到状态为整理中将运行terminated()方法
    private static final int TERMINATED =  3 << COUNT_BITS;//ctl高3位值是TERMINATED的代表线程池状态是终止：terminated()已完成

    private final BlockingQueue<Runnable> workQueue;//任务队列

    private final ReentrantLock mainLock = new ReentrantLock();//线程池主锁，操作线程池工作线程集合，线程池状态等都需要先获得该锁

    private final HashSet<Worker> workers = new HashSet<Worker>();//工作线程Worker集合，用于处理提交的任务或任务队列中的任务

    private int largestPoolSize;//记录线程池有过的最大线程数，操作时需要获取主锁，通过锁保证了可见性和原子性

    private long completedTaskCount;//记录线程池完成的任务数，工作线程退出时累加上去，操作时需要获取主锁，通过锁保证了可见性和原子性

    private volatile ThreadFactory threadFactory;//线程创建工厂，用来创建线程

    private volatile RejectedExecutionHandler handler;//饱和策略

    private volatile long keepAliveTime;//线程空闲时的存活时间，单位纳秒

    private volatile boolean allowCoreThreadTimeOut;//核心线程是否允许超时终止

    private volatile int corePoolSize;//核心线程数量上限

    private volatile int maximumPoolSize;//最大线程数量上限
```

## ThreadPoolExecutor的工作线程Worker

Worker与其说是工作线程，其实是管理工作线程，每个Worker内部的线程在run方法中通过不断的从任务队列中获取任务来不断的处理任务，自身继承自AQS，所以每个Worker自身也是一个锁，保护获取到的任务的执行，在锁状态意味着该Worker正在执行任务。除了创建Worker时提交的任务，其他提交过来的任务都是放入任务队列交给Workers去消费的。

```
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
    
        final Thread thread;//Work管理的线程，final保证构造函数内的操作在构造函数结束后都执行完成，而且对所有线程可见

        Runnable firstTask;//第一个任务

        volatile long completedTasks;//完成的任务计数

        Worker(Runnable firstTask) {
            setState(-1); //初始化并防止中断该Worker
            this.firstTask = firstTask;//创建时提交过来的第一个任务，启动线程后将执行
            this.thread = getThreadFactory().newThread(this);//通过线程工厂创建线程
        }

        public void run() {//运行方法
            runWorker(this);
        }

        protected boolean isHeldExclusively() {//是否被锁，1代表锁已经被获取，0代表锁已经被释放
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {//状态由0到1代表成功获取锁
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {//状态设置为0代表释放锁
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }//获取
        public boolean tryLock()  { return tryAcquire(1); }//尝试获取锁
        public void unlock()      { release(1); }//释放
        public boolean isLocked() { return isHeldExclusively(); }//是否被锁

        void interruptIfStarted() {//如果已经启动则中断该线程
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }

	final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();//获取工作线程的Thread
        Runnable task = w.firstTask;//将task设置为Worker的第一个任务
        w.firstTask = null;//将Worker中的第一个任务置为null
        w.unlock(); //无条件释放锁，其实是将锁的状态从初始化的-1设置为0，代表锁已经准备好，可以被获取
        boolean completedAbruptly = true;//代表runWorker停止的原因，当completedAbruptly为true时代表worker停止是因为worker执行的外部业务逻辑代码抛出异常引起的。当completedAbruptly为false时代表worker停止是线程池内部工作机制下的正常退出。
        try {
            while (task != null || (task = getTask()) != null) {//Worker的第一个任务不为null，或者从任务队列中获取到任务，则执行，此处while循环调用getTask方法从任务队列中获取任务，直到超时获取不到，或者线程池终止
                w.lock();//Worker上锁，保证不被shutDown方法中断

                if ((runStateAtLeast(ctl.get(), STOP) ||//线程池被终止或者
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())//如果线程池已停止且该工作线程还没有被中断则中断该线程；否则清除中断并重新检查是否停止，如果线程被中断过且重新检查时线程池变成了停止状态则重新中断该线程。确保线程池停止时，该工作线程一定被中断了，否则线程一定不能被中断，重新检查确保并发的停止线程池导致的中断被清除
                    wt.interrupt();//中断该工作线程
                try {
                    beforeExecute(wt, task);//线程池的扩展点，用于执行每个任务之前执行。
                    Throwable thrown = null;
                    try {
                        task.run();//调用任务的run方法，这里虽然是个Runnable但是并不需要创建线程来启动，而是直接用线程池中的工作线程调用Runnable的run方法来执行。
                    } catch (RuntimeException x) {//若抛出异常则将异常向上抛出，终止该工作线程并从线程池中移除该Worker，因为此处并非调用方直接调用，所以这个异常会被线程池吃掉，无法跟踪记录该异常栈，我们可以通过扩展点afterExecute来获取抛出的异常，做相应的监控和记录，以便于排查问题。
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);//线程池的扩展点，用于执行每个任务之后执行。
                    }
                } finally {
                    task = null;//清空当前task
                    w.completedTasks++;//该工作线程Worker完成过的任务数量
                    w.unlock();//释放锁
                }
            }
            completedAbruptly = false;//正常退出则设置completedAbruptly为false
        } finally {
            processWorkerExit(w, completedAbruptly);//处理Worker退出的逻辑
        }
    }

    private Runnable getTask() {
        boolean timedOut = false; //代表当前getTask方法上次poll是否超时未能获取到task对象

        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {//如果线程池状态大于等于STOP，或者是SHUTDOWN且任务队列是空
                decrementWorkerCount();//减少线程数量
                return null;//返回null后，Worker正常退出
            }

            boolean timed;      //是否允许超时退出标记

            for (;;) {//此处内层循环是为了设置是否允许超时方式获取任务，并且判断当前是否可以以超时的方式退出
                int wc = workerCountOf(c);
                timed = allowCoreThreadTimeOut || wc > corePoolSize;//如果核心线程允许超时退出，或者线程数量大于核心线程数，则允许该次获取任务超时退出

                if (wc <= maximumPoolSize && ! (timedOut && timed))
                    break;//如果当前线程数量小于等于最大线程且并未超时或者当前线程池不允许超时退出，则跳出内层循环
                if (compareAndDecrementWorkerCount(c))//如果当前线程数大于最大线程且当前已经超时为获取到且当前允许超时退出，则减少线程数量，成功后直接返回，失败后重试
                    return null;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)//如果线程池运行状态变化则重新执行外层循环，外层循环会做状态判断，因为CAS失败可能是线程池数量发生变化，则继续内存循环重试
                    continue retry;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();//如果允许超时方式获取，则用poll，否则使用take方法，从任务队列中获取任务
                if (r != null)//获取到任务则直接返回该任务，否则设置timedOut标记为已经超时没获取到
                    return r;
                timedOut = true;//设置timedOut标记为已经超时没获取到，然后重新执行外层和内存循环，判断状态和是否超时退出Worker
            } catch (InterruptedException retry) {
                timedOut = false;//被中断不认为超时没获取到，有可能是线程池关闭，重新调用外层循环判断获取。
            }
        }
    }

    private void processWorkerExit(Worker w, boolean completedAbruptly) {//completedAbruptly为true代表Worker因为业务异常退出，false代表正常退出
        if (completedAbruptly) //如果Worker是异常退出，则需要在此处将线程池数量减一。正常退出则会在getTask方法减少线程数量
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();//获取主锁
        try {
            completedTaskCount += w.completedTasks;//将退出的线程完成过的任务计数加到线程池总完成任务计数上
            workers.remove(w);//从工作线程集合中移除该工作线程
        } finally {
            mainLock.unlock();//释放锁
        }

        tryTerminate();//尝试终止线程池，因为这个Worker可能是当前线程池中最后一个Worker，tryTerminate方法在所有可能终止当前线程的地方被调用

        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {//如果线程池没有停止
            if (!completedAbruptly) {//如果Worker是正常退出
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;//如果允许核心线程超时退出，则线程池最小线程数可以为0，否则最小线程数为corePoolSize
                if (min == 0 && ! workQueue.isEmpty())//如果最小线程为0且任务队列不为空，则至少保证池中有一个线程可以处理任务队列中的任务
                    min = 1;
                if (workerCountOf(c) >= min)//如果线程池中的线程数大于等于最小线程，则什么都不做，否则创建工作线程。此处为了工作线程正常退出时，保证池中至少存有核心线程数量的线程
                    return;
            }
            addWorker(null, false);//创建一个工作线程。正常退出时是为了保证池中至少存有核心线程数量的线程；异常退出则直接重新补充一个工作线程。
        }
    }
```

## ThreadPoolExecutor核心方法execute()方法源码分析

### public void	execute(Runnable command)

在将来某个时间执行给定任务。该方法直接返回不等待任务执行完成，异步处理任务。

```
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {//线程数量小于核心线程数量，则直接添加线程并处理任务
            if (addWorker(command, true))//添加工作线程并处理任务，添加成功直接返回。
                return;
            c = ctl.get();//添加失败重新获取线程池状态计数
        }
        if (isRunning(c) && workQueue.offer(command)) {//如果线程池处于运行中状态，则添加任务到任务队列中等待工作线程处理
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))//添加成功重新校验状态，因为在添加到任务队列的时候线程池状态可能发生变化。如果不是运行中，则从任务队列中移除该任务
                reject(command);//移除成功则调用handler做线程池饱和的相应处理
            else if (workerCountOf(recheck) == 0)//如果线程池线程数量是0，则添加一个空的非核心工作线程
                addWorker(null, false);
        }
        else if (!addWorker(command, false))//如果添加队列失败或者不在运行中，说明任务队列已满，则添加非核心线程，添加失败则调用handler做线程池饱和的相应处理
            reject(command);//调用handler做线程池饱和的相应处理
    }

    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))//表示ctl状态为RUNNING状态或者为SHUTDOWN状态且此时任务队列仍有任务未执行完时，可以继续调用addWorker添加工作线程，但不能新建任务，即firstTask参数必须为null.否则这里将返回false,即新建工作线程失败。
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))//如果当前线程数量超过最大容许CAPACITY，或者核心线程超出corePoolSize，或者非核心线程超出maximumPoolSize，都返回失败。
                    return false;
                if (compareAndIncrementWorkerCount(c))//原子自增一次计数器，自增成功代表线程数量增加，后续将创建线程。成功则直接跳出循环，否则重试直到返回
                    break retry;//自增成功则跳出循环
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)//自增失败则重新读取线程状态，如果线程状态和进入时发生变化，则也跳出循环。
                    continue retry;
            }
        }

        boolean workerStarted = false;//工作线程是否启动
        boolean workerAdded = false;//工作线程是否成功添加
        Worker w = null;
        try {
            final ReentrantLock mainLock = this.mainLock;
            w = new Worker(firstTask);//创建一个工作线程管理者Worker，里面会初始化一个线程，初始化的时候会附带第一个启动后将要执行的任务
            final Thread t = w.thread;//获取Worker管理的线程
            if (t != null) {
                mainLock.lock();//首先获取主锁
                try {
                    int c = ctl.get();
                    int rs = runStateOf(c);

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {//校验线程池状态，运行中，或者SHUTDOWN且没有任务（这个时候还新建Worker是为了处理任务队列中可能还存在的任务）
                        if (t.isAlive()) //校验任务没有被提前启动，已经提前启动则抛出异常
                            throw new IllegalThreadStateException();
                        workers.add(w);//添加Worker到工作线程集合
                        int s = workers.size();//获取队列大小，用来记录线程池中出现过的最大线程数
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;//设置标记为成功添加工作线程
                    }
                } finally {
                    mainLock.unlock();//释放主锁
                }
                if (workerAdded) {//如果成功添加工作线程则启动线程，并设置标记为启动工作线程成功
                    t.start();
                    workerStarted = true;//设置标记为启动工作线程成功，至此为止一个worker正式被添加进入workers数组并且正式开始运转
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);//启动失败则移除该工作线程
        }
        return workerStarted;//创建并添加，并启动工作线程成功才算完全成功。
    }
```

### public &lt;T&gt; List&lt;Future&lt;T&gt;&gt; invokeAll(Collection&lt;? extends Callable&lt;T&gt;&gt; tasks, long timeout, TimeUnit unit) throws InterruptedException

执行给定的任务，当所有任务完成或超时期满时（无论哪个首先发生），返回保持任务状态和结果的 Future 列表。返回列表的所有元素的 Future.isDone() 为 true。一旦返回后，即取消尚未完成的任务。注意，可以正常地或通过抛出异常来终止已完成的任务。如果此操作正在进行时修改了给定的 collection，则此方法的结果是不确定的。

```
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException {
        if (tasks == null || unit == null)
            throw new NullPointerException();
        long nanos = unit.toNanos(timeout);//总超时时间
        List<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
        boolean done = false;
        try {
            for (Callable<T> t : tasks)
                futures.add(newTaskFor(t));//将所有任务包装成一个个Future，顺序和tasks中任务顺序一致

            long lastTime = System.nanoTime();
            
            Iterator<Future<T>> it = futures.iterator();
            while (it.hasNext()) {//执行所有任务
                execute((Runnable)(it.next()));
                long now = System.nanoTime();
                nanos -= now - lastTime;
                lastTime = now;
                if (nanos <= 0)
                    return futures;
            }

            for (Future<T> f : futures) {//循环获取任务结果，是为了检测是否任务能在nanos纳秒内执行完成。如果有任务超时没有完成，则会在finally中取消所有能够取消的未完成任务
                if (!f.isDone()) {
                    if (nanos <= 0)
                        return futures;
                    try {
                        f.get(nanos, TimeUnit.NANOSECONDS);
                    } catch (CancellationException ignore) {
                    } catch (ExecutionException ignore) {
                    } catch (TimeoutException toe) {
                        return futures;
                    }
                    long now = System.nanoTime();
                    nanos -= now - lastTime;
                    lastTime = now;
                }
            }
            done = true;//正常完成所有任务
            return futures;
        } finally {
            if (!done)//如果没有完成则取消所有能够取消的未完成任务，已经完成的任务不会被取消。保证返回的futures集合都能够直接从Future.get()方法立即返回结果（正常或异常结果）。
                for (Future<T> f : futures)
                    f.cancel(true);
        }
    }
```

## ThreadPoolExecutor关闭

### public void shutdown()

关闭线程池，仍会处理任务队列中的任务，但是不接受新任务。如果已经关闭，则调用没有其他作用。

```
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();//校验是否有执行shutdown的权限
            advanceRunState(SHUTDOWN);//将ctl状态置为SHUTDOWN
            interruptIdleWorkers();//关闭尚未获得task对象的worker(即还未执行到getTask()方法或者还未得到getTask()返回的worker
            onShutdown(); //钩子方法，用户可以在这里处理自定义逻辑。
        } finally {
            mainLock.unlock();
        }
        tryTerminate();//尝试终止线程池。
    }

    private void interruptIdleWorkers() {//中断空闲的Worker
        interruptIdleWorkers(false);
    }

    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();//获取主锁
        try {
            for (Worker w : workers) {//遍历所有workers
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) {//如果当前Worker线程没有被中断，且尝试获取锁成功（代表任务没有在线程执行中）
                    try {
                        t.interrupt();//中断该线程
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();//解锁Worker
                    }
                }
                if (onlyOne)//true代表只中断第一个空闲的Worker
                    break;
            }
        } finally {
            mainLock.unlock();//解锁主锁
        }
    }

    final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))//线程池运行中或者已经终止，或者SHUTDOWN且任务队列不为空，直接返回（SHUTDOWN且任务队列不为空时，会在Worker退出时调用tryTerminate方法终止线程池）
                return;
            if (workerCountOf(c) != 0) { //工作线程不为0则中断一个工作线程，直接返回。（STOP时，任务正在运行中，运行完成后，会在Worker退出时调用tryTerminate方法终止线程池）
                interruptIdleWorkers(ONLY_ONE);
                return;
            }
            //ctl状态为STOP,或者为SHUTDOWN且任务队列为空，并且工作线程为0，才继续执行

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();//获取主锁
            try {
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {//将状态设置为TIDYING，执行到这一步意味着工作线程为0，任务队列空，状态为STOP或SHUTDOWN。
                    try {
                        terminated();//线程池扩展点，终止线程池时执行
                    } finally {
                        ctl.set(ctlOf(TERMINATED, 0));//执行完terminated后将状态设置为TERMINATED
                        termination.signalAll();//唤醒等待在awaitTermination方法上的线程，awaitTermination是等待线程池终止的方法，阻塞直到线程池终止或者超时，中断
                    }
                    return;//退出，否则将会for循环重试
                }
            } finally {
                mainLock.unlock();//获取主锁
            }
        }
    }
```

### public List`<Runnable>` shutdownNow()

尝试停止所有的活动执行任务、停止任务队列的处理，并返回等待执行的任务列表。并不保证能够停止正在处理的活动执行任务，但是会尽力尝试。 此实现通过 Thread.interrupt() 取消任务，所以无法响应中断的任何任务可能永远无法终止。

```
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            interruptIdleWorkers();//关闭尚未获得task对象的

            checkShutdownAccess();//校验是否有执行
            advanceRunState(STOP);//将ctl状态置为STOP
            interruptWorkers();//关闭所有启动了的的worker，中断所有worker的线程
            tasks = drainQueue();//清空任务队列并返回未执行的任务列表
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;//返回未执行的任务列表
    }
    
    private List<Runnable> drainQueue() {
        BlockingQueue<Runnable> q = workQueue;
        List<Runnable> taskList = new ArrayList<Runnable>();
        q.drainTo(taskList);//将workQueue中的任务转移到taskList返回
        if (!q.isEmpty()) {
            for (Runnable r : q.toArray(new Runnable[0])) {
                if (q.remove(r))
                    taskList.add(r);
            }
        }
        return taskList;
    }

```

## ThreadPoolExecutor扩展点

ThreadPoolExecutor提供了几个protected的方法，在任务执行前，执行后，和线程池终止时执行。可以在自定义线程池中自由实现，用来记录和监控线程池的运行情况。

```
    protected void beforeExecute(Thread t, Runnable r) { }//在执行给定线程中的给定 Runnable 之前调用的方法。

    protected void afterExecute(Runnable r, Throwable t) { }//基于完成执行给定 Runnable 所调用的方法。

    protected void terminated() { }//当 Executor 已经终止时调用的方法。
```
---
title: Java的强引用，软引用，弱引用，虚引用及其使用场景
date: 2017-09-16 19:00:00
categories: Java
tags: [Java]
---

Java中有如下四种类型的引用：

* 强引用(Strong Reference)
* 软引用(SoftReference)
* 弱引用(WeakReference)
* 虚引用(PhantomReference)

四种引用的级别由高到低依次为：

	 Strong Reference > SoftReference > WeakReference > PhantomReference

<!--more-->


## 强引用(Strong Reference)

强引用是我们编程中最常使用的引用，任何被强引用所引用的对象都不能被垃圾回收器回收，垃圾回收时，也是根据强引用去做的可达性分析。当内存空间不足，Java虚拟机会抛出OutOfMemoryError错误，使程序异常终止，所以在使用强引用的时候，需要注意对象的大小和生命周期防止发生OOM，尤其是使用一些Map，List之类的成员变量时更要注意。

```java
    @Test
    public void testStrongReference() {
        Object obj = new Object();
    }
```

变量obj就是该Object对象的一个强引用。只有强引用消失，对象才能被GC回收。


## 软引用(SoftReference)

一个对象只有软引用，如果内存空间不足，就会回收这些对象的内存；如果内存空间足够，垃圾回收器不会回收它。只要垃圾回收器没有回收它，该对象就可以被使用。

软引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被垃圾回收器回收，Java虚拟机就会把这个软引用加入到与之关联的引用队列中。

### 软引用何时被回收

首先看下SoftReference实现：

```java
public class SoftReference<T> extends Reference<T> {

    /**
     * Timestamp clock, updated by the garbage collector
     */
    static private long clock;

    /**
     * Timestamp updated by each invocation of the get method.  The VM may use
     * this field when selecting soft references to be cleared, but it is not
     * required to do so.
     */
    private long timestamp;

    /**
     * Creates a new soft reference that refers to the given object.  The new
     * reference is not registered with any queue.
     *
     * @param referent object the new soft reference will refer to
     */
    public SoftReference(T referent) {
        super(referent);
        this.timestamp = clock;
    }

    /**
     * Creates a new soft reference that refers to the given object and is
     * registered with the given queue.
     *
     * @param referent object the new soft reference will refer to
     * @param q the queue with which the reference is to be registered,
     *          or <tt>null</tt> if registration is not required
     *
     */
    public SoftReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
        this.timestamp = clock;
    }

    /**
     * Returns this reference object's referent.  If this reference object has
     * been cleared, either by the program or by the garbage collector, then
     * this method returns <code>null</code>.
     *
     * @return   The object to which this reference refers, or
     *           <code>null</code> if this reference object has been cleared
     */
    public T get() {
        T o = super.get();
        if (o != null && this.timestamp != clock)
            this.timestamp = clock;
        return o;
    }

}
```

SoftReference中有一个全局变量clock代表最后一次GC的时间点，有一个属性timestamp，每次访问SoftReference时，会将timestamp其设置为clock值。

当GC发生时，以下几个因素影响SoftReference引用的对象是否被回收：

1. SoftReference对象实例多久未访问，通过`clock - timestamp`得出对象大概有多久未访问；
2. 内存空闲空间的大小；
3. SoftRefLRUPolicyMSPerMB常量值；

是否保留SoftReference引用对象的判断参考表达式，true为不回收，false为回收：

	clock - timestamp <= freespace * SoftRefLRUPolicyMSPerMB 


说明：

clock - timestamp：最后一次GC时间和SoftReference对象实例timestamp的属性的差。就是这个SoftReference引用对象大概有多久未访问过了。

freespace：JVMHeap中空闲空间大小，单位为MB。

SoftRefLRUPolicyMSPerMB：每1M空闲空间可保持的SoftReference对象生存的时长（单位ms）。这个参数就是一个常量，默认值1000，可以通过参数：`-XX:SoftRefLRUPolicyMSPerMB`进行设置。


简单地理解就是：如果SoftReference引用对象未访问的时长小于等于空闲内存可保持软引用的最大时间范围，则不清除SoftReference所引用的对象；否则，则将其清除。空闲内存越大，SoftRefLRUPolicyMSPerMB越大，对象越不容易被回收；对象越久未被访问，越容易被回收。

SoftReference会至少经历1次gc而不被回收，可以看下面的例子：

```java
    //-XX:SoftRefLRUPolicyMSPerMB=0
    @Test
    public void testSoftReference() throws Exception {
        Object obj = new Object();
        SoftReference<Object> softReference = new SoftReference<>(obj);

        obj = null;

        System.gc();
        Thread.sleep(3000l);//obj未被回收
        
        System.gc();
        Thread.sleep(3000l);
        System.out.println(softReference.get());//obj被回收

    }
```

在设置了`-XX:SoftRefLRUPolicyMSPerMB=0`的前提下，当执行`obj = null;`后，第一次GC时，`clock - timestamp = 0`，`freespace * SoftRefLRUPolicyMSPerMB = 0`，表达式结果为true，不回收；第二次回收时，`clock - timestamp > 0`，`freespace * SoftRefLRUPolicyMSPerMB = 0`，表达式结果为false，将会回收掉obj。

### 使用场景

软引用一般用做堆内本地缓存，当内存不足时，垃圾回收器就会回收这些只被软引用指向的对象，通过这种机制可以去控制缓存的淘汰过期，防止发生OOM，但是使用软引用做为缓存时，对于缓存结构整体大小，缓存的过期时间，淘汰策略，都不好加以灵活控制，所以一般本地缓存还是会使用一些guava cache之类的更加可控一些的本地缓存。对于一些大小可控的资源，还是可以使用软引用作为缓存的，具体根据使用场景灵活评估。


## 弱引用(WeakReference)

只具有弱引用的对象拥有更短的生命周期。在垃圾回收时，如果发现对象只有弱引用，不管当前内存空间是否足够，都会直接回收它。相比软引用，弱引用的回收时机更可控。

弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java虚拟机就会把这个弱引用加入到与之关联的引用队列中。

```java
    @Test
    public void testWeakReference() throws Exception {
        Object obj = new Object();
        WeakReference<Object> weakReference = new WeakReference<>(obj);

        System.out.println(weakReference.get());

        System.gc();
        Thread.sleep(3000l);
        System.out.println(weakReference.get());//可以获取到obj

        obj = null;

        System.gc();
        Thread.sleep(3000l);
        System.out.println(weakReference.get());//获取不到obj，已经被GC回收
    }
```

当执行完`obj = null;`时，obj已经没有强引用了，再做完GC后，弱引用weakReference引用的对象obj就已经被GC回收掉了。

### 使用场景

当我们需要引用一个对象，又希望不影响对象自己的生命周期，同时当对象不可达的时候，可以自动回收淘汰掉，那么强引用就不合适，因为强引用会影响对象的可达性，这时候就可以使用弱引用，当对象只有弱引用，垃圾回收的时候就会回收它。

弱引用非常适合存储元数据，例如：存储ClassLoader引用。如果ClassLoader还可达，那么我们可以获取到该ClassLoader的数据以及相关数据，但如果ClassLoader不可达了，那么只有弱引用的ClassLoader就会被自动回收。具体比如JDK的Proxy类中有一个WeakCache，WeakCache类似一个map的结构，它的key是ClassLoader，value是所有跟该ClassLoader相关的动态类，当WeakCache中的ClassLoader对象不可达了，那么所有相关数据就会从WeakCache中淘汰掉。

## 虚引用(PhantomReference)

虚引用并不会决定对象的生命周期。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，对象不可达时就会被垃圾回收器回收，但是任何时候都无法通过虚引用获得对象。虚引用主要用来跟踪对象被垃圾回收器回收的活动。

虚引用与软引用和弱引用的一个区别在于：虚引用必须和引用队列（ReferenceQueue）联合使用。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中。

```java
    @Test
    public void testPhantomReference() throws Exception {
        Object obj = new Object();
        ReferenceQueue refQueue = new ReferenceQueue();
        PhantomReference<Object> phantomReference = new PhantomReference<Object>(obj, refQueue);
        System.out.println(phantomReference.get());//null
    }
```

### 使用场景

虚引用主要用来跟踪对象被垃圾回收器回收的活动。一般可以通过虚引用达到回收一些非java内的一些资源比如堆外内存的行为。例如：在DirectByteBuffer中，会创建一个PhantomReference的子类Cleaner的虚引用实例用来引用该DirectByteBuffer实例，Cleaner创建时会添加一个Runnable实例，当被引用的DirectByteBuffer对象不可达被垃圾回收时，将会执行Cleaner实例内部的Runnable实例的run方法，用来回收堆外资源。

```java
public class Cleaner extends PhantomReference {
    private static final ReferenceQueue dummyQueue = new ReferenceQueue();
    private static Cleaner first = null;
    private Cleaner next = null;
    private Cleaner prev = null;
    private final Runnable thunk;

    private static synchronized Cleaner add(Cleaner var0) {
        if(first != null) {
            var0.next = first;
            first.prev = var0;
        }

        first = var0;
        return var0;
    }

    private static synchronized boolean remove(Cleaner var0) {
        if(var0.next == var0) {
            return false;
        } else {
            if(first == var0) {
                if(var0.next != null) {
                    first = var0.next;
                } else {
                    first = var0.prev;
                }
            }

            if(var0.next != null) {
                var0.next.prev = var0.prev;
            }

            if(var0.prev != null) {
                var0.prev.next = var0.next;
            }

            var0.next = var0;
            var0.prev = var0;
            return true;
        }
    }

    private Cleaner(Object var1, Runnable var2) {
        super(var1, dummyQueue);
        this.thunk = var2;
    }

    public static Cleaner create(Object var0, Runnable var1) {
        return var1 == null?null:add(new Cleaner(var0, var1));
    }

    public void clean() {//当垃圾回收回收掉虚引用引用的对象时，如果虚引用是Cleaner实例，将会执行Cleaner的clean方法，进而执行Runnable实例的run方法进行一些额外的回收工作
        if(remove(this)) {
            try {
                this.thunk.run();
            } catch (final Throwable var2) {
                AccessController.doPrivileged(new PrivilegedAction() {
                    public Void run() {
                        if(System.err != null) {
                            (new Error("Cleaner terminated abnormally", var2)).printStackTrace();
                        }

                        System.exit(1);
                        return null;
                    }
                });
            }

        }
    }
}

```

当垃圾回收回收掉虚引用引用的对象时，如果虚引用是Cleaner实例，将会执行Cleaner的clean方法，进而执行内部的Runnable实例的run方法进行一些额外的回收工作。

```java
    private static class ReferenceHandler extends Thread {//会有一个高优先级的线程执行垃圾回收后的额外工作

        ReferenceHandler(ThreadGroup g, String name) {
            super(g, name);
        }

        public void run() {
            for (;;) {

                Reference r;
                synchronized (lock) {
                    if (pending != null) {
                        r = pending;
                        Reference rn = r.next;
                        pending = (rn == r) ? null : rn;
                        r.next = r;
                    } else {
                        try {
                            lock.wait();
                        } catch (InterruptedException x) { }
                        continue;
                    }
                }

                // Fast path for cleaners
                if (r instanceof Cleaner) {//如果被回收的是Cleaner实例的引用，将会执行Cleaner实例的clean方法进行一些额外的回收工作
                    ((Cleaner)r).clean();
                    continue;
                }

                ReferenceQueue q = r.queue;
                if (q != ReferenceQueue.NULL) q.enqueue(r);
            }
        }
    }
```
 

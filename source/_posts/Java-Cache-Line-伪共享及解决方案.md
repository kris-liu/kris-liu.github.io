---
title: Java Cache Line 伪共享及解决方案
date: 2016-11-15 17:42:19
categories: [Java]
tags: [Java,并发,缓存]
---
## 伪共享问题本质：
**多个变量被cpu加载在同一个缓存行中，当在多线程环境下，多个变量被不同的cpu执行，导致缓存行失效而引起的大量的缓存命中率降低。**

缓存一致性协议MESI协议中，每个CPU的Cache控制器不仅知道自己的读写操作，而且也监听其它CPU的读写操作。每个Cache line所处的状态根据本核和其它核的读写操作在4个状态间进行迁移。其他CPU的写会导致本CPU的Cache无效，下次访问时如果CPU访问的内存数据不在CpuCache中，这就产生了Cache Line miss问题，此时CPU不得不发出新的加载指令，从内存中获取数据。一旦CPU要从内存中访问数据就会产生一个较大的时延，程序性能显著降低。为此我们不得不提高Cache命中率，也就是充分发挥局部性原理。

局部性包括时间局部性、空间局部性。时间局部性：对于同一数据可能被多次使用，自第一次加载到Cache Line后，后面的访问就可以多次从Cache Line中命中，从而提高读取速度。空间局部性：一个Cache Line有64字节块，我们可以充分利用一次加载64字节的空间，把程序后续会访问的数据，一次性全部加载进来，从而提高Cache Line命中率（而不是重新去寻址读取）。

## 测试伪共享缓存行：

```
public final class FalseSharing implements Runnable {
    public static int NUM_THREADS = 4;
    public final static long ITERATIONS = 500L * 1000L * 1000L;
    private final int arrayIndex;
    private static VolatileLong[] longs;

    public FalseSharing(final int arrayIndex) {
        this.arrayIndex = arrayIndex;
    }

    public static void main(final String[] args) throws Exception {
        System.out.println("starting....");
        longs = new VolatileLong[NUM_THREADS];
        for (int i = 0; i < longs.length; i++) {
            longs[i] = new VolatileLong();
        }
        final long start = System.nanoTime();
        runTest();
        System.out.println("duration = " + (System.nanoTime() - start));
    }

    private static void runTest() throws InterruptedException {
        Thread[] threads = new Thread[NUM_THREADS];
        for (int i = 0; i < threads.length; i++) {
            threads[i] = new Thread(new FalseSharing(i));
        }
        for (Thread t : threads) {
            t.start();
        }
        for (Thread t : threads) {
            t.join();
        }
    }

    public void run() {
        long i = ITERATIONS;
        while (0 != --i) {
            longs[arrayIndex].value = i;
        }
    }

    @Contended //注释用来测试
    public final static class VolatileLong {
        public volatile long value = 0L;
//        public long p1, p2, p3, p4, p5, p6;

//        public long getValue() {
//            return p1 + p2 + p3+ p4 +p5+p6;
//        }
    }
}
```

java6,7下使用p1, p2, p3, p4, p5, p6这些额外变量使多个相邻value变量不在一个缓存行上；
java8下使用@Contended注解加-XX:-RestrictContended启动参数使变量在独立的缓存行

### 测试结果：
独立缓存行时间 : 7199674127 ns
共享缓存行时间 : 26970122072 ns
共享缓存行性能低。

### 结论：
多个共享变量各自被多个线程操作时，最好把多个变量分布在不同的缓存行中，防止一个线程对其中一个变量修改导致其他线程中的该缓存行失效，而从主存中加载数据带来的性能损耗。

## 解决方式：
一般可以用填充的手段来将多个变量分布在不同的缓存行中。
java6,7下使用额外变量使多个相邻value变量不在一个缓存行上，需要防止编译器优化掉额外变量一般还需要加入一个无用方法操作这几个额外变量；
 java8下使用@Contended注解加-XX:-RestrictContended启动参数，使变量在独立的缓存行。

## 参考资料：

[CPU Cache与高性能编程](http://geek.csdn.net/news/detail/114619)

[Cache一致性协议之MESI](http://blog.csdn.net/muxiqingyang/article/details/6615199)
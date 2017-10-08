---
title: ThreadLocal 源码分析
date: 2016-11-18 19:15:44
categories: Java
tags: [Java, 源码]
---

ThreadLocal线程局部变量，使得各线程能够保持各自独立的一份对象。通常被定义为类的静态类变量。

ThreadLocal类本身定义了有get(), set(), remove()和initialValue()方法。前面三个方法是public的，initialValue()是protected的，主要用于我们在定义ThreadLocal对象的时候根据需要来重写。这样我们初始化这么一个对象在里面设置它的初始值时就用到这个方法。ThreadLocal变量因为本身定位为要被多个线程来访问，它通常被定义为static变量。

<!--more-->

# ThreadLocal API

## 方法
 T	get() 
          返回此线程局部变量的当前线程副本中的值。
          
 protected  T	initialValue() 
          返回此线程局部变量的当前线程的“初始值”。
          
 void	remove() 
          移除此线程局部变量当前线程的值。
          
 void	set(T value) 
          将此线程局部变量的当前线程副本中的值设置为指定值。
          

# 源码分析
ThreadLocal有一个ThreadLocalMap静态内部类，ThreadLocalMap的实例是java.lang.Thread的成员变量，每个线程有唯一的一个threadLocalMap。这个map以ThreadLocal对象为key，实际存储内容为值。对ThreadLocal的操作，实际委托给当前Thread，每个Thread都会有自己独立的ThreadLocalMap实例。ThreadLocalMap使用线性探测法解决hash冲突。

## ThreadLocal

### T	get() 
返回此线程局部变量的当前线程副本中的值。

```java
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);//活动当前Thread中的ThreadLocalMap
        if (map != null) {//不为空则从ThreadLocalMap中get当前ThreadLocal对应的value
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null)
                return (T)e.value;
        }
        //为空或者get不到，则调用setInitialValue初始化
        return setInitialValue();
    }
    
    private T setInitialValue() {
        T value = initialValue();//初始化方法
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);//获取当前Thread中的ThreadLocalMap
        if (map != null)//不为空则set
            map.set(this, value);
        else//为空则初始化并set
            createMap(t, value);
        return value;//返回初始化的值
    }
```
          
### protected  T	initialValue() 
返回此线程局部变量的当前线程的“初始值”。
	
```java
    protected T initialValue() {//初始化方法，由子类实现
        return null;
    }
```

      
### void	remove() 
移除此线程局部变量当前线程的值。
	
```java
     public void remove() {//获取当前线程中的ThreadLocalMap，并从中remove掉当前ThreadLocal实例对应的value
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }
```

### void	set(T value) 
将此线程局部变量的当前线程副本中的值设置为指定值。     
	  
```java
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);//获取当前线程中的ThreadLocalMap，存在则直接set，不存在则初始化并set
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

```java
	ThreadLocalMap getMap(Thread t) {//ThreadLocalMap是Thread实例的变量
        return t.threadLocals;
    }
    
    void createMap(Thread t, T firstValue) {//创建并set
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

## ThreadLocalMap

```java
    static class ThreadLocalMap {

        static class Entry extends WeakReference<ThreadLocal> {//ThreadLocalMap每个单元Entry，是WeakReference类型，当ThreadLocal没有强引用，则GC时会被回收，防止内存泄漏
            Object value;

            Entry(ThreadLocal k, Object v) {
                super(k);
                value = v;
            }
        }

        private static final int INITIAL_CAPACITY = 16;//初始大小

        private Entry[] table;//map的节点数组

        private int size = 0;//ThreadLocalMap中有多少key

        private int threshold; //扩容阈值，超出则扩大table

        private void setThreshold(int len) {//扩容方式一般是超出当前table长度的三分之二则扩容
            threshold = len * 2 / 3;
        }

        private static int nextIndex(int i, int len) {//线性探测法法，自增方式hash后定位的位置＋1
            return ((i + 1 < len) ? i + 1 : 0);
        }

        private static int prevIndex(int i, int len) {//前一位置
            return ((i - 1 >= 0) ? i - 1 : len - 1);
        }
        
        ThreadLocalMap(ThreadLocal firstKey, Object firstValue) {//初始化table长度为INITIAL_CAPACITY，通过ThreadLocal的threadLocalHashCode&长度－1得到当前ThreadLocal的桶的位置。
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }

        private ThreadLocalMap(ThreadLocalMap parentMap) {//复制ThreadLocalMap，一般用于InheritableThreadLocal初始化
            Entry[] parentTable = parentMap.table;
            int len = parentTable.length;
            setThreshold(len);
            table = new Entry[len];

            for (int j = 0; j < len; j++) {
                Entry e = parentTable[j];
                if (e != null) {
                    ThreadLocal key = e.get();
                    if (key != null) {
                        Object value = key.childValue(e.value);
                        Entry c = new Entry(key, value);
                        int h = key.threadLocalHashCode & (len - 1);
                        while (table[h] != null)
                            h = nextIndex(h, len);
                        table[h] = c;
                        size++;
                    }
                }
            }
        }

        private Entry getEntry(ThreadLocal key) {//获取ThreadLocal对应value
            int i = key.threadLocalHashCode & (table.length - 1);//定位算法
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;//获取到则直接返回
            else
                return getEntryAfterMiss(key, i, e);
        }

        private void set(ThreadLocal key, Object value) {//定位ThreadLocal的下标，然后set，冲突则用线性探测法，找到一个可用下标，超出一定的key数量则rehash

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }

        private void remove(ThreadLocal key) {//移除ThreadLocal对应的value
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {
                    e.clear();
                    expungeStaleEntry(i);
                    return;
                }
            }
        }

        private void rehash() {
            expungeStaleEntries();
            if (size >= threshold - threshold / 4)//key数量超出3/4则resize
                resize();
        }

        private void resize() {//创建两倍于原来的table，并重新set所有Key
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;
            Entry[] newTab = new Entry[newLen];
            int count = 0;

            for (int j = 0; j < oldLen; ++j) {
                Entry e = oldTab[j];
                if (e != null) {
                    ThreadLocal k = e.get();
                    if (k == null) {
                        e.value = null; // Help the GC
                    } else {
                        int h = k.threadLocalHashCode & (newLen - 1);
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);
                        newTab[h] = e;
                        count++;
                    }
                }
            }

            setThreshold(newLen);
            size = count;
            table = newTab;
        }
        
    }
```

## ThreadLocalMap的定位方法：

**int index = threadLocalHashCode & (len - 1)**

HASH_INCREMENT = 0x61c88647

每个ThreadLocal中的threadLocalHashCode都是HASH_INCREMENT的倍数，0x61c88647这个数字&上2的n次方-1的数字，hash分布十分的均匀。

```java
public class ThreadLocal<T> {

    private final int threadLocalHashCode = nextHashCode();

    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    private static final int HASH_INCREMENT = 0x61c88647;

    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }

	... ...
}
```

### 测试hash分布

```java
public class ThreadLocalTest {

    public static void main(String[] args) {
        int HASH_INCREMENT = 0x61c88647;
        AtomicInteger atomicInteger = new AtomicInteger();
        for (int i = 0; i < 32; i++) {
            int hash = atomicInteger.getAndAdd(HASH_INCREMENT);
            System.out.println(hash & 31);
        }
    }

}
```

输出：
0
7
14
21
28
3
10
17
24
31
6
13
20
27
2
9
16
23
30
5
12
19
26
1
8
15
22
29
4
11
18
25
可以看到几乎没有发生hash冲突。

# ThreadLocal使用方式

```java
public class ThreadLocalTest {

    private static class Count {
        private int count;

        public int getCount() {
            return count;
        }

        public void setCount(int count) {
            this.count = count;
        }
    }

    public static final ThreadLocal<Count> threadLocal = new ThreadLocal<Count>() {
        @Override
        protected Count initialValue() {
            return new Count();
        }
    };
｝
```

## 使用建议

1. ThreadLocal应定义为静态成员变量。
2. 能通过传值传递的参数，不要通过ThreadLocal存储，以免造成ThreadLocal的滥用。
3. 在线程池的情况下，在ThreadLocal业务周期处理完成时，最好显式的调用remove()方法。

# 参考

[深入JDK源码之ThreadLocal类](https://my.oschina.net/xianggao/blog/392440?fromerr=CLZtT4xC)
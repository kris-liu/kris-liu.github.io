---
title: Java NIO Buffer详解
date: 2017-01-14 23:47:59
categories: IO&NIO
tags: [Java, NIO]
---

## Buffer缓冲区

缓冲区提供了对数据的结构化访问，而且还可以跟踪系统的读/写进程。所有的缓冲区都具有四个属性来提供关于其所包含的数据元素的信息。它们是: 

- 容量(Capacity) 

	缓冲区能够容纳的数据元素的最大数量。这一容量在缓冲区创建时被设定,并且永远不能被改变。 

- 上界(Limit) 

	缓冲区的第一个不能被读或写的元素。或者说,缓冲区中现存元素的计数。 

- 位置(Position) 

	下一个要被读或写的元素的索引。位置会自动由相应的get()和put()函数更新。 

- 标记(Mark)

	一个备忘位置。调用mark()来设定mark=postion。调用reset()设定position=mark。标记在设定前是未定义的(undefined)。 

<!--more-->

这四个属性之间总是遵循以下关系: 
0 <= mark <= position <= limit <= capacity


使用Buffer读写数据一般遵循以下四个步骤：

1. 写入数据到Buffer
2. 调用flip()方法
3. 从Buffer中读取数据
4. 调用clear()方法或者compact()方法

当向buffer写入数据时，buffer会记录下写了多少数据。一旦要读取数据，需要通过flip()方法将Buffer从写模式切换到读模式。在读模式下，可以读取之前写入到buffer的所有数据。一旦读完了所有的数据，就需要清空缓冲区，让它可以再次被写入。有两种方式能清空缓冲区：调用clear()或compact()方法。clear()方法会清空整个缓冲区。compact()方法只会清除已经读过的数据。任何未读的数据都被移到缓冲区的起始处，新写入的数据将放到缓冲区未读数据的后面。

最常用的缓冲区类型是ByteBuffer。一个ByteBuffer可以在其底层字节数组上进行get/set操作(即字节的获取和设置)。
ByteBuffer不是NIO中唯一的缓冲区类型。对于每一种基本Java类型都有一种缓冲区类型：

- ByteBuffer
- CharBuffer
- ShortBuffer
- IntBuffer
- LongBuffer
- FloatBuffer
- DoubleBuffer

每一个Buffer类都是Buffer接口的一个实例。除了ByteBuffer，每一个Buffer类都有完全一样的操作，只是它们所处理的数据类型不一样。因为操作系统的IO是以字节为单位的，因此，字节缓冲区跟其他缓冲区不同，对操作系统的IO只能是基于字节缓冲区的ByteBuffer，所以它具有所有共享的缓冲区操作以及一些特有的操作。

ByteBuffer分为直接缓冲区和非直接缓冲区。

- 非直接缓冲区可以通过ByteBuffer.wrap(byte[] array);ByteBuffer.allocate(int capacity)这两类方法来创建

- 直接缓冲区可通过ByteBuffer.allocateDirect(int capacity)来创建

>字节缓冲区要么是直接的，要么是非直接的。如果为直接字节缓冲区，则Java虚拟机会尽最大努力直接在此缓冲区上执行本机I/O操作。也就是说，在每次调用基础操作系统的一个本机I/O操作之前（或之后），虚拟机都会尽量避免将缓冲区的内容复制到中间缓冲区中（或从中间缓冲区中复制内容）。

>对直接缓冲区进行分配和取消分配所需成本通常高于非直接缓冲区。直接缓冲区的内容可以驻留在常规的垃圾回收堆之外，因此，它们对应用程序的内存需求量造成的影响可能并不明显。所以，建议将直接缓冲区主要分配给那些易受基础系统的本机I/O操作影响的大型、持久的缓冲区。一般情况下，最好仅在直接缓冲区能在程序性能方面带来明显好处时分配它们。

>直接字节缓冲区还可以通过映射将文件区域直接映射到内存中来创建。Java平台的实现有助于通过JNI从本机代码创建直接字节缓冲区。  


## Buffer API

介绍缓冲区API

### 构造

```java
public static CharBuffer allocate (int capacity)public static CharBuffer wrap (char [] array)public static CharBuffer wrap (char [] array, int offset,     int length)
     
```

```java
    public static CharBuffer allocate(int capacity) {
        if (capacity < 0)
            throw new IllegalArgumentException();
        return new HeapCharBuffer(capacity, capacity);
    }
    
    public static CharBuffer wrap(char[] array,
                                    int offset, int length)
    {
        try {
            return new HeapCharBuffer(array, offset, length);
        } catch (IllegalArgumentException x) {
            throw new IndexOutOfBoundsException();
        }
    }
    
    public static CharBuffer wrap(char[] array) {
        return wrap(array, 0, array.length);
    }
```

Buffer通过allocate和wrap两类方法创建，通过这两类方法调用相应实现类的构造方法创建Buffer实例，最终调用父类Buffer的构造函数设置Buffer的pos，limit，cap，mark几个关键位置标记

```java
    Buffer(int mark, int pos, int lim, int cap) {       // package-private
        if (cap < 0)
            throw new IllegalArgumentException("Negative capacity: " + cap);
        this.capacity = cap;
        limit(lim);
        position(pos);
        if (mark >= 0) {
            if (mark > pos)
                throw new IllegalArgumentException("mark > position: ("
                                                   + mark + " > " + pos + ")");
            this.mark = mark;
        }
    }

```

*特别的是ByteBuffer，通过额外的allocateDirect方法可以创建一个直接缓冲区DirectByteBuffer，直接缓冲区借助unsafe类去操作直接内存区域。

```java
    DirectByteBuffer(int cap) {                   // package-private

        super(-1, 0, cap, cap);
        boolean pa = VM.isDirectMemoryPageAligned();
        int ps = Bits.pageSize();
        long size = Math.max(1L, (long)cap + (pa ? ps : 0));
        Bits.reserveMemory(size, cap);

        long base = 0;
        try {
            base = unsafe.allocateMemory(size);
        } catch (OutOfMemoryError x) {
            Bits.unreserveMemory(size, cap);
            throw x;
        }
        unsafe.setMemory(base, size, (byte) 0);
        if (pa && (base % ps != 0)) {
            // Round up to page boundary
            address = base + ps - (base & (ps - 1));
        } else {
            address = base;
        }
        cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
        att = null;



    }
    
    private static class Deallocator
        implements Runnable
    {

        private static Unsafe unsafe = Unsafe.getUnsafe();

        private long address;
        private long size;
        private int capacity;

        private Deallocator(long address, long size, int capacity) {
            assert (address != 0);
            this.address = address;
            this.size = size;
            this.capacity = capacity;
        }

        public void run() {
            if (address == 0) {
                // Paranoia
                return;
            }
            unsafe.freeMemory(address);
            address = 0;
            Bits.unreserveMemory(size, capacity);
        }

    }
```

直接缓冲区DirectByteBuffer通过调用unsafe类的本地代码API在直接内存中分配一块内存，DirectByteBuffer类只记录直接内存的地址，和pos，limit，cap，mark几个关键位置标记，操作API通过unsafe类去操作底层的直接内存。

直接内存的释放是通过Cleaner.create创建了一个PhantomReference幽灵引用的实例Cleaner，Cleaner会在DirectByteBuffer实例被回收后调用其clean方法触发Deallocator执行，在Deallocator的run方法中通过unsafe释放之前在直接内存中分配的内存区域，达到在GC回收DirectByteBuffer实例的同时去释放其占用的直接内存。

### 获取和设置position，limit，capacity

```java
    public final int capacity() {
        return capacity;
    }

    public final int position() {
        return position;
    }
    
    public final Buffer position(int newPosition) {
        if ((newPosition > limit) || (newPosition < 0))
            throw new IllegalArgumentException();
        position = newPosition;
        if (mark > position) mark = -1;
        return this;
    }

    public final int limit() {
        return limit;
    }
    
    public final Buffer limit(int newLimit) {
        if ((newLimit > capacity) || (newLimit < 0))
            throw new IllegalArgumentException();
        limit = newLimit;
        if (position > limit) position = limit;
        if (mark > limit) mark = -1;
        return this;
    }
```

### 存取

```java
public abstract class ByteBuffer       extends Buffer implements Comparable{       // This is a partial API listing	public abstract byte get();	public abstract byte get (int index);	public abstract ByteBuffer put (byte b);	public abstract ByteBuffer put (int index, byte b);}
```

取

```java
    public byte get() {
        return hb[ix(nextGetIndex())];
    }

    public byte get(int i) {
        return hb[ix(checkIndex(i))];
    }
    
    final int nextGetIndex() {                          // package-private
        if (position >= limit)
            throw new BufferUnderflowException();
        return position++;
    }
    
    final int checkIndex(int i) {                       // package-private
        if ((i < 0) || (i >= limit))
            throw new IndexOutOfBoundsException();
        return i;
    }
    
    public ByteBuffer get(byte[] dst, int offset, int length) {
        checkBounds(offset, length, dst.length);
        if (length > remaining())
            throw new BufferUnderflowException();
        int end = offset + length;
        for (int i = offset; i < end; i++)
            dst[i] = get();
        return this;
    }
    
    public ByteBuffer get(byte[] dst) {
        return get(dst, 0, dst.length);
    }
```

存

```java
    public ByteBuffer put(byte x) {

        hb[ix(nextPutIndex())] = x;
        return this;
    }

    public ByteBuffer put(int i, byte x) {

        hb[ix(checkIndex(i))] = x;
        return this;
    }
    
    final int nextPutIndex() {                          // package-private
        if (position >= limit)
            throw new BufferOverflowException();
        return position++;
    }
    
    public ByteBuffer put(ByteBuffer src) {
        if (src == this)
            throw new IllegalArgumentException();
        int n = src.remaining();
        if (n > remaining())
            throw new BufferOverflowException();
        for (int i = 0; i < n; i++)
            put(src.get());
        return this;
    }
    
    public ByteBuffer put(byte[] src, int offset, int length) {
        checkBounds(offset, length, src.length);
        if (length > remaining())
            throw new BufferOverflowException();
        int end = offset + length;
        for (int i = offset; i < end; i++)
            this.put(src[i]);
        return this;
    }
    
    public final ByteBuffer put(byte[] src) {
        return put(src, 0, src.length);
    }
```

Buffer具有一系列便捷的存取和批量存取的API，get和put可以是相对的或者是绝对的。在前面的程序列表中，相对方案是不带有索引参数的函数。当相对函数被调用时，位置在返回时前进一。如果位置前进过多，相对运算就会抛出异常。对于put()，如果运算会导致位置超出上界，就会抛出BufferOverflowException异常。对于get()，如果位置不小于上界，就会抛出BufferUnderflowException异常。绝对存取不会影响缓冲区的位置属性，但是如果您所提供的索引超出范围(负数或不小于上界)，也将抛出IndexOutOfBoundsException异常。

### 翻转

```java
    public final Buffer flip() {
        limit = position;
        position = 0;
        mark = -1;
        return this;
    }
    
    public final Buffer rewind() {
        position = 0;
        mark = -1;
        return this;
    }
```

flip()函数将上界属性设置为当前位置，然后将位置重置为0，将一个能够继续添加数据元素的填充状态的缓冲区翻转成一个准备读出元素的释放状态，然后就可以去get()获取数据了。

rewind()函数与flip()相似，但不影响上界属性。它只是将位置值设回0。您可以使用rewind()后退，重读已经被翻转的缓冲区中的数据。

### 释放

```java
    public final int remaining() {
        return limit - position;
    }

    public final boolean hasRemaining() {
        return position < limit;
    }
```

hasRemaining()会在释放缓冲区时告诉您是否已经达到缓冲区的上界。

remaining()函数将告知您从当前位置到上界还剩余的元素数目。

```java
    while (buffer.hasRemaining()) {	     System.out.print(buffer.get()); 
    }
    
    int count = buffer.remaining();    for (int i = 0; i < count, i++) {
		myByteArray[i] = buffer.get();
    }
    
```

### 压缩

```java
    public ByteBuffer compact() {
        System.arraycopy(hb, ix(position()), hb, ix(0), remaining());
        position(remaining());
        limit(capacity());
        discardMark();
        return this;
    }
```

只想从缓冲区中释放一部分数据，而不是全部，然后重新填充。为了实现这一点，未读的数据元素需要下移以使第一个元素索引为0。

### 标记

```java
    public final Buffer mark() {
        mark = position;
        return this;
    }

    public final Buffer reset() {
        int m = mark;
        if (m < 0)
            throw new InvalidMarkException();
        position = m;
        return this;
    }
    
    private static boolean equals(byte x, byte y) {
        return x == y;
    }
```

mark()函数调用时标记被设为当前位置的值。

reset()函数将位置设为当前的标记值。

### 比较

```java
    public boolean equals(Object ob) {
        if (this == ob)
            return true;
        if (!(ob instanceof ByteBuffer))
            return false;
        ByteBuffer that = (ByteBuffer)ob;
        if (this.remaining() != that.remaining())
            return false;
        int p = this.position();
        for (int i = this.limit() - 1, j = that.limit() - 1; i >= p; i--, j--)
            if (!equals(this.get(i), that.get(j)))
                return false;
        return true;
    }
```

如果每个缓冲区中剩余的内容相同，那么equals()函数将返回true，否则返回false。
两个缓冲区被认为相等的充要条件是:1. 两个对象类型相同。包含不同数据类型的buffer永远不会相等，而且buffer绝不会等于非 buffer对象。
2. 两个对象都剩余同样数量的元素。Buffer的容量不需要相同，而且缓冲区中剩余数据的索引也不必相同。但每个缓冲区中剩余元素的数目(从位置到上界)必须相同。
3. 在每个缓冲区中应被get()函数返回的剩余数据元素序列必须一致。如果不满足以上任意条件，就会返回 false。


```java
    public int compareTo(ByteBuffer that) {
        int n = this.position() + Math.min(this.remaining(), that.remaining());
        for (int i = this.position(), j = that.position(); i < n; i++, j++) {
            int cmp = compare(this.get(i), that.get(j));
            if (cmp != 0)
                return cmp;
        }
        return this.remaining() - that.remaining();
    }
    
    private static int compare(byte x, byte y) {
        return Byte.compare(x, y);
    }
```

缓冲区也支持用compareTo()函数以词典顺序进行比较，这一函数在缓冲区参数小于，等于，或者大于引用compareTo()的对象实例时，分别返回一个负整数，0和正整数。这些就是所有典型的缓冲区所实现的java.lang.Comparable接口语义。这意味着缓冲区数组可以通过调用java.util.Arrays.sort()函数按照它们的内容进行排序。compareTo()不允许不同对象间进行比较。比较是针对每个缓冲区内剩余数据进行的，与它们在equals()中的方式相同，直到不相等的元素被发现或者到达缓冲区的上界。如果一个缓冲区在不相等元素发现前已经被耗尽，较短的缓冲区被认为是小于较长的缓冲区。


### 清除

```java
    public final Buffer clear() {
        position = 0;
        limit = capacity;
        mark = -1;
        return this;
    }
```

clear()函数将位置设置为0，上界设置为容量。

### 复制

```java
	public abstract CharBuffer duplicate(); 
	public abstract CharBuffer asReadOnlyBuffer(); 
	public abstract CharBuffer slice();
```

```java
    public CharBuffer duplicate() {
        return new HeapCharBuffer(hb,
                                        this.markValue(),
                                        this.position(),
                                        this.limit(),
                                        this.capacity(),
                                        offset);
    }
    
    public CharBuffer asReadOnlyBuffer() {

        return new HeapCharBufferR(hb,
                                     this.markValue(),
                                     this.position(),
                                     this.limit(),
                                     this.capacity(),
                                     offset);
    }
    
    public CharBuffer slice() {
        return new HeapCharBuffer(hb,
                                        -1,
                                        0,
                                        this.remaining(),
                                        this.remaining(),
                                        this.position() + offset);
    }

```

duplicate()函数创建了一个与原始缓冲区相似的新缓冲区。两个缓冲区共享数据元素，拥有同样的容量，但每个缓冲区拥有各自的位置，上界和标记属性。对一个缓冲区内的数据元素所做的改变会反映在另外一个缓冲区上。这一副本缓冲区具有与原始缓冲区同样的数据视图。如果原始的缓冲区为只读，或者为直接缓冲区，新的缓冲区将继承这些属性。

asReadOnlyBuffer()函数来生成一个只读的缓冲区视图。这与duplicate()相同，除了这个新的缓冲区不允许使用put()，并且其isReadOnly()函数将会返回true。对这一只读缓冲区的 put()函数的调用尝试会导致抛出ReadOnlyBufferException异常。

slice()分割缓冲区与复制相似，但slice()创建一个从原始缓冲区的当前位置开始的新缓冲 区，并且其容量是原始缓冲区的剩余元素数量(limit-position)。这个新缓冲区与原始缓冲区共享一段数据元素子序列。分割出来的缓冲区也会继承只读和直接属性。

## 字节顺序

每个基本数据类型都是以连续字节序列的形式存储在内存中。多字节数值被存储在内存中的方式一般被称为endian-ness(字节顺序)。如果数字数值的最高字节——big end(大端)，位于低位地址，那么系统就是大端字节顺序。如果最低字节最先保存在内存中，那么是小端字节顺序。

```java
public final class ByteOrder {

    private String name;

    private ByteOrder(String name) {
        this.name = name;
    }

    public static final ByteOrder BIG_ENDIAN
        = new ByteOrder("BIG_ENDIAN");

    public static final ByteOrder LITTLE_ENDIAN
        = new ByteOrder("LITTLE_ENDIAN");

    public static ByteOrder nativeOrder() {
        return Bits.byteOrder();
    }

    public String toString() {
        return name;
    }

}
```

ByteOrder类定义了决定从缓冲区中存储或检索多字节数值时使用哪一字节顺序的常量。获取JVM运行的硬件平台的固有字节顺序，可以调用静态类函数nativeOrder()。

## 视图缓冲区

ByteBuffer类提供了丰富的API来创建视图缓冲区。视图缓冲区通过已存在的缓冲区对象实例的工厂方法来创建。这种视图对象维护它自己的属性，容量，位置，上界和标记，但是和原来的缓冲区共享数据元素。

```java
	public abstract class ByteBuffer
            extends Buffer implements Comparable {

        public abstract CharBuffer asCharBuffer();

        public abstract ShortBuffer asShortBuffer();

        public abstract IntBuffer asIntBuffer();

        public abstract LongBuffer asLongBuffer();

        public abstract FloatBuffer asFloatBuffer();

        public abstract DoubleBuffer asDoubleBuffer();
    }
```

每一个工厂方法都在原有的ByteBuffer对象上创建一个视图缓冲区。调用其中的任何一个方法都会创建对应的缓冲区类型，这个缓冲区是基础缓冲区的一个切分，由基础缓冲区的位置和上界决定。新的缓冲区的容量是字节缓冲区中存在的元素数量除以视图类型中组成一个数据类型的字节数。在切分中任一个超过上界的元素对于这个视图缓冲区都是不可见的。视图缓冲区的第一个元素从创建它的ByteBuffer对象的位置开始 (positon()函数的返回值)。具有能被自然数整除的数据元素个数的视图缓冲区是一种较好的实现。

## 数据元素视图

ByteBuffer类提供了一个不太重要的机制来以多字节数据类型的形式存取byte数组。ByteBuffer类为每一种原始数据类型提供了存取的和转化的方法:

```java
    public abstract class ByteBuffer
            extends Buffer implements Comparable {
        public abstract char getChar();

        public abstract char getChar(int index);

        public abstract short getShort();

        public abstract short getShort(int index);

        public abstract int getInt();

        public abstract int getInt(int index);

        public abstract long getLong();

        public abstract long getLong(int index);

        public abstract float getFloat();

        public abstract float getFloat(int index);

        public abstract double getDouble();

        public abstract double getDouble(int index);

        public abstract ByteBuffer putChar(char value);

        public abstract ByteBuffer putChar(int index, char value);

        public abstract ByteBuffer putShort(short value);

        public abstract ByteBuffer putShort(int index, short value);

        public abstract ByteBuffer putInt(int value);

        public abstract ByteBuffer putInt(int index, int value);

        public abstract ByteBuffer putLong(long value);

        public abstract ByteBuffer putLong(int index, long value);

        public abstract ByteBuffer putFloat(float value);

        public abstract ByteBuffer putFloat(int index, float value);

        public abstract ByteBuffer putDouble(double value);

        public abstract ByteBuffer putDouble(int index, double value);
    }
```

些函数从当前位置开始存取ByteBuffer的字节数据，就好像一个数据元素被存储在那里一样。根据这个缓冲区的当前的有效的字节顺序，这些字节数据会被排列或打乱成需要的原始数据类型。比如说，如果getInt()函数被调用，从当前的位置开始的四个字节会被包装成一个int类型的变量然后作为函数的返回值返回。
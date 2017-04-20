---
title: Java NIO 文件IO-内存映射文件MappedByteBuffer与zerocopy技术
date: 2017-01-31 23:30:00
categories: IO&NIO
tags: [Java, NIO]
---


在传统的文件IO操作中，我们都是调用操作系统提供的底层标准IO系统调用函数read()、write() ，此时调用此函数的进程（在JAVA中即java进程）由当前的用户态切换到内核态，然后OS的内核代码负责将相应的文件数据读取到内核的IO缓冲区，然后再把数据从内核IO缓冲区拷贝到进程的私有地址空间中去，这样便完成了一次IO操作。这么做是为了减少磁盘的IO操作，为了提高性能而考虑的，因为我们的程序访问一般都带有局部性，也就是所谓的局部性原理，在这里主要是指的空间局部性，即我们访问了文件的某一段数据，那么接下去很可能还会访问接下去的一段数据，由于磁盘IO操作的速度比直接访问内存慢了好几个数量级，所以OS根据局部性原理会在一次read()系统调用过程中预读更多的文件数据缓存在内核IO缓冲区中，当继续访问的文件数据在缓冲区中时便直接拷贝数据到进程私有空间，避免了再次的低效率磁盘IO操作。

<!--more-->

内存映射文件是将硬盘上文件的位置与进程逻辑地址空间中一块大小相同的区域之间一一对应， 建立内存映射由mmap()系统调用将文件直接映射到用户空间，mmap()中没有进行数据拷贝，真正的数据拷贝是在缺页中断处理时进行的，mmap()会返回一个指针ptr，它指向进程逻辑地址空间中的一个地址，要操作其中的数据时即第一次访问ptr指向的内存区域，必须通过MMU将逻辑地址转换成物理地址，MMU在地址映射表中是无法找到与ptr相对应的物理地址的，也就是MMU失败，将产生一个缺页中断，缺页中断的中断响应函数会通过mmap()建立的映射关系，从硬盘上将文件读取到物理内存中，这个过程只进行了一次数据拷贝。因此，内存映射的效率要比read/write调用效率高。



## 内存映射文件 MappedByteBuffer

JAVA处理大文件，一般用BufferedReader，BufferedInputStream这类带缓冲的IO类，不过如果文件超大的话，更快的方式是采用MappedByteBuffer。

```java
MappedByteBuffer buffer = fileChannel.map(FileChannel.MapMode.READ_ONLY, 0, len);
```

FileChannel提供了map方法来把文件映射为内存映像文件：可以把文件的从position开始的size大小的区域映射为内存映像文件，MapMode表示了可访问该内存映像文件的方式：

- READ_ONLY,（只读）： 试图修改得到的缓冲区将导致抛出ReadOnlyBufferException。(MapMode.READ_ONLY)

- READ_WRITE（读/写）： 对得到的缓冲区的更改最终将传播到文件；该更改对映射到同一文件的其他程序不一定是可见的。(MapMode.READ_WRITE)

- PRIVATE（专用）： 对得到的缓冲区的更改不会传播到文件，并且该更改对映射到同一文件的其他程序也不是可见的；相反，会创建缓冲区已修改部分的专用副本。(MapMode.PRIVATE)


MappedByteBuffer是ByteBuffer的子类，其扩充了三个方法：

- force()：缓冲区是READ_WRITE模式下，此方法对缓冲区内容的修改强行写入文件；

- load()：将缓冲区的内容载入内存，并返回该缓冲区的引用；

- isLoaded()：如果缓冲区的内容在物理内存中，则返回真，否则返回假；


通过下面的例子测试普通文件通道IO和内存映射IO的速度：

```java
public class FileChannelTest {

    public static void main(String[] args) throws IOException {
        testFileChannel();
        testMappedByteBuffer();
    }

    public static void testFileChannel() throws IOException {
        RandomAccessFile file = null;
        try {
            file = new RandomAccessFile("/Users/xin/Downloads/b.txt", "rw");
            FileChannel channel = file.getChannel();
            ByteBuffer buff = ByteBuffer.allocate(1024);
            long timeBegin = System.currentTimeMillis();
            while (channel.read(buff) != -1) {
                buff.flip();
                buff.clear();
            }
            long timeEnd = System.currentTimeMillis();
            System.out.println("Read time: " + (timeEnd - timeBegin) + "ms");
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (file != null) {
                    file.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

    }

    public static void testMappedByteBuffer() throws IOException {
        RandomAccessFile file = null;
        try {
            file = new RandomAccessFile("/Users/xin/Downloads/b.txt", "rw");
            FileChannel fc = file.getChannel();
            int len = (int) file.length();
            MappedByteBuffer buffer = fc.map(FileChannel.MapMode.READ_ONLY, 0, len);
            byte[] b = new byte[1024];

            long timeBegin = System.currentTimeMillis();
            for (int offset = 0; offset < len; offset += 1024) {

                if (len - offset > 1024) {
                    buffer.get(b);
                } else {
                    buffer.get(new byte[len - offset]);
                }
            }
            long timeEnd = System.currentTimeMillis();

            System.out.println("Read time: " + (timeEnd - timeBegin) + "ms");
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (file != null) {
                    file.close();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```

控制台输出结果如下：

```
Read time: 302ms
Read time: 61ms
```

根据测试结果证明了内存映射文件比文件通道速度快很多。

内存映射文件和之前说的标准IO操作最大的不同之处就在于它虽然最终也是要从磁盘读取数据，但是它并不需要将数据读取到OS内核缓冲区，而是直接将进程的用户私有地址空间中的一部分区域与文件对象建立起映射关系，就好像直接从内存中读、写文件一样，速度当然快了。内存映射文件的效率比标准IO高的重要原因就是因为少了把数据拷贝到OS内核缓冲区这一步。


## zerocopy技术

zerocopy技术的目标就是提高IO密集型JAVA应用程序的性能。IO操作需要数据频繁地在内核缓冲区和用户缓冲区之间拷贝，而zerocopy技术可以减少这种拷贝的次数，同时也降低了上下文切换(用户态与内核态之间的切换)的次数。在Java中的应用就是java.nio.channels.FileChannel类的transferTo()方法可以直接将字节传送到可写的通道中，并不需要将字节转入用户缓冲区。

```java
package java.nio.channels;

public abstract class FileChannel
    extends AbstractInterruptibleChannel
    implements SeekableByteChannel, GatheringByteChannel, ScatteringByteChannel
{
    ... ...
    
    public abstract long transferTo(long position, long count, WritableByteChannel target) throws IOException;
        
    ... ...
}
```

```java
FileChannel fileChannel = fis.getChannel();
fileChannel.transferTo(0, fileChannel.size(), targetChannel);
```

正常读取文件再发送出去需要经历一下几个步骤：

```
从本地磁盘或者网络读取数据--->数据进入内核缓冲区--->用户缓冲区--->内核缓冲区--->通过socket发送

```

数据每次在内核缓冲区与用户缓冲区之间的拷贝会消耗CPU以及内存的带宽。而zerocopy有效减少了这种拷贝次数，用户程序执行transferTo()方法，导致一次系统调用，从用户态切换到内核态，完成的动作是：

```
从本地磁盘或者网络读取数据--->数据进入内核缓冲区--->通过socket发送

```

zerocopy好处就是减少了将数据从内核缓冲区拷贝到用户缓冲区，再拷贝回内核缓冲区，减少了拷贝次数和上下文切换次数。


## 参考资料

[JAVA NIO之浅谈内存映射文件原理与DirectMemory](http://blog.csdn.net/fcbayernmunchen/article/details/8635427)

[JAVA IO 以及 NIO 理解 zerocopy技术介绍](http://www.cnblogs.com/hapjin/p/5736188.html)


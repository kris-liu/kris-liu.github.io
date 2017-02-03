---
title: Java NIO Selector详解
date: 2017-01-24 00:33:00
categories: IO&NIO
tags: [Java, NIO]
---
## Selector选择器

Selector（选择器）是Java NIO中能够检测一到多个NIO通道，并能够发现通道是否为读写等事件做好准备的组件。这样，一个单独的线程可以管理多个channel，从而管理多个网络连接。

Selector的实现根据JVM运行的操作系统不同会有相应的不同的实现，上层API对底层做了抽象，这样上层API无需关心底层操作系统的变化，可以在不同操作系统上实现相同的功能。

实现了SelectableChannel接口的通道可以被注册到Selector上，配合Selector工作，实现IO多路复用。

## SelectionKey选择键 
选择键封装了某一个的通道与某一个的选择器的注册关系。通道和选择器往往是配合工作，选择键对象被SelectableChannel.register()返回并提供一个表示这种注册关系的键。选择键包含了两个比特集(以整数的形式进行编码)，指示了该注册关系所关心的通道操作，以及通道已经准备好的操作。

```java
public abstract class SelectionKey {

    protected SelectionKey() { }

    public abstract SelectableChannel channel();

    public abstract Selector selector();

    public abstract boolean isValid();

    public abstract void cancel();

    public abstract int interestOps();

    public abstract SelectionKey interestOps(int ops);

    public abstract int readyOps();

	 //关心的操作的比特掩码
    public static final int OP_READ = 1 << 0;

    public static final int OP_WRITE = 1 << 2;

    public static final int OP_CONNECT = 1 << 3;

    public static final int OP_ACCEPT = 1 << 4;

    public final boolean isReadable() {
        return (readyOps() & OP_READ) != 0;
    }

    public final boolean isWritable() {
        return (readyOps() & OP_WRITE) != 0;
    }

    public final boolean isConnectable() {
        return (readyOps() & OP_CONNECT) != 0;
    }

    public final boolean isAcceptable() {
        return (readyOps() & OP_ACCEPT) != 0;
    }

    private volatile Object attachment = null;

    private static final AtomicReferenceFieldUpdater<SelectionKey,Object>
        attachmentUpdater = AtomicReferenceFieldUpdater.newUpdater(
            SelectionKey.class, Object.class, "attachment"
        );

    public final Object attach(Object ob) {
        return attachmentUpdater.getAndSet(this, ob);
    }

    public final Object attachment() {
        return attachment;
    }

}
```


一个键表示了一个特定的通道对象和一个特定的选择器对象之间的注册关系。

channel()方法返回与该键相关的SelectableChannel对象。

selector()则返回相关的Selector对象。
调用SelectionKey对象的cancel()方法可以取消通道和选择器的关联。可以通过调用isValid()方法来检查它是否仍然是有效的关联关系。当键被取消时，它将被放在相关的选择器的已取消的键的集合里。注册不会立即被取消，但键会立即失效。当再次调用select()方法时(或者一个正在进行的select()调用结束时)，已取消的键的集合中的被取消的键将被清理掉，并且相应的注销也将完成。当通道关闭时，所有相关的键会自动取消。当选择器关闭时，所有被注册到该选择器的通道都将被注销，并且相关的键将立即被无效化。一旦键被无效化，调用它的与选择相关的方法就将抛出CancelledKeyException。


```java

    public static final int OP_READ = 1 << 0;//1

    public static final int OP_WRITE = 1 << 2;//4

    public static final int OP_CONNECT = 1 << 3;//8

    public static final int OP_ACCEPT = 1 << 4;//16

```


上面的常量表示通道相关的操作的比特掩码。

一个SelectionKey对象包含两个以整数形式进行编码的比特掩码：一个用于指示那些通道选择器组合体所关心的操作(instrest集合)，另一个表示通道准备好要执行的操作(ready集合)。

- instrest集合：当前的interest集合可以通过调用键对象的interestOps()方法来获取。可以通过调用interestOps()方法并传入一个新的比特掩码参数来改变它。当相关的Selector上的select()操作正在进行时改变键的interest集合，不会影响那个正在进行的选择操作。所有更改将会在select()的下一个调用中体现出来。- ready集合：可以通过调用键的readyOps()方法来获取相关的通道的已经就绪的操作，不能直接改变键的ready集合。ready集合是interest集合的子集，并且表示了interest集合中从上次调用select()以来已经就绪的那些操作。


SelectionKey类定义了四个便于使用的布尔方法来为您测试这些比特值，用来检测channel中什么事件或操作已经就绪：

- isReadable()
- isWritable()
- isConnectable()
- isAcceptable()


attach()方法将在键对象中保存所提供的对象的引用。SelectionKey类除了保存它之外，不 会将它用于任何其他用途。任何一个之前保存在键中的附件引用都会被替换。可以使用null值来清除附件。可以通过调用attachment()方法来获取与键关联的附件句柄。


## Selector API

### 创建

```java
Selector selector = Selector.open();
```

```java    
public abstract class Selector implements Closeable {

    public static Selector open() throws IOException {
        return SelectorProvider.provider().openSelector();
    }
        
}
```

通过Selector的open()方法可以获取一个选择器实例，底层通过SelectorProvider创建一个相应的通道实例。SelectorProvider实例根据JVM运行的操作系统不同会有相应的不同的实现，上层API无需关心底层操作系统的变化。

### 注册

```java
	channel.configureBlocking(false);
	SelectionKey key = channel.register(selector, Selectionkey.OP_READ);
```

```java
    public final SelectionKey register(Selector sel, int ops,
                                       Object att)
        throws ClosedChannelException
    {
        synchronized (regLock) {
            if (!isOpen())
                throw new ClosedChannelException();
            if ((ops & ~validOps()) != 0) //validOps是通道支持的所有操作的比特码
                throw new IllegalArgumentException();
            if (blocking)
                throw new IllegalBlockingModeException();
            SelectionKey k = findKey(sel);//查看通道是否已经注册到该选择器
            if (k != null) {//已经注册上则只更新其关心的IO操作和附件attach
                k.interestOps(ops);
                k.attach(att);
            }
            if (k == null) {
                // New registration
                synchronized (keyLock) {
                    if (!isOpen())
                        throw new ClosedChannelException();
                    k = ((AbstractSelector)sel).register(this, ops, att);//最终要掉用Selector的register方法将自己注册到选择器上，并将SelectionKey添加到Selector维护的集合中
                    addKey(k);//添加到当前通道维护的SelectionKey集合
                }
            }
            return k;
        }
    }
```

与Selector一起使用时，Channel必须处于非阻塞模式下。 

register()方法定义在SelectableChannel接口上，接受一个Selector对象作为参数，以及一个名为ops的整数参数。第二个参数表示所关心的通道操作。这是一个表示选择器在检查通道就绪状态时需要关心的操作的比特掩码。特定的操作比特值在SelectonKey类中被定义为public static字段。

register()方法首先检测通道是open的且要关心的操作是当前通道支持的，且是非阻塞状态的通道。通过校验之后，会在通道所维护的键集合中查看是否已经有当前通道和当前选择器相关联SelectionKey，有意味着已经注册过了，则修改关心的操作和附件；没有则需要调用Selector的注册方法，将当前通道注册到选择器上，注册操作会将该SelectionKey添加Selector维护的已注册的集合中，并添加到当前通道维护的SelectionKey集合中。

### 选择器内部三个集合

```java
public abstract class Selector implements Closeable {

    public abstract Set<SelectionKey> keys();

    public abstract Set<SelectionKey> selectedKeys();
    
}
```

选择器维护着已注册的键的集合，已选择的键的集合，已取消的键的集合：

- 已注册的键的集合	keys()方法返回与选择器关联的已经注册的键的集合。并不是所有注册过的键都仍然有效。这个集合通过keys()方法返回，并且可能是空的。这个已注册的键的集合是不可以直接修改的，只能通过注册和注销选择器的行为添加或者移除。
	- 已选择的键的集合
	selectedKeys()方法返回已选择的键的集合。他是已注册的键的集合的子集。这个集合的每个成员都是相关的通道被选择器上一次选择过程中被识别为已经准备好的IO操作的，并且是包含于键的interest集合中的IO操作。这个集合通过selectedKeys()方法返回，并有可能是空的。集合中每个键都关联一个已经准备好至少一种操作的通道。每个键都有一个内嵌的ready集合，指示了所关联的通道已经准备好何种操作。键可以直接从这个集合中移除，但不能添加。
- 已取消的键的集合
	已注册的键的集合的子集，这个集合包含了键的cancel()方法被调用过的键，但它们还没有被注销，它们会在每次选择操作前后完成注销。这个集合是选择器对象的私有成员，因而无法直接访问。
在一个刚初始化的 Selector 对象中，这三个集合都是空的。


### 选择

```java
    public int select(long var1) throws IOException;

    public int select() throws IOException;
    
    public int selectNow() throws IOException;
```

select方法返回的int值表示有多少通道已经就绪。有可能是0。这三种select的形式，仅在阻塞和超时设置上有所不同。

- select()阻塞到至少有一个通道在你注册的事件上就绪了。

- select(long timeout)和select()一样，但是最长会阻塞timeout毫秒(参数)。

- selectNow()不会阻塞，不管什么通道就绪都立刻返回。

选择方法是选择器的核心，选择方法是对select()、poll()、epoll()等本地调用或者类似的操作系统特定的系统调用的一个包装，依赖底层操作系统的支持。

当三种形式的select()中的任意一种被调用时，下面步骤将被执行:
1. 已取消的键的集合将会被检查。如果它是非空的，每个已取消的键的集合中的键将从另外两 个集合中移除，并且相关的通道将被注销。这个步骤结束后，已取消的键的集合将是空的。

2. 执行底层操作系统相关的select，poll或者epoll之类的底层操作系统调用，底层操作系统将会进行检查，以确定每个通道所关心的操作的真实就绪状态。依赖于特定的select()方法调用，如果没有通道已经准备好，线程可能会在这时阻塞，通常会有一个超时值。

3. 步骤2可能会花费很长时间，特别是线程处于休眠状态时。与该选择器相关的键可能会同时被取消。当步骤2结束时，步骤1将重新执行，以完成任意一个在选择进行的过程中，键已经被取消的通道的注销。

4. 当前每个通道的就绪状态将确定下来。对于那些还没准备好的通道将不会执行任何的操作。对于那些操作系统指示至少已经准备好interest集合中的一种操作的通道，将执行以下两种操作中的一种:	- a. 如果通道的键还没有处于已选择的键的集合中，那么键的ready集合将先被清空，然后表示操作系统发现的当前通道已经准备好的操作的比特掩码将被设置。	- b.否则，也就是键在已选择的键的集合中。键的ready集合将被表示操作系统发现的当前已经准备好的操作的比特掩码更新。所有之前的已经不再是就绪状态的操作不会被清除。由操作系统决定的ready集合是与之前的ready集合按位分离的，一旦键被放置于选择器的已选择的键的集合中，它的ready集合将是累积的。比特位只会被添加，不会被清理，所以一般在select之后操作时会将已选择的键从已选择的键的集合中移除。
	select操作返回的值是在步骤4中被修改的键的数量，返回值不是已准备好的通道的总数，而是从上一个select()调用之后进入就绪状态的通道的数量。之前的调用中就绪的，并且在本次调用中仍然就绪的通道不会被计入，而那些在前一次调用中已经就绪但已经不再处于就绪状态的通道也不会被计入。这些通道可能仍然在已选择的键的集合中，但不会被计入返回值中。
使用内部的已取消的键的集合来延迟注销，是一种防止线程在取消键时阻塞，并防止与正在进行的选择操作冲突的优化。注销通道是一个潜在的代价很高的操作，这可能需要重新分配资源。清理已取消的键，并在选择操作之前和之后立即注销通道，可以消除它们可能正好在选择的过程中执行的潜在棘手问题。这是另一个兼顾健壮性的折中方案。


### 停止选择

```java
public abstract class Selector implements Closeable {

    //方法1
    public abstract void wakeup()

    //方法2
    public abstract void close() throws IOException;

}
```

```java
public class Thread implements Runnable {

    //方法3
    public void interrupt() {
        if (this != Thread.currentThread())
            checkAccess();

        synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {
                interrupt0();           // Just to set the interrupt flag
                b.interrupt(this);
                return;
            }
        }
        interrupt0();
    }
    
}
```

使线程从被阻塞的select()方法中退出的三种方法：

1. wakeup()
	调用Selector对象的wakeup()方法将使得选择器上的第一个还没有返回的选择操作立即返回。如果当前没有在进行中的选择，那么下一次对select()方法的一种形式的调用将立即返回。在选择操作之间多次调用wakeup()方法与调用它一次没有什么不同。wakeup()提供了使线程从被阻塞的select()方法中优雅地退出的能力。
	
2. close()
	调用Selector对象的close()方法，那么任何一个在选择操作中阻塞的线程都将被唤醒，因为内部会调用wakeup()方法。与选择器相关的通道将被注销，而键将被取消。
	
3. 	interrupt()

	调用选择过程中的线程的interrupt()方法，Selector对象将捕捉InterruptedException异常并调用wakeup()方法。如果被唤醒的线程之后试图在通道上执行I/O操作，通道将立即关闭，需要清理中断状态。


## 代码示例

通过一个服务端应用的代码示例展示选择器如何与通道和缓冲区共同使用，具体选择器和选择键还有通道如何相互配合以及底层如何调用还是自己看一下关键方法的源代码了解一下吧。^_^


```java
public class SocketServer3 {

    private static final Logger LOGGER = LoggerFactory.getLogger(SocketServer3.class);


    private static final int PORT_NUMBER = 10003;
    private static ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

    public static void main(String[] args) throws Exception {
        int port = PORT_NUMBER;
        LOGGER.info("Listening on port " + port);

        Selector selector = Selector.open();

        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.bind(new InetSocketAddress(port));
        serverChannel.configureBlocking(false);
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);

        while (true) {

            //查看是否有注册的IO事件发生
            int n = selector.select();
            if (n == 0) {
                continue; // nothing to do
            }

            //获取已准备好IO事件的通道集合
            Iterator it = selector.selectedKeys().iterator();
            while (it.hasNext()) {
                SelectionKey key = (SelectionKey) it.next();

                //通道有accept事件发生
                if (key.isValid() && key.isAcceptable()) {
                    ServerSocketChannel server = (ServerSocketChannel) key.channel();
                    SocketChannel channel = server.accept();
                    registerChannel(selector, channel, SelectionKey.OP_READ);
                }

                //通道有read事件发生
                if (key.isValid() && key.isReadable()) {
                    readDataFromSocket(key);
                }

                //通道有write事件发生
                if (key.isValid() && key.isWritable()) {
                    LOGGER.info("isWritable = true");
                }

                //通道有connect事件发生
                if (key.isValid() && key.isConnectable()) {
                    LOGGER.info("isConnectable = true");
                }

                //移除已处理IO事件的通道
                it.remove();
            }
        }
    }

    /**
     * 处理监听IO操作，将接收到通道注册到Selector上
     * @param selector
     * @param channel
     * @param ops
     * @throws Exception
     */
    private static void registerChannel(Selector selector, SelectableChannel channel, int ops) throws Exception {
        if (channel == null) {
            return;
        }
        channel.configureBlocking(false);
        channel.register(selector, ops);
    }

    /**
     * 处理读取IO操作，将服务端接收到的数据发送回客户端
     * @param key
     * @throws Exception
     */
    private static void readDataFromSocket(SelectionKey key) throws Exception {
        SocketChannel socketChannel = (SocketChannel) key.channel();
        int count;
        buffer.clear();

        while ((count = socketChannel.read(buffer)) > 0) {
            buffer.flip();
            while (buffer.hasRemaining()) {
                socketChannel.write(buffer);
            }
            buffer.clear();
            key.interestOps(SelectionKey.OP_READ | SelectionKey.OP_WRITE);
        }

        if (count < 0) {
            // Close channel on EOF, invalidates the key
            socketChannel.close();
        }
    }
}
```

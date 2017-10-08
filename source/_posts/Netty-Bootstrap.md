---
title: Netty源码分析 Bootstrap客户端启动
date: 2017-04-10 20:00:00
categories: Netty
tags: [Netty, 源码]
---

> 本文使用`netty-4.1.5.Final`版本源码进行分析

Bootstrap是Socket客户端创建工具类，用户通过Bootstrap可以方便的创建Netty的客户端并发起异步TCP连接操作。下面我们看一下客户端Bootstrap如何启动并对启动过程的源码进行分析。

<!--more-->
## Bootstrap客户端启动

```java
        // Configure the client.
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap();
            b.group(group)
             .channel(NioSocketChannel.class)
             .option(ChannelOption.TCP_NODELAY, true)
             .handler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     p.addLast(new EchoClientHandler());
                 }
             });

            // Start the client.
            ChannelFuture f = b.connect(HOST, PORT).sync();

            // Wait until the connection is closed.
            f.channel().closeFuture().sync();
        } finally {
            // Shut down the event loop to terminate all threads.
            group.shutdownGracefully();
        }
```

创建Bootstrap实例，绑定用于处理I/O事件及task任务的workerGroup，并设置其他相关定制参数，最后对指定的host和port发起建立连接请求，就完成了启动Netty客户端的代码。


## Bootstrap服务端启动源码分析

### 创建Bootstrap实例

```java
	Bootstrap b = new Bootstrap();
```

### 设置Bootstrap参数

绑定Reactor线程池

```java
    public B group(EventLoopGroup group) {
        if (group == null) {
            throw new NullPointerException("group");
        }
        if (this.group != null) {
            throw new IllegalStateException("group set already");
        }
        this.group = group;
        return (B) this;
    }
```

group线程组主要用于I/O读写以及task执行


设置channel类型，用于根据class类型反射创建对应channel

```java
    public B channel(Class<? extends C> channelClass) {
        if (channelClass == null) {
            throw new NullPointerException("channelClass");
        }
        return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
    }
    
    public B channelFactory(ChannelFactory<? extends C> channelFactory) {
        if (channelFactory == null) {
            throw new NullPointerException("channelFactory");
        }
        if (this.channelFactory != null) {
            throw new IllegalStateException("channelFactory set already");
        }

        this.channelFactory = channelFactory;
        return (B) this;
    }
    
public class ReflectiveChannelFactory<T extends Channel> implements ChannelFactory<T> {

    private final Class<? extends T> clazz;

    public ReflectiveChannelFactory(Class<? extends T> clazz) {
        if (clazz == null) {
            throw new NullPointerException("clazz");
        }
        this.clazz = clazz;
    }

    @Override
    public T newChannel() {//通过反射创建channel
        try {
            return clazz.newInstance();
        } catch (Throwable t) {
            throw new ChannelException("Unable to create Channel from class " + clazz, t);
        }
    }

    @Override
    public String toString() {
        return StringUtil.simpleClassName(clazz) + ".class";
    }
}
```

根据channel的类型，创建一个生产channel的工厂，用于通过channel类型反射创建对应的channel，服务端一般使用`NioServerSocketChannel.class`


设置channel参数

```java
    public <T> B option(ChannelOption<T> option, T value) {
        if (option == null) {
            throw new NullPointerException("option");
        }
        if (value == null) {
            synchronized (options) {
                options.remove(option);
            }
        } else {
            synchronized (options) {
                options.put(option, value);
            }
        }
        return (B) this;
    }
```

设置handler

```java
    public B handler(ChannelHandler handler) {
        if (handler == null) {
            throw new NullPointerException("handler");
        }
        this.handler = handler;
        return (B) this;
    }
```
为NioSocketChannel设置ChannelHandler，ChannelHandler是netty提供给用户定制和扩展的关键接口，用户可以通过自定义ChannelHandler，添加具体的业务逻辑处理。


### 启动Bootstrap的核心逻辑源码分析

启动客户端Bootstrap，需要对指定的host和port发起建立连接请求，首先需要将Channel初始化并注册到eventLoop中。

Bootstrap启动的核心逻辑总共分三步操作：

1. 初始化
2. 注册
3. 发起连接

首先看下启动过程的主干逻辑，然后再具体分析每一步操作的具体逻辑：

```java
    public ChannelFuture connect(SocketAddress remoteAddress) {
        if (remoteAddress == null) {
            throw new NullPointerException("remoteAddress");
        }
        validate();//参数基本校验
        return doResolveAndConnect(remoteAddress, config.localAddress());//发起连接并返回连接结果
    }
```

```java
    private ChannelFuture doResolveAndConnect(final SocketAddress remoteAddress, final SocketAddress localAddress) {
        final ChannelFuture regFuture = initAndRegister();//初始化NioSocketChannel并注册到group的eventLoop上
        final Channel channel = regFuture.channel();

        if (regFuture.isDone()) {//如果注册到group上已完成则直接进行连接操作
            if (!regFuture.isSuccess()) {
                return regFuture;
            }
            return doResolveAndConnect0(channel, remoteAddress, localAddress, channel.newPromise());//发起连接
        } else {//如果注册到group上还未完成，则添加Listener当执行完注册操作后再回调Listener进行连接操作
            // Registration future is almost always fulfilled already, but just in case it's not.
            final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    // Direclty obtain the cause and do a null check so we only need one volatile read in case of a
                    // failure.
                    Throwable cause = future.cause();
                    if (cause != null) {
                        // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                        // IllegalStateException once we try to access the EventLoop of the Channel.
                        promise.setFailure(cause);
                    } else {
                        // Registration was successful, so set the correct executor to use.
                        // See https://github.com/netty/netty/issues/2586
                        promise.registered();
                        doResolveAndConnect0(channel, remoteAddress, localAddress, promise);//发起连接
                    }
                }
            });
            return promise;//外部可以用过promise得到是否连接完成的结果，或者阻塞直到完成连接。
        }
    }
```

首先初始化NioSocketChannel并注册到group的eventLoop上，当NioSocketChannel已经执行完成注册操作，则直接发起连接操作，否则添加Listener当执行完注册操作后再回调Listener发起连接。

初始化NioSocketChannel并注册到group的eventLoop上

```java
    final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            channel = channelFactory.newChannel();//通过创建Bootstrap时设置的channel创建工厂通过反射创建对应的channel，客户端一般是NioSocketChannel
            init(channel);//初始化channel
        } catch (Throwable t) {
            if (channel != null) {
                // channel can be null if newChannel crashed (eg SocketException("too many open files"))
                channel.unsafe().closeForcibly();
            }
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }

        ChannelFuture regFuture = config().group().register(channel);//注册channel到eventLoop上
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }

        // If we are here and the promise is not failed, it's one of the following cases:
        // 1) If we attempted registration from the event loop, the registration has been completed at this point.
        //    i.e. It's safe to attempt bind() or connect() now because the channel has been registered.
        // 2) If we attempted registration from the other thread, the registration request has been successfully
        //    added to the event loop's task queue for later execution.
        //    i.e. It's safe to attempt bind() or connect() now:
        //         because bind() or connect() will be executed *after* the scheduled registration task is executed
        //         because register(), bind(), and connect() are all bound to the same thread.

        return regFuture;//返回注册操作的结果，用于判断是否已注册完成
    }
```

创建并初始化NioSocketChannel，然后将其注册到group的eventLoop上去，并返回注册操作的结果。

#### 初始化

```java
    void init(Channel channel) throws Exception {
        ChannelPipeline p = channel.pipeline();//获取负责处理网络事件的职责链，用来管理和执行ChannelHandler
        p.addLast(config.handler());////将创建Bootstrap时设置的handler添加到NioSocketChannel的责任链上

        final Map<ChannelOption<?>, Object> options = options0();//设置channel的参数
        synchronized (options) {
            for (Entry<ChannelOption<?>, Object> e: options.entrySet()) {
                try {
                    if (!channel.config().setOption((ChannelOption<Object>) e.getKey(), e.getValue())) {
                        logger.warn("Unknown channel option: " + e);
                    }
                } catch (Throwable t) {
                    logger.warn("Failed to set a channel option: " + channel, t);
                }
            }
        }

        final Map<AttributeKey<?>, Object> attrs = attrs0();//设置channel的附加属性
        synchronized (attrs) {
            for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
                channel.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
            }
        }
    }
```
    
#### 注册

注册NioSocketChannel，从group选择一个eventLoop线程，将NioSocketChannel注册到该eventLoop的selector上，通过channel获取unsafe，进而操作底层NIO的api进行注册操作

```java
    public ChannelFuture register(final ChannelPromise promise) {
        ObjectUtil.checkNotNull(promise, "promise");
        promise.channel().unsafe().register(this, promise);
        return promise;
    }
```

unsafe的register方法，判断当前线程是否是对应channel的eventLoop线程来决定是直接执行register0还是封装一个task交由对应的eventLoop来执行register0

```java
        public final void register(EventLoop eventLoop, final ChannelPromise promise) {
            if (eventLoop == null) {
                throw new NullPointerException("eventLoop");
            }
            if (isRegistered()) {
                promise.setFailure(new IllegalStateException("registered to an event loop already"));
                return;
            }
            if (!isCompatible(eventLoop)) {
                promise.setFailure(
                        new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
                return;
            }

            AbstractChannel.this.eventLoop = eventLoop;

            if (eventLoop.inEventLoop()) {//判断当前线程是否是对应的eventLoop线程来决定是直接执行register0还是封装一个task交由对应的eventLoop来执行
                register0(promise);
            } else {
                try {
                    eventLoop.execute(new Runnable() {//提交task执行register0
                        @Override
                        public void run() {
                            register0(promise);
                        }
                    });
                } catch (Throwable t) {
                    logger.warn(
                            "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                            AbstractChannel.this, t);
                    closeForcibly();
                    closeFuture.setClosed();
                    safeSetFailure(promise, t);
                }
            }
        }
        
        private void register0(ChannelPromise promise) {
            try {
                // check if the channel is still open as it could be closed in the mean time when the register
                // call was outside of the eventLoop
                if (!promise.setUncancellable() || !ensureOpen(promise)) {
                    return;
                }
                boolean firstRegistration = neverRegistered;//是否第一次注册
                doRegister();//调用底层NIO的api执行注册操作
                neverRegistered = false;//标识已经注册过一次
                registered = true;////标识已经注册的状态

                // Ensure we call handlerAdded(...) before we actually notify the promise. This is needed as the
                // user may already fire events through the pipeline in the ChannelFutureListener.
                pipeline.invokeHandlerAddedIfNeeded();//最终会调用ChannelInitializer的handlerAdded方法，进而调用ChannelInitializer的initChannel方法初始化定制的ChannelHandler

                safeSetSuccess(promise);//设置promise结果为成功
                pipeline.fireChannelRegistered();//触发fireChannelRegistered事件，该方法也会调用invokeHandlerAddedIfNeeded()，不过通过状态位保证了invokeHandlerAddedIfNeeded只会执行一次，该方法主要是用于传播注册完成事件
                // Only fire a channelActive if the channel has never been registered. This prevents firing
                // multiple channel actives if the channel is deregistered and re-registered.
                if (isActive()) {//channel状态是否已绑定或已连接
                    if (firstRegistration) {//第一次注册需要触发fireChannelActive事件来设置监听位
                        pipeline.fireChannelActive();
                    } else if (config().isAutoRead()) {//不是第一次注册，但是AutoRead，也需要设置监听位
                        // This channel was registered before and autoRead() is set. This means we need to begin read
                        // again so that we process inbound data.
                        //
                        // See https://github.com/netty/netty/issues/4805
                        beginRead();//设置监听位
                    }
                }
            } catch (Throwable t) {
                // Close the channel directly to avoid FD leak.
                closeForcibly();
                closeFuture.setClosed();
                safeSetFailure(promise, t);
            }
        }
```

调用底层NIO的api执行注册操作

```java
    protected void doRegister() throws Exception {//调用底层NIO的api执行注册操作
        boolean selected = false;
        for (;;) {
            try {
                selectionKey = javaChannel().register(eventLoop().selector, 0, this);//此处没有设置监听位，后续会通过fireChannelActive事件传播到HeadContext中再根据channel类型来设置需要的监听位
                return;
            } catch (CancelledKeyException e) {
                if (!selected) {
                    // Force the Selector to select now as the "canceled" SelectionKey may still be
                    // cached and not removed because no Select.select(..) operation was called yet.
                    eventLoop().selectNow();
                    selected = true;
                } else {
                    // We forced a select operation on the selector before but the SelectionKey is still cached
                    // for whatever reason. JDK bug ?
                    throw e;
                }
            }
        }
    }
```

#### 发起连接

注册完成回到连接方法，当NioSocketChannel已经执行完成注册操作，则直接发起连接操作，否则添加Listener当执行完注册操作后再回调Listener发起连接操作，连接操作需要操作channel，所以需要封装成task任务交由channel对应的eventLoop线程执行其connect方法

```java
    private ChannelFuture doResolveAndConnect0(final Channel channel, SocketAddress remoteAddress,
                                               final SocketAddress localAddress, final ChannelPromise promise) {
        try {
            final EventLoop eventLoop = channel.eventLoop();
            final AddressResolver<SocketAddress> resolver = this.resolver.getResolver(eventLoop);//对目标地址做一些解析校验

            if (!resolver.isSupported(remoteAddress) || resolver.isResolved(remoteAddress)) {//无法解析或者已经解析过则立刻发起连接操作
                // Resolver has no idea about what to do with the specified remote address or it's resolved already.
                doConnect(remoteAddress, localAddress, promise);//对目标地址发起连接
                return promise;
            }

            final Future<SocketAddress> resolveFuture = resolver.resolve(remoteAddress);//解析目标地址

            if (resolveFuture.isDone()) {//解析完成则直接发起连接操作
                final Throwable resolveFailureCause = resolveFuture.cause();

                if (resolveFailureCause != null) {
                    // Failed to resolve immediately
                    channel.close();
                    promise.setFailure(resolveFailureCause);
                } else {
                    // Succeeded to resolve immediately; cached? (or did a blocking lookup)
                    doConnect(resolveFuture.getNow(), localAddress, promise);//对目标地址发起连接
                }
                return promise;
            }

            // Wait until the name resolution is finished.
            resolveFuture.addListener(new FutureListener<SocketAddress>() {//解析未完成则添加Listener当解析完成通过校验后再回调Listener发起连接操作
                @Override
                public void operationComplete(Future<SocketAddress> future) throws Exception {
                    if (future.cause() != null) {
                        channel.close();
                        promise.setFailure(future.cause());
                    } else {
                        doConnect(future.getNow(), localAddress, promise);//对目标地址发起连接
                    }
                }
            });
        } catch (Throwable cause) {
            promise.tryFailure(cause);
        }
        return promise;
    }
```

封装成task任务交由channel对应的eventLoop线程来执行，防止并发操作channel

```java
    private static void doConnect(
            final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise connectPromise) {

        // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
        // the pipeline in its channelRegistered() implementation.
        final Channel channel = connectPromise.channel();
        channel.eventLoop().execute(new Runnable() {//封装成task任务交由channel对应的eventLoop线程来执行，防止并发操作channel
            @Override
            public void run() {
                if (localAddress == null) {
                    channel.connect(remoteAddress, connectPromise);//执行channel的connect方法进行连接
                } else {
                    channel.connect(remoteAddress, localAddress, connectPromise);
                }
                connectPromise.addListener(ChannelFutureListener.CLOSE_ON_FAILURE);//如果连接失败则通过回调进行关闭
            }
        });
    }
```

从责任链中获取ChannelOutboundHandler执行connect方法

```java
    public ChannelFuture connect(
            final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {

        if (remoteAddress == null) {
            throw new NullPointerException("remoteAddress");
        }
        if (!validatePromise(promise, false)) {
            // cancelled
            return promise;
        }

        final AbstractChannelHandlerContext next = findContextOutbound();//从责任链中获取ChannelOutboundHandler
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {//如果当前线程是channel对应的EventLoop线程，则直接执行，否则封装成task交由channel对应的EventLoop线程执行invokeConnect方法
            next.invokeConnect(remoteAddress, localAddress, promise);
        } else {
            safeExecute(executor, new Runnable() {
                @Override
                public void run() {
                    next.invokeConnect(remoteAddress, localAddress, promise);
                }
            }, promise, null);
        }
        return promise;
    }
```

```java
    private void invokeConnect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise) {
        if (invokeHandler()) {
            try {
                ((ChannelOutboundHandler) handler()).connect(this, remoteAddress, localAddress, promise);
            } catch (Throwable t) {
                notifyOutboundHandlerException(t, promise);
            }
        } else {
            connect(remoteAddress, localAddress, promise);
        }
    }
```

连接需要用的ChannelOutboundHandler是责任链的头部ChannelHandler：HeadContext

```java
        public void connect(
                ChannelHandlerContext ctx,
                SocketAddress remoteAddress, SocketAddress localAddress,
                ChannelPromise promise) throws Exception {
            unsafe.connect(remoteAddress, localAddress, promise);
        }
```

最终调用unsafe的connect方法，也是真正进行连接操作的核心逻辑

```java
        public final void connect(
                final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
            if (!promise.setUncancellable() || !ensureOpen(promise)) {
                return;
            }

            try {
                if (connectPromise != null) {
                    // Already a connect in process.
                    throw new ConnectionPendingException();
                }

                boolean wasActive = isActive();
                if (doConnect(remoteAddress, localAddress)) {//发起连接，如果成功，则执行fulfillConnectPromise触发fireChannelActive事件，
                    fulfillConnectPromise(promise, wasActive);
                } else {//如果连接未成功，需要添加处理连接超时的定时任务
                    connectPromise = promise;
                    requestedRemoteAddress = remoteAddress;

                    // Schedule connect timeout.
                    int connectTimeoutMillis = config().getConnectTimeoutMillis();//获取连接超时时间。默认30s，最好根据实际情况使用ChannelOption定制合理的超时时间
                    if (connectTimeoutMillis > 0) {
                        connectTimeoutFuture = eventLoop().schedule(new Runnable() {//根据连接超时时间设置定时任务，如果超时仍未能连接则关闭连接，设置连接结果Promise为失败
                            @Override
                            public void run() {
                                ChannelPromise connectPromise = AbstractNioChannel.this.connectPromise;
                                ConnectTimeoutException cause =
                                        new ConnectTimeoutException("connection timed out: " + remoteAddress);
                                if (connectPromise != null && connectPromise.tryFailure(cause)) {
                                    close(voidPromise());
                                }
                            }
                        }, connectTimeoutMillis, TimeUnit.MILLISECONDS);
                    }

                    promise.addListener(new ChannelFutureListener() {//为连接结果promise设置Listener，当完成操作后触发回调，如果连接失败，则取消掉处理当前连接超时的定时任务。
                        @Override
                        public void operationComplete(ChannelFuture future) throws Exception {
                            if (future.isCancelled()) {
                                if (connectTimeoutFuture != null) {
                                    connectTimeoutFuture.cancel(false);
                                }
                                connectPromise = null;
                                close(voidPromise());
                            }
                        }
                    });
                }
            } catch (Throwable t) {
                promise.tryFailure(annotateConnectException(t, remoteAddress));
                closeIfClosed();
            }
        }
```

```java
    protected boolean doConnect(SocketAddress remoteAddress, SocketAddress localAddress) throws Exception {
        if (localAddress != null) {
            doBind0(localAddress);
        }

        boolean success = false;
        try {
            boolean connected = javaChannel().connect(remoteAddress);//获取SocketChannel发起连接
            if (!connected) {//如果连接失败需要设置`SelectionKey.OP_CONNECT`监听位。
                selectionKey().interestOps(SelectionKey.OP_CONNECT);
            }
            success = true;
            return connected;//返回是否连接成功的结果
        } finally {
            if (!success) {
                doClose();
            }
        }
    }
```

调用doConnect方法获得底层NIO原生的SocketChannel发起连接操作，并返回连接操作的结果，因为此处的SocketChannel被设置为非阻塞，所以这里有可能没有立即连接成功，如果当前操作立即连接成功，则调用fulfillConnectPromise方法进而触发fireChannelActive事件；如果当前操作未能立即连接成功，则设置`SelectionKey.OP_CONNECT`监听位，异步监听连接就绪事件来完成连接操作，并为当前连接添加处理连接超时的定时任务task，并设置Listener用来在连接失败时清理废弃的定时任务task。

如果当前操作立即连接成功，则调用fulfillConnectPromise方法进而触发fireChannelActive事件

```java
        private void fulfillConnectPromise(ChannelPromise promise, boolean wasActive) {
            if (promise == null) {
                // Closed via cancellation and the promise has been notified already.
                return;
            }

            // Get the state as trySuccess() may trigger an ChannelFutureListener that will close the Channel.
            // We still need to ensure we call fireChannelActive() in this case.
            boolean active = isActive();

            // trySuccess() will return false if a user cancelled the connection attempt.
            boolean promiseSet = promise.trySuccess();

            // Regardless if the connection attempt was cancelled, channelActive() event should be triggered,
            // because what happened is what happened.
            if (!wasActive && active) {//需要保证是当前操作完成的连接操作，才触发fireChannelActive事件，确保fireChannelActive事件只被执行一次，这块不会有并发问题是因为对一个channel的操作都是在channel对应的eventLoop线程内串行执行的。
                pipeline().fireChannelActive();
            }

            // If a user cancelled the connection attempt, close the channel, which is followed by channelInactive().
            if (!promiseSet) {
                close(voidPromise());
            }
        }
```

如果当前操作未能立即连接成功，则设置`SelectionKey.OP_CONNECT`监听位，异步监听连接就绪事件来完成连接操作，异步监听的逻辑在NioEventLoop的处理就绪的SelectedKey的方法中

```java
            if ((readyOps & SelectionKey.OP_CONNECT) != 0) {//监听SelectionKey.OP_CONNECT就绪事件
                // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
                // See https://github.com/netty/netty/issues/924
                int ops = k.interestOps();
                ops &= ~SelectionKey.OP_CONNECT;//已经监听到SelectionKey.OP_CONNECT就绪事件，需要清除掉SelectionKey.OP_CONNECT的监听位
                k.interestOps(ops);

                unsafe.finishConnect();//完成连接操作，触发fireChannelActive事件
            }
```

```java
        public final void finishConnect() {
            // Note this method is invoked by the event loop only if the connection attempt was
            // neither cancelled nor timed out.

            assert eventLoop().inEventLoop();

            try {
                boolean wasActive = isActive();
                doFinishConnect();//完成连接操作
                fulfillConnectPromise(connectPromise, wasActive);//完成连接操作后调用之前的fulfillConnectPromise方法进而触发fireChannelActive事件
            } catch (Throwable t) {
                fulfillConnectPromise(connectPromise, annotateConnectException(t, requestedRemoteAddress));//连接失败需要设置连接结果Promise为失败，并清理关闭该连接
            } finally {
                // Check for null as the connectTimeoutFuture is only created if a connectTimeoutMillis > 0 is used
                // See https://github.com/netty/netty/issues/1770
                if (connectTimeoutFuture != null) {//如果有处理连接超时的定时任务则取消该任务
                    connectTimeoutFuture.cancel(false);
                }
                connectPromise = null;
            }
        }
```

```java
    protected void doFinishConnect() throws Exception {
        if (!javaChannel().finishConnect()) {//获取NIO原生api的SocketChannel调用finishConnect方法完成连接操作，返回false代表连接失败
            throw new Error();
        }
    }
```

至此，Bootstrap客户端启动部分源码已经分析完成。
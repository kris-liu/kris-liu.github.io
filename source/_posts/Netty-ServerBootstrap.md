---
title: Netty源码分析 ServerBootstrap服务端启动
date: 2017-04-01 20:00:00
categories: Netty
tags: [Netty, 源码]
---

> 本文使用`netty-4.1.5.Final`版本源码进行分析

ServerBootstrap是Socket服务端的启动辅助类，用户通过ServerBootstrap可以方便的创建Netty的服务端，并加以定制。下面我们看一下服务端ServerBootstrap如何启动并对启动过程的源码进行分析。

<!--more-->

## ServerBootstrap服务端启动

```java
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class)
             .option(ChannelOption.SO_BACKLOG, 100)
             .handler(new LoggingHandler(LogLevel.INFO))
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     p.addLast(new EchoServerHandler());
                 }
             });

            // Start the server.
            ChannelFuture f = b.bind(PORT).sync();

            // Wait until the server socket is closed.
            f.channel().closeFuture().sync();
        } finally {
            // Shut down all event loops to terminate all threads.
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
```

创建ServerBootstrap实例，绑定用于监听TCP连接的bossGroup和用于处理I/O事件及task任务的workerGroup，并设置其他相关定制参数，最后绑定一个监听TCP连接的端口，就完成了启动Netty服务端的代码。


## ServerBootstrap服务端启动源码分析

### 创建ServerBootstrap实例

```java
	ServerBootstrap b = new ServerBootstrap();
```

### 设置ServerBootstrap参数

绑定Reactor线程池

```java
    public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
        super.group(parentGroup);
        if (childGroup == null) {
            throw new NullPointerException("childGroup");
        }
        if (this.childGroup != null) {
            throw new IllegalStateException("childGroup set already");
        }
        this.childGroup = childGroup;
        return this;
    }
    
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

parentGroup线程组主要用于处理TCP连接请求，childGroup线程组主要用于I/O读写以及task执行。


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

根据channel的类型，创建一个生产channel的工厂，用于通过channel类型反射创建对应的channel，服务端一般使用`NioServerSocketChannel.class`。


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
为NioServerSocketChannel设置ChannelHandler。

设置子handler

```java
    public ServerBootstrap childHandler(ChannelHandler childHandler) {
        if (childHandler == null) {
            throw new NullPointerException("childHandler");
        }
        this.childHandler = childHandler;
        return this;
    }
```
每个NioSocketChannel设置ChannelHandler，ChannelHandler是netty提供给用户定制和扩展的关键接口，用户可以通过自定义ChannelHandler，添加具体的业务逻辑处理。


### 启动ServerBootstrap的核心逻辑源码分析

绑定一个监听端口来启动服务端ServerBootstrap，绑定操作不仅要绑定端口并设置监听，首先需要将Channel初始化并注册到eventLoop中。

ServerBootstrap启动的核心逻辑总共分三步操作：

1. 初始化
2. 注册
3. 绑定端口

首先看下启动过程的主干逻辑，然后再具体分析每一步操作的具体逻辑：

```java
    public ChannelFuture bind(SocketAddress localAddress) {
        validate();//参数基本校验
        if (localAddress == null) {
            throw new NullPointerException("localAddress");
        }
        return doBind(localAddress);//发起绑定并返回绑定结果
    }
```

```java
    private ChannelFuture doBind(final SocketAddress localAddress) {
        final ChannelFuture regFuture = initAndRegister();//初始化NioServerSocketChannel并注册到bossGroup的eventLoop上
        final Channel channel = regFuture.channel();
        if (regFuture.cause() != null) {
            return regFuture;
        }

        if (regFuture.isDone()) {//如果注册到bossGroup上已完成则直接进行绑定操作
            // At this point we know that the registration was complete and successful.
            ChannelPromise promise = channel.newPromise();
            doBind0(regFuture, channel, localAddress, promise);//绑定端口
            return promise;
        } else {//如果注册到bossGroup上还未完成，则添加Listener当执行完注册操作后再回调Listener进行端口绑定操作
            // Registration future is almost always fulfilled already, but just in case it's not.
            final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    Throwable cause = future.cause();
                    if (cause != null) {
                        // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                        // IllegalStateException once we try to access the EventLoop of the Channel.
                        promise.setFailure(cause);
                    } else {
                        // Registration was successful, so set the correct executor to use.
                        // See https://github.com/netty/netty/issues/2586
                        promise.registered();

                        doBind0(regFuture, channel, localAddress, promise);//绑定端口
                    }
                }
            });
            return promise;//外部可以用过promise得到是否绑定完成的结果，或者阻塞直到完成绑定。
        }
    }
```

首先初始化NioServerSocketChannel并注册到bossGroup的eventLoop上，当NioServerSocketChannel已经执行完成注册操作，则直接进行端口绑定操作，否则添加Listener当执行完注册操作后再回调Listener进行端口绑定操作。

初始化NioServerSocketChannel并注册到bossGroup的eventLoop上

```java
    final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            channel = channelFactory.newChannel();//通过创建ServerBootstrap时设置的channel创建工厂通过反射创建对应的channel，服务端一般是NioServerSocketChannel
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

创建并初始化NioServerSocketChannel，然后将其注册到bossGroup的eventLoop上去，并返回注册操作的结果。


#### 初始化

```java
    void init(Channel channel) throws Exception {
        final Map<ChannelOption<?>, Object> options = options0();//设置channel的参数
        synchronized (options) {
            channel.config().setOptions(options);
        }

        final Map<AttributeKey<?>, Object> attrs = attrs0();//设置channel的附加属性
        synchronized (attrs) {
            for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
                @SuppressWarnings("unchecked")
                AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
                channel.attr(key).set(e.getValue());
            }
        }

        ChannelPipeline p = channel.pipeline();//获取负责处理网络事件的职责链，用来管理和执行ChannelHandler

        //处理ServerBootstrapAcceptor的参数
        final EventLoopGroup currentChildGroup = childGroup;//workGroup，处理I/O相关操作的线程组
        final ChannelHandler currentChildHandler = childHandler;//创建ServerBootstrap时设置的childHandler
        final Entry<ChannelOption<?>, Object>[] currentChildOptions;
        final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
        synchronized (childOptions) {
            currentChildOptions = childOptions.entrySet().toArray(newOptionArray(childOptions.size()));//创建ServerBootstrap时设置的socket属性childOptions
        }
        synchronized (childAttrs) {
            currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(childAttrs.size()));//创建ServerBootstrap时设置的附加参数childAttrs
        }

        p.addLast(new ChannelInitializer<Channel>() {//NioServerSocketChannel注册的时候会调用initChannel方法设置ChannelHandler
            @Override
            public void initChannel(Channel ch) throws Exception {
                final ChannelPipeline pipeline = ch.pipeline();
                ChannelHandler handler = config.handler();//将创建ServerBootstrap时设置的handler添加到NioServerSocketChannel的责任链上
                if (handler != null) {
                    pipeline.addLast(handler);
                }

                // We add this handler via the EventLoop as the user may have used a ChannelInitializer as handler.
                // In this case the initChannel(...) method will only be called after this method returns. Because
                // of this we need to ensure we add our handler in a delayed fashion so all the users handler are
                // placed in front of the ServerBootstrapAcceptor.
                ch.eventLoop().execute(new Runnable() {//封装成task任务交由channel对应的eventLoop线程来执行，防止并发操作channel
                    @Override
                    public void run() {
                        pipeline.addLast(new ServerBootstrapAcceptor(
                                currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));//添加ServerBootstrapAcceptor，主要用于接收TCP连接后初始化并注册NioSocketChannel到workGroup
                    }
                });
            }
        });
    }
```

ServerBootstrapAcceptor主要用于接收TCP连接后初始化并注册NioSocketChannel到workGroup

```java
    private static class ServerBootstrapAcceptor extends ChannelInboundHandlerAdapter {

        private final EventLoopGroup childGroup;//workGroup，处理I/O相关操作的线程组
        private final ChannelHandler childHandler;//创建ServerBootstrap时设置的childHandler
        private final Entry<ChannelOption<?>, Object>[] childOptions;//创建ServerBootstrap时设置的socket属性childOptions
        private final Entry<AttributeKey<?>, Object>[] childAttrs;//创建ServerBootstrap时设置的附加参数childAttrs

        ServerBootstrapAcceptor(
                EventLoopGroup childGroup, ChannelHandler childHandler,
                Entry<ChannelOption<?>, Object>[] childOptions, Entry<AttributeKey<?>, Object>[] childAttrs) {
            this.childGroup = childGroup;
            this.childHandler = childHandler;
            this.childOptions = childOptions;
            this.childAttrs = childAttrs;
        }

        @Override
        @SuppressWarnings("unchecked")
        public void channelRead(ChannelHandlerContext ctx, Object msg) {//Channel关心的网络事件`SelectionKey.OP_READ`或者`SelectionKey.OP_ACCEPT`发生都会触发read事件，进而调用责任链上ChannelHandler的channelRead相关逻辑，此处是因为`SelectionKey.OP_ACCEPT`事件导致的channelRead事件被触发。
        
            final Channel child = (Channel) msg;//msg即NioSocketChannel

            child.pipeline().addLast(childHandler);//为NioSocketChannel的责任链上添加childHandler

            for (Entry<ChannelOption<?>, Object> e: childOptions) {//为NioSocketChannel添加socket属性Option
                try {
                    if (!child.config().setOption((ChannelOption<Object>) e.getKey(), e.getValue())) {
                        logger.warn("Unknown channel option: " + e);
                    }
                } catch (Throwable t) {
                    logger.warn("Failed to set a channel option: " + child, t);
                }
            }

            for (Entry<AttributeKey<?>, Object> e: childAttrs) {//为NioSocketChannel添加附加参数attr

                child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
            }

            try {
                childGroup.register(child).addListener(new ChannelFutureListener() {//将child注册到workGroup线程组上并添加Listener用于处理注册失败后的关闭操作
                    @Override
                    public void operationComplete(ChannelFuture future) throws Exception {
                        if (!future.isSuccess()) {
                            forceClose(child, future.cause());
                        }
                    }
                });
            } catch (Throwable t) {
                forceClose(child, t);
            }
        }

        private static void forceClose(Channel child, Throwable t) {
            child.unsafe().closeForcibly();
            logger.warn("Failed to register an accepted channel: " + child, t);
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {//当发生链路异常，则关闭channel的AutoRead属性，并设置定时task在1s后恢复AutoRead属性，关闭AutoRead将导致不再监听新的连接请求，用于对java.io.IOException: Too many open files进行处理。
            final ChannelConfig config = ctx.channel().config();
            if (config.isAutoRead()) {
                // stop accept new connections for 1 second to allow the channel to recover
                // See https://github.com/netty/netty/issues/1328
                config.setAutoRead(false);
                ctx.channel().eventLoop().schedule(new Runnable() {//定时task需要提交给channel对应的eventLoop线程处理
                    @Override
                    public void run() {
                        config.setAutoRead(true);
                    }
                }, 1, TimeUnit.SECONDS);
            }
            // still let the exceptionCaught event flow through the pipeline to give the user
            // a chance to do something with it
            ctx.fireExceptionCaught(cause);
        }
    }
```

Channel关心的网络事件`SelectionKey.OP_READ`或者`SelectionKey.OP_ACCEPT`发生都会触发read事件，进而调用责任链上ChannelHandler的channelRead相关逻辑，此处是因为`SelectionKey.OP_ACCEPT`事件导致的ServerBootstrapAcceptor的channelRead事件被触发。这里主要是用于accept到连接后，对新建的NioSocketChannel进行属性设置并注册到workGroup线程组。

#### 注册

注册NioServerSocketChannel，从bossGroup选择一个eventLoop线程，将NioServerSocketChannel注册到该eventLoop的selector上，通过channel获取unsafe，进而操作底层NIO的api进行注册操作

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

#### 绑定端口

提交注册后回到doBind端口绑定方法，当NioServerSocketChannel已经执行完成注册操作，则直接进行端口绑定操作，否则添加Listener当执行完注册操作后再回调Listener进行端口绑定操作，绑定操作需要操作channel，所以需要封装成task任务交由channel对应的eventLoop线程执行其bind方法

```java
    private static void doBind0(
            final ChannelFuture regFuture, final Channel channel,
            final SocketAddress localAddress, final ChannelPromise promise) {

        // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
        // the pipeline in its channelRegistered() implementation.
        channel.eventLoop().execute(new Runnable() {//封装成task任务交由channel对应的eventLoop线程来执行，防止并发操作channel
            @Override
            public void run() {
                if (regFuture.isSuccess()) {//注册成功才需要绑定
                    channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);//执行绑定操作，Listener用于处理bind失败时的close操作
                } else {
                    promise.setFailure(regFuture.cause());
                }
            }
        });
    }
```

```java
    public void execute(Runnable task) {
        if (task == null) {
            throw new NullPointerException("task");
        }

        boolean inEventLoop = inEventLoop();
        if (inEventLoop) {//如果当前线程是EventLoop线程，直接将任务提交到队列，否则需要启动EventLoop线程再提交任务
            addTask(task);
        } else {
            startThread();//当前线程不是EventLoop线程，则看是否需要先启动EventLoop的线程
            addTask(task);
            if (isShutdown() && removeTask(task)) {
                reject();
            }
        }

        if (!addTaskWakesUp && wakesUpForTask(task)) {
            wakeup(inEventLoop);//唤醒Selector，进而唤醒EventLoop线程来处理task任务
        }
    }
```

为了防止并发操作channel，需要由channel对应的eventLoop线程执行bind操作，此处将bind操作封装成task任务交由对应的eventLoop线程执行bind操作。

从责任链中获取ChannelOutboundHandler执行bind方法

```java 
    public ChannelFuture bind(final SocketAddress localAddress, final ChannelPromise promise) {
        if (localAddress == null) {
            throw new NullPointerException("localAddress");
        }
        if (!validatePromise(promise, false)) {
            // cancelled
            return promise;
        }

        final AbstractChannelHandlerContext next = findContextOutbound();//从责任链中获取ChannelOutboundHandler
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {//如果当前线程是channel对应的EventLoop线程，则直接执行，否则封装成task交由channel对应的EventLoop线程执行invokeBind方法
            next.invokeBind(localAddress, promise);
        } else {
            safeExecute(executor, new Runnable() {
                @Override
                public void run() {
                    next.invokeBind(localAddress, promise);
                }
            }, promise, null);
        }
        return promise;
    }
```

```java
    private void invokeBind(SocketAddress localAddress, ChannelPromise promise) {
        if (invokeHandler()) {
            try {
                ((ChannelOutboundHandler) handler()).bind(this, localAddress, promise);
            } catch (Throwable t) {
                notifyOutboundHandlerException(t, promise);
            }
        } else {
            bind(localAddress, promise);
        }
    }
```

绑定需要用的ChannelOutboundHandler是责任链的头部ChannelHandler：HeadContext

```java
        public void bind(
                ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise)
                throws Exception {
            unsafe.bind(localAddress, promise);
        }
```

最终调用unsafe的bind方法

```java
        public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
            assertEventLoop();

            if (!promise.setUncancellable() || !ensureOpen(promise)) {
                return;
            }

            // See: https://github.com/netty/netty/issues/576
            if (Boolean.TRUE.equals(config().getOption(ChannelOption.SO_BROADCAST)) &&
                localAddress instanceof InetSocketAddress &&
                !((InetSocketAddress) localAddress).getAddress().isAnyLocalAddress() &&
                !PlatformDependent.isWindows() && !PlatformDependent.isRoot()) {
                // Warn a user about the fact that a non-root user can't receive a
                // broadcast packet on *nix if the socket is bound on non-wildcard address.
                logger.warn(
                        "A non-root user can't receive a broadcast packet if the socket " +
                        "is not bound to a wildcard address; binding to a non-wildcard " +
                        "address (" + localAddress + ") anyway as requested.");
            }

            boolean wasActive = isActive();//此处为完成绑定
            try {
                doBind(localAddress);//使用原生api绑定端口
            } catch (Throwable t) {
                safeSetFailure(promise, t);
                closeIfClosed();
                return;
            }

            if (!wasActive && isActive()) {//如果之前未完成绑定，执行完doBind后isbind方法返回完成绑定，则触发责任链的fireChannelActive事件，fireChannelActive最终会设置SelectionKey.OP_ACCEPT监听位
                invokeLater(new Runnable() {
                    @Override
                    public void run() {
                        pipeline.fireChannelActive();
                    }
                });
            }

            safeSetSuccess(promise);//设置promise结果为已成功完成绑定操作，外部可以通过bind返回的该promise得知已完成绑定操作。
        }
```

doBind获得底层NIO原生的ServerSocketChannel进行绑定端口操作

```java
    @Override
    protected void doBind(SocketAddress localAddress) throws Exception {
        if (PlatformDependent.javaVersion() >= 7) {
            javaChannel().bind(localAddress, config.getBacklog());//获取NIO原生的ServerSocketChannel进行绑定端口操作
        } else {
            javaChannel().socket().bind(localAddress, config.getBacklog());
        }
    }
```

绑定完成后如果绑定成功则触发fireChannelActive方法，通过责任链会调用责任链的头部HeadContext处理，主要用于设置channel需要监听的操作位

```java
        public void channelActive(ChannelHandlerContext ctx) throws Exception {
            ctx.fireChannelActive();

            readIfIsAutoRead();
        }

        private void readIfIsAutoRead() {
            if (channel.config().isAutoRead()) {
                channel.read();
            }
        }

        public void read(ChannelHandlerContext ctx) {
            unsafe.beginRead();
        }
```

最终调用unsafe的方法操作底层NIO的原生api做处理

```java
        public final void beginRead() {
            assertEventLoop();

            if (!isActive()) {//没有绑定或者没有完成连接，则不需要处理监听位
                return;
            }

            try {
                doBeginRead();
            } catch (final Exception e) {
                invokeLater(new Runnable() {
                    @Override
                    public void run() {
                        pipeline.fireExceptionCaught(e);
                    }
                });
                close(voidPromise());
            }
        }
```

在这里根据当前channel类型设置相应的监听位，此处处理的是NioServerSocketChannel，所以最终会设置`SelectionKey.OP_ACCEPT`监听位

```java
    protected void doBeginRead() throws Exception {
        // Channel.read() or ChannelHandlerContext.read() was called
        final SelectionKey selectionKey = this.selectionKey;
        if (!selectionKey.isValid()) {
            return;
        }

        readPending = true;

        final int interestOps = selectionKey.interestOps();
        if ((interestOps & readInterestOp) == 0) {//如果SelectionKey没有当前channel需要关心的监听位，则添加该监听
            selectionKey.interestOps(interestOps | readInterestOp);
        }
    }
```

至此，ServerBootstrap服务端启动部分源码已经分析完成。
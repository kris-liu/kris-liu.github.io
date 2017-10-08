---
title: Netty源码分析 解决TCP粘包拆包问题
date: 2017-04-23 20:00:00
categories: Netty
tags: [Netty, 源码]
---

> 本文使用`netty-4.1.5.Final`版本源码进行分析

客户端或者服务端在进行消息发送和接收的时候，需要考虑TCP底层的粘包拆包机制。一个完整的包可能会被TCP拆分成多个包进行发送，也有可能把多个小的包封装成一个大的数据包发送，这就是所谓的TCP粘包和拆包问题。一般需要上层的应用协议栈设计上，来在应用层，识别出单个完整的数据包。

<!--more-->


### 粘包、拆包发生原因

发生TCP粘包或拆包有很多原因，常见的几点:

1. 要发送的数据大于TCP发送缓冲区剩余空间大小，将会发生拆包。

2. 待发送数据大于MSS（最大报文长度），TCP在传输前将进行拆包。

3. 要发送的数据小于TCP发送缓冲区的大小，TCP将多次写入缓冲区的数据一次发送出去，将会发生粘包。

4. 接收数据端的应用层没有及时读取接收缓冲区中的数据，将发生粘包。等等。



### 粘包、拆包解决策略

TCP底层无法识别业务上一个完整的数据包，所以需要在设计应用协议栈的时候，在应用层，识别出单个完整的数据包。一般需要在消息接受方做应用层协议栈的处理，协议需要能识别出当前是否已经读取完了一个完整的数据包，如果读取到多个数据包，需要分割多个数据包，拆分成多个完整的数据包；如果没有读取到完整的数据包，需要继续读取，直到读取到一个完整的数据包，最终将单个完整的数据包交由应用做进一步处理。

主流协议的解决方案：

1. 消息定长。优点是最简单；缺点就是消息体长度受限，长度太长浪费，长度不够又限制应用数据传输，所以一般应用上不会采取
2. 包尾增加分隔符。需要保证分割符不会出现在应用传输的数据中
3. 消息中包含消息的总长度。比如，在消息的前缀增加4个字节int值作为当前数据包总长度；比如dubbo协议，由一个定长包头和变长包体组成，包头中的特定字节中包含变长包体的总长度。这种方式相对更灵活。

### Netty解决策略

为了解决TCP粘包、拆包导致的半包读写问题，Netty默认提供了多种编解码器用于处理半包，或者可以根据实际情况自行实现ChannelHandler来定制自己的应用协议栈，一般可以直接实现ByteToMessageDecoder。使用时只需要将需要的编解码器添加到channel的责任链上即可，熟练掌握这些类库的使用，就可以非常容易的处理TCP粘包拆包带来的问题。

#### Netty默认提供的编解码器：

* 定长消息解码器：
	* FixedLengthFrameDecoder

	从ByteBuf中读取自定义的长度的数据，做为一个完整数据包。

* 分隔符消息解码器：
	* LineBasedFrameDecoder

	依次遍历ByteBuf中的可读字节，判断看是否有“\n”或者“\r\n”，如果有，就以此位置为结束位置。

	* DelimiterBasedFrameDecoder

	依次遍历ByteBuf中的可读字节，判断看是否有自定义的分隔符，如果有，就以此位置为结束位置。

* 定长字段标识消息长度的消息解码器：
	* LengthFieldBasedFrameDecoder
	* ProtobufVarint32FrameDecoder

	读取ByteBuf的某几个字节，转化为长度值，做为消息体的总长度。


#### 源码分析

我们以定长消息解码器FixedLengthFrameDecoder源码来看下Netty如何处理粘包拆包问题的

```java
public class FixedLengthFrameDecoder extends ByteToMessageDecoder {

    private final int frameLength;

    public FixedLengthFrameDecoder(int frameLength) {
        if (frameLength <= 0) {
            throw new IllegalArgumentException(
                    "frameLength must be a positive integer: " + frameLength);
        }
        this.frameLength = frameLength;
    }

    @Override
    protected final void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        Object decoded = decode(ctx, in);
        if (decoded != null) {//如果decode到一个完整数据包，则添加该数据包
            out.add(decoded);
        }
    }

    protected Object decode(
            @SuppressWarnings("UnusedParameters") ChannelHandlerContext ctx, ByteBuf in) throws Exception {
        if (in.readableBytes() < frameLength) {//如果可读字节小于定长长度，则不处理
            return null;
        } else {//如果有足够的字节可读，就立刻读区出frameLength长度的字节做为一个完整数据包
            return in.readRetainedSlice(frameLength);
        }
    }
}
```

```java
    ByteBuf cumulation;//cumulation用于每次channelRead事件触发后，保存下来所有当前可读字节，用于后序处理，处理完成后，如果没有内容可读，代表数据里都是完整数据包，如果处理完成后还有内容可读，那么意味着有半包消息，需要用cumulation保存下来未解析完的半包消息内容
    
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        if (msg instanceof ByteBuf) {
            CodecOutputList out = CodecOutputList.newInstance();
            try {
                ByteBuf data = (ByteBuf) msg;
                first = cumulation == null;
                if (first) {
                    cumulation = data;//如果之前都是整包，cumulation直接设置为当前的ByteBuf
                } else {
                    cumulation = cumulator.cumulate(ctx.alloc(), cumulation, data);//如果之前有半包，需要将新的ByteBuf添加到cumulation尾部继续处理之前的半包和新的消息
                }
                callDecode(ctx, cumulation, out);
            } catch (DecoderException e) {
                throw e;
            } catch (Throwable t) {
                throw new DecoderException(t);
            } finally {
                if (cumulation != null && !cumulation.isReadable()) {//代表当前所有消息内容都读取完成，没有剩余的半包消息
                    numReads = 0;
                    cumulation.release();
                    cumulation = null;//cumulation置空，下次不需要处理剩余半包消息
                } else if (++ numReads >= discardAfterReads) {
                    // We did enough reads already try to discard some bytes so we not risk to see a OOME.
                    // See https://github.com/netty/netty/issues/4275
                    numReads = 0;
                    discardSomeReadBytes();//清除掉历史已经读取出来的已读字节，防止OOM
                }

                int size = out.size();
                decodeWasNull = !out.insertSinceRecycled();
                fireChannelRead(ctx, out, size);//循环将每个完整数据包都进行一次fireChannelRead事件的传播处理
                out.recycle();
            }
        } else {
            ctx.fireChannelRead(msg);
        }
    }
    
    static void fireChannelRead(ChannelHandlerContext ctx, CodecOutputList msgs, int numElements) {
        for (int i = 0; i < numElements; i ++) {//循环将每个完整数据包都进行一次fireChannelRead事件的传播处理
            ctx.fireChannelRead(msgs.getUnsafe(i));
        }
    }
    
    protected void callDecode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        try {
            while (in.isReadable()) {//如果当前ByteBuf仍然可读，则继续读取，直到没有完整数据包，才退出
                int outSize = out.size();

                if (outSize > 0) {//有完整的数据包
                    fireChannelRead(ctx, out, outSize);//循环将每个完整数据包都进行一次fireChannelRead事件的传播处理
                    out.clear();//清除已经处理完的完整数据包

                    // Check if this handler was removed before continuing with decoding.
                    // If it was removed, it is not safe to continue to operate on the buffer.
                    //
                    // See:
                    // - https://github.com/netty/netty/issues/4635
                    if (ctx.isRemoved()) {
                        break;
                    }
                    outSize = 0;
                }

                int oldInputLength = in.readableBytes();
                decode(ctx, in, out);//进行一次完整数据包读取

                // Check if this handler was removed before continuing the loop.
                // If it was removed, it is not safe to continue to operate on the buffer.
                //
                // See https://github.com/netty/netty/issues/1664
                if (ctx.isRemoved()) {
                    break;
                }

                if (outSize == out.size()) {//没有decode出新的完整数据包
                    if (oldInputLength == in.readableBytes()) {//代表decode操作没有进行实际的读取操作，意味着没有读取到完整数据包，跳出循环，否则尝试继续读取
                        break;
                    } else {
                        continue;
                    }
                }

                if (oldInputLength == in.readableBytes()) {
                    throw new DecoderException(
                            StringUtil.simpleClassName(getClass()) +
                            ".decode() did not read anything but decoded a message.");
                }

                if (isSingleDecode()) {
                    break;
                }
            }
        } catch (DecoderException e) {
            throw e;
        } catch (Throwable cause) {
            throw new DecoderException(cause);
        }
    }
```

#### 总结

通过看FixedLengthFrameDecoder，再对比其他几种处理粘包拆包的编解码器，可以总结出来，Netty处理粘包拆包问题的核心思路就是：每次读取的时候，如果能读取到一个完整的数据包，才真正读取出来，一直读到没有数据可读，如果有半包消息，则保存下来未处理的半包消息，下次读事件触发的时候，将未处理的半包消息和新的消息内容合并在一起再继续处理。最后将所有解析出来的完整数据包依次进行fireChannelRead事件的传播，进行后续的业务处理。


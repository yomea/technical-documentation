在上一节中，我们分析了调用接口的过程，当接口返回了数据之后，dubbo需要告诉客户端调用的接口，这个时候dubbo又是这么进行回应的呢？在com.alibaba.dubbo.remoting.exchange.support.header.HeaderExchangeHandler#received方法中有如下代码

```
public void received(Channel channel, Object message) throws RemotingException {
        。。。。。。省略
                if (request.isTwoWay()) {
                    //这里我们在第6小节的时候阅读过
                    //这里将返回值包装成Response
                    Response response = handleRequest(exchangeChannel, request);
                    //回应客户端
                    channel.send(response);
                }
            }
            
             。。。。。。省略
    }
}
```
在消息发送到客户端的时候是需要编码的，把Response进行编码，在创建netty服务的时候，dubbo向netty通道加入了三个通道处理器，NettyServerHandler，com.alibaba.dubbo.remoting.transport.netty4.NettyCodecAdapter.InternalEncoder，com.alibaba.dubbo.remoting.transport.netty4.NettyCodecAdapter.InternalDecoder

下图是InternalEncoder编码器的类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ffef2f3084849c0da5d2659ec6f4dca7.png)
```
public void io.netty.handler.codec.MessageToByteEncoder#write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    ByteBuf buf = null;
    try {
        //检查msg的java类型是否支持
        if (acceptOutboundMessage(msg)) {
            @SuppressWarnings("unchecked")
            I cast = (I) msg;
            //申请内存创建ByteBuf
            buf = allocateBuffer(ctx, cast, preferDirect);
            try {
                //编码
                encode(ctx, cast, buf);
            } finally {
                //如果msg属于应用类，需要释放引用
                ReferenceCountUtil.release(cast);
            }
            //继续下一个通道处理，写出数据
            if (buf.isReadable()) {
                ctx.write(buf, promise);
            } else {
                //释放引用计数
                buf.release();
                //空数据
                ctx.write(Unpooled.EMPTY_BUFFER, promise);
            }
            buf = null;
        } else {
            //交由下一个通道处理
            ctx.write(msg, promise);
        }
    } catch (EncoderException e) {
        throw e;
    } catch (Throwable e) {
        throw new EncoderException(e);
    } finally {
        if (buf != null) {
            buf.release();
        }
    }
}
                                                            |
                                                            V
protected void com.alibaba.dubbo.remoting.transport.netty4.NettyCodecAdapter.InternalEncoder#encode(ChannelHandlerContext ctx, Object msg, ByteBuf out) throws Exception {
        com.alibaba.dubbo.remoting.buffer.ChannelBuffer buffer = new NettyBackedChannelBuffer(out);
        Channel ch = ctx.channel();
        NettyChannel channel = NettyChannel.getOrAddChannel(ch, url, handler);
        try {
            //通过DubboCountCodec最终调用DubboCodec的encode方法
            codec.encode(channel, buffer, msg);
        } finally {
            NettyChannel.removeChannelIfDisconnected(ch);
        }
    }
                                                            |
                                                            V
public void com.alibaba.dubbo.remoting.exchange.codec.ExchangeCodec(DubboCodec的实例)#encode(Channel channel, ChannelBuffer buffer, Object msg) throws IOException {
    //如果是请求，进行编码，比如客户端向服务提供者发送请求或者服务端向客户端发送心跳
    if (msg instanceof Request) {
        encodeRequest(channel, buffer, (Request) msg);
        //对响应编码
    } else if (msg instanceof Response) {
        encodeResponse(channel, buffer, (Response) msg);
    } else {
        //调用父类编码方法
        super.encode(channel, buffer, msg);
    }
}

```
对Response进行编码，我们先来回顾下在第5节分析的dubbo协议数据

- 0~15位：魔数

- 16位：是否为Request,1为request，0为response

- 17位：是否为twoWay，从源代码来看这个twoWay的意思好像表示是否需要响应的意思（可能理解的有误哈）

- 18位：是否是事件，表示心跳事件，可能当前的请求或者响应来自心跳，不是接口消费请求

- 19~23位：序列化id，用于标识当前使用的数据序列化方式是什么，比如hessian2

- 24~31位：状态，比如ok（20）

- 32~95位：请求的id，从后面的源码中，我发现它会以key为请求id -> DefaultFuture存储在com.alibaba.dubbo.remoting.exchange.support.DefaultFuture#FUTURES中 而这个DefaultFuture类似于JDK的Future，调用其get方法会发生阻塞知道有结果处理完毕

- 96~127位：标识请求体或者响应体的数据长度

- 128及以后：数据

```
protected void encodeResponse(Channel channel, ChannelBuffer buffer, Response res) throws IOException {
    int savedWriteIndex = buffer.writerIndex();
    try {
        //默认使用hessian2序列化
        Serialization serialization = getSerialization(channel);
        // header.
        //协议头，16个字节
        byte[] header = new byte[HEADER_LENGTH];
        // set magic number.
        //写入魔数
        Bytes.short2bytes(MAGIC, header);
        // set request and serialization flag.
        //第二个字节的低五位为序列化类型
        header[2] = serialization.getContentTypeId();
        //是否为事件类型
        if (res.isHeartbeat()) header[2] |= FLAG_EVENT;
        // set response status.
        byte status = res.getStatus();
        //第三个字节表示状态
        header[3] = status;
        // set request id.
        //在第四个字节开始写入请求id，占用8个字节
        Bytes.long2bytes(res.getId(), header, 4);
        //将写入数据的开始位置定位到16个字节后
        buffer.writerIndex(savedWriteIndex + HEADER_LENGTH);
        ChannelBufferOutputStream bos = new ChannelBufferOutputStream(buffer);
        //hessian2序列化
        ObjectOutput out = serialization.serialize(channel.getUrl(), bos);
        // encode response data or error message.
        if (status == Response.OK) {
            if (res.isHeartbeat()) {
                //编码心跳
                encodeHeartbeatData(channel, out, res.getResult());
            } else {
                //编码响应结果集
                encodeResponseData(channel, out, res.getResult(), res.getVersion());
            }
        } else 
        //如果接口调用失败的，那么写入的是错误信息
        out.writeUTF(res.getErrorMessage());
        //刷新
        out.flushBuffer();
        if (out instanceof Cleanable) {
            ((Cleanable) out).cleanup();
        }
        bos.flush();
        bos.close();
        
        int len = bos.writtenBytes();
        //检查负载
        checkPayload(channel, len);
        //写入长度
        Bytes.int2bytes(len, header, 12);
        // write
        //写入协议头
        buffer.writerIndex(savedWriteIndex);
        buffer.writeBytes(header); // write header.
        buffer.writerIndex(savedWriteIndex + HEADER_LENGTH + len);
    } catch (Throwable t) {
        。。。。。。省略异常处理
    }
}
```

Response数据体的编码写入

```
protected void com.alibaba.dubbo.rpc.protocol.dubbo.DubboCodec#encodeResponseData(Channel channel, ObjectOutput out, Object data, String version) throws IOException {
    Result result = (Result) data;
    // currently, the version value in Response records the version of Request
    //检查dubbo版本是否支持附加参数
    boolean attach = Version.isSupportResponseAttatchment(version);
    Throwable th = result.getException();
    if (th == null) {
        Object ret = result.getValue();
        //无异常，返回为null的
        if (ret == null) {
            out.writeByte(attach ? RESPONSE_NULL_VALUE_WITH_ATTACHMENTS : RESPONSE_NULL_VALUE);
        } else {
            //写入返回值
            out.writeByte(attach ? RESPONSE_VALUE_WITH_ATTACHMENTS : RESPONSE_VALUE);
            out.writeObject(ret);
        }
    } else {
        //吸入异常
        out.writeByte(attach ? RESPONSE_WITH_EXCEPTION_WITH_ATTACHMENTS : RESPONSE_WITH_EXCEPTION);
        out.writeObject(th);
    }

    if (attach) {
        // returns current version of Response to consumer side.
        //写入附加参数
        result.getAttachments().put(Constants.DUBBO_VERSION_KEY, Version.getProtocolVersion());
        out.writeObject(result.getAttachments());
    }
}
```
对于发送回应的处理，dubbo的通道处理没有做更多的逻辑，除了发送请求时在com.alibaba.dubbo.remoting.exchange.support.header.HeaderExchangeHandler#sent
记录一下Future

```
public void sent(Channel channel, Object message) throws RemotingException {
    。。。。。。省略部分代码
    if (message instanceof Request) {
        Request request = (Request) message;
        DefaultFuture.sent(channel, request);
    }
    。。。。。。省略部分代码
}
```
看了回应的编码，那么请求编码呢？其实我们在分析解析请求体的时候就可以了解到了，这只不过是一个逆过程而已



上一节我们分析了服务的暴露，这一节我们来分析dubbo是如果处理消费者的请求的，我们来回顾一下com.alibaba.dubbo.remoting.transport.netty4.NettyServer#doOpen方法

```
protected void com.alibaba.dubbo.remoting.transport.netty4.NettyServer#doOpen() throws Throwable {
    bootstrap = new ServerBootstrap();

    bossGroup = new NioEventLoopGroup(1, new DefaultThreadFactory("NettyServerBoss", true));
    workerGroup = new NioEventLoopGroup(getUrl().getPositiveParameter(Constants.IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS),
            new DefaultThreadFactory("NettyServerWorker", true));

    final NettyServerHandler nettyServerHandler = new NettyServerHandler(getUrl(), this);
    channels = nettyServerHandler.getChannels();

    bootstrap.group(bossGroup, workerGroup)
            .channel(NioServerSocketChannel.class)
            .childOption(ChannelOption.TCP_NODELAY, Boolean.TRUE)
            .childOption(ChannelOption.SO_REUSEADDR, Boolean.TRUE)
            .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
            .childHandler(new ChannelInitializer<NioSocketChannel>() {
                @Override
                protected void initChannel(NioSocketChannel ch) throws Exception {
                    NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
                    ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                            .addLast("decoder", adapter.getDecoder())
                            .addLast("encoder", adapter.getEncoder())
                            .addLast("handler", nettyServerHandler);
                }
            });
    // bind
    ChannelFuture channelFuture = bootstrap.bind(getBindAddress());
    channelFuture.syncUninterruptibly();
    channel = channelFuture.channel();

}
```
上面一段代码就是我们熟悉netty服务，我们这次重点来研究一下NettyCodecAdapter里面的Decoder和NettyServerHandler

当一个请求发过来，netty的处理链是NettyCodecAdapter.getDecoder() -> NettyServerHandler

首先来研究一下NettyCodecAdapter.getDecoder()的逻辑，NettyCodecAdapter.getDecoder()对应的类为
com.alibaba.dubbo.remoting.transport.netty4.NettyCodecAdapter.InternalDecoder，它继承自netty的ByteToMessageDecoder

> 注意，此段代码来自netty4，具体版本为4.1.39.Final

当有请求到来时，netty会触发通道读取事件，最终会调用到InternalDecoder的channelRead方法

```
public void io.netty.handler.codec.ByteToMessageDecoder#channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    if (msg instanceof ByteBuf) {
        //CodecOutputList继承自AbstractList，所以它是个list集合
        CodecOutputList out = CodecOutputList.newInstance();
        try {
            ByteBuf data = (ByteBuf) msg;
            //cumulation等于null就意味着从上次请求以来，没有缓存的数据（第一次请求或者上次请求将数据完全使用掉了）
            first = cumulation == null;
            if (first) {
                cumulation = data;
            } else {
                //检查空间是否够用，如果不够用会进行扩容
                //具体扩容逻辑请查看标号(*1*)
                cumulation = cumulator.cumulate(ctx.alloc(), cumulation, data);
            }
            //调用解码方法
            callDecode(ctx, cumulation, out);
        } catch (DecoderException e) {
            throw e;
        } catch (Exception e) {
            throw new DecoderException(e);
        } finally {
            if (cumulation != null && !cumulation.isReadable()) {
                numReads = 0;
                cumulation.release();
                cumulation = null;
            } else if (++ numReads >= discardAfterReads) {
                // We did enough reads already try to discard some bytes so we not risk to see a OOME.
                // See https://github.com/netty/netty/issues/4275
                numReads = 0;
                discardSomeReadBytes();
            }

            int size = out.size();
            firedChannelRead |= out.insertSinceRecycled();
            fireChannelRead(ctx, out, size);
            out.recycle();
        }
    } else {
        ctx.fireChannelRead(msg);
    }
}


//(*1*)
public static final Cumulator MERGE_CUMULATOR = new Cumulator() {
    @Override
    public ByteBuf cumulate(ByteBufAllocator alloc, ByteBuf cumulation, ByteBuf in) {
        try {
            final ByteBuf buffer;
            //如果想要容量足够存放新的内容，那么需要满足以下条件
            //cumulation.maxCapacity() - cumulation.writerIndex() > in.readableBytes() 推导出
            //cumulation.maxCapacity()  - cumulation.writerIndex() - in.readableBytes() > 0 推导出
            //cumulation.maxCapacity()  - in.readableBytes() > cumulation.writerIndex()
            //所以如果出现不满足的情况，那就是容量不够用了，需要扩容
            //或者这个ByteBuf属于只读的（我们需要能写入数据的ByteBuf）
            //或者引用计数大于1，比如调用了retain，slice等方法，这个时候我们不应该改变这个buff
            //导致使用它的地方发生一些问题
            if (cumulation.writerIndex() > cumulation.maxCapacity() - in.readableBytes()
                || cumulation.refCnt() > 1 || cumulation.isReadOnly()) {
                // Expand cumulation (by replace it) when either there is not more room in the buffer
                // or if the refCnt is greater then 1 which may happen when the user use slice().retain() or
                // duplicate().retain() or if its read-only.
                //
                // See:
                // - https://github.com/netty/netty/issues/2327
                // - https://github.com/netty/netty/issues/1764
                //扩容，扩容逻辑请查看（*2*）
                buffer = expandCumulation(alloc, cumulation, in.readableBytes());
            } else {
                buffer = cumulation;
            }
            buffer.writeBytes(in);
            return buffer;
        } finally {
            // We must release in in all cases as otherwise it may produce a leak if writeBytes(...) throw
            // for whatever release (for example because of OutOfMemoryError)
            in.release();
        }
    }
};

//（*2*）
static ByteBuf expandCumulation(ByteBufAllocator alloc, ByteBuf cumulation, int readable) {
    ByteBuf oldCumulation = cumulation;
    //申请一个容量为oldCumulation.readableBytes() + readable的ByteBuf
    cumulation = alloc.buffer(oldCumulation.readableBytes() + readable);
    //写入老数据
    cumulation.writeBytes(oldCumulation);
    //释放老的ByteBuf，避免内存泄漏
    oldCumulation.release();
    return cumulation;
}
```
调用解码方法

```
protected void io.netty.handler.codec.ByteToMessageDecoder#callDecode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
    try {
        //只要还存在可度数据，循环
        while (in.isReadable()) {
            //是否存在解码后的数据
            int outSize = out.size();
            //如果存在，触发通道的read事件，交由后面的通道处理器处理
            if (outSize > 0) {
                fireChannelRead(ctx, out, outSize);
                out.clear();

                // Check if this handler was removed before continuing with decoding.
                // If it was removed, it is not safe to continue to operate on the buffer.
                //
                // See:
                // - https://github.com/netty/netty/issues/4635
                //检查当前通道处理的上下文是否被移除，如果被移除就不再进行编码处理，退出后交由下一个通道处理上下文处理
                if (ctx.isRemoved()) {
                    break;
                }
                outSize = 0;
            }
            //可读字节数
            int oldInputLength = in.readableBytes();
            //具体的处理逻辑
            decodeRemovalReentryProtection(ctx, in, out);

            // Check if this handler was removed before continuing the loop.
            // If it was removed, it is not safe to continue to operate on the buffer.
            //
            // See https://github.com/netty/netty/issues/1664
            if (ctx.isRemoved()) {
                break;
            }
            //如果没有编码好的数据（可能所需的数据不够用）
            if (outSize == out.size()) {
                //如果可读的数据还是和原来一样，可以任务压根就没有进行数据解码逻辑，那么就直接退出
                if (oldInputLength == in.readableBytes()) {
                    break;
                } else {
                    continue;
                }
            }
            //如果数据进行了解码，也就是out中有值，无论是瞎add进去的，还是通过不影响byteBuf读取下标的方式获取的数据编码，在byteBuf
            //的可读数据未发生改变的情况下，抛错
            if (oldInputLength == in.readableBytes()) {
                throw new DecoderException(
                        StringUtil.simpleClassName(getClass()) +
                                ".decode() did not read anything but decoded a message.");
            }
            //是否只进行单此编码，默认false
            if (isSingleDecode()) {
                break;
            }
        }
    } catch (DecoderException e) {
        throw e;
    } catch (Exception cause) {
        throw new DecoderException(cause);
    }
}
```

> decodeRemovalReentryProtection

```
final void io.netty.handler.codec.ByteToMessageDecoder#decodeRemovalReentryProtection(ChannelHandlerContext ctx, ByteBuf in, List<Object> out)
            throws Exception {
    //编码状态修改为调用子类编码
    decodeState = STATE_CALLING_CHILD_DECODE;
    try {
        //交由子类处理
        decode(ctx, in, out);
    } finally {
        //如果子类修改编码状态为移除数据
        boolean removePending = decodeState == STATE_HANDLER_REMOVED_PENDING;
        //重新设置为初始化状态
        decodeState = STATE_INIT;
        if (removePending) {
            //(*1*)
            handlerRemoved(ctx);
        }
    }
}

//(*1*)
public final void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
    
    if (decodeState == STATE_CALLING_CHILD_DECODE) {
        decodeState = STATE_HANDLER_REMOVED_PENDING;
        return;
    }
    ByteBuf buf = cumulation;
    if (buf != null) {
        // Directly set this to null so we are sure we not access it in any other method here anymore.
        //清空数据，释放内存
        cumulation = null;
        numReads = 0;
        int readable = buf.readableBytes();
        if (readable > 0) {
            //如果存在可读数据，触发通道read事件
            ByteBuf bytes = buf.readBytes(readable);
            //释放引用
            buf.release();
            ctx.fireChannelRead(bytes);
            //触发读取完毕事件
            ctx.fireChannelReadComplete();
        } else {
            //释放引用
            buf.release();
        }
    }
    //空方法
    handlerRemoved0(ctx);
}
```

调用子类进行解码后才开始真正的进入到了dubbo的InternalDecoder.decode代码

```
protected void com.alibaba.dubbo.remoting.transport.netty4.NettyCodecAdapter.InternalDecoder.decode(ChannelHandlerContext ctx, ByteBuf input, List<Object> out) throws Exception {
    //对netty传递过来的数据进行包装备份
    //基本上可以认为这是一个适配器模式的运用
    ChannelBuffer message = new NettyBackedChannelBuffer(input);
    //创建NettyChannel，维护这netty的channel与服务提供url，服务提供通道处理
    NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);

    Object msg;
    //记录开始读取下标
    int saveReaderIndex;

    try {
        // decode object.
        do {
            saveReaderIndex = message.readerIndex();
            try {
                //调用DubboCountCodec处理
                msg = codec.decode(channel, message);
            } catch (IOException e) {
                throw e;
            }
            if (msg == Codec2.DecodeResult.NEED_MORE_INPUT) {
                //如果需要更多的数据，恢复原来的读取位置
                message.readerIndex(saveReaderIndex);
                break;
            } else {
                //is it possible to go here ?
                if (saveReaderIndex == message.readerIndex()) {
                    throw new IOException("Decode without read data.");
                }
                //添加经过解码的数据
                if (msg != null) {
                    out.add(msg);
                }
            }
        } while (message.readable());
    } finally {
        //如果这个通道被关闭了，那么从缓存中移除
        NettyChannel.removeChannelIfDisconnected(ctx.channel());
    }
}
```
> DubboCountCodec#decode

```
public Object com.alibaba.dubbo.rpc.protocol.dubbo.DubboCountCodec#decode(Channel channel, ChannelBuffer buffer) throws IOException {
    int save = buffer.readerIndex();
    //创建多消息集合，这个MultiMessage对象内部维护一个List集合，自身实现了Iterable接口
    MultiMessage result = MultiMessage.create();
    do {
        //调用DubboCodec
        Object obj = codec.decode(channel, buffer);
        //需要更多的数据
        if (Codec2.DecodeResult.NEED_MORE_INPUT == obj) {
            //恢复读取开始下标
            buffer.readerIndex(save);
            break;
        } else {
            //记录解码后的数据
            result.addMessage(obj);
            //给结果类型，如果是Request（请求），那么给它设置读取的字节码数作为附加参数
            //如果是Response，那么给它设置输出自己数作为附加参数
            logMessageLength(obj, buffer.readerIndex() - save);
            //更新读取开始下标
            save = buffer.readerIndex();
        }
    } while (true);
    //如果为空，表示需要更多的数据
    if (result.isEmpty()) {
        return Codec2.DecodeResult.NEED_MORE_INPUT;
    }
    //返回解码后的数据
    if (result.size() == 1) {
        return result.get(0);
    }
    return result;
}
```
> DubboCodec解码

在继续分析以下代码之前，先给大家透露一下dubbo请求协议，以便好理解
协议头有16个字节（0 ~ 127）大小

- 0~15位：魔数  0xdabb

- 16位：是否为Request,1为request，0为response

- 17位：是否为twoWay，从源代码来看这个twoWay的意思好像表示是否需要响应的意思（可能理解的有误哈）

- 18位：是否是事件，表示心跳事件，可能当前的请求或者响应来自心跳，不是接口消费请求

- 19~23位：序列化id，用于标识当前使用的数据序列化方式是什么，比如hessian2

- 24~31位：状态，比如ok（20）

- 32~95位：请求的id，从后面的源码中，我发现它会以key为请求id -> DefaultFuture存储在com.alibaba.dubbo.remoting.exchange.support.DefaultFuture#FUTURES中
  而这个DefaultFuture类似于JDK的Future，调用其get方法会发生阻塞知道有结果处理完毕

- 96~127位：标识请求体或者响应体的数据长度

- 128及以后：数据

```
public Object com.alibaba.dubbo.remoting.exchange.codec.ExchangeCodec#decode(Channel channel, ChannelBuffer buffer) throws IOException {
    //可读字节数
    int readable = buffer.readableBytes();
    //HEADER_LENGTH = 16, 在dubbo协议中，使用的是hessian2作为序列化器
    //这里的16字节是dubbo的协议头
    byte[] header = new byte[Math.min(readable, HEADER_LENGTH)];
    //将buffer中的数据读取到header字节数组中
    buffer.readBytes(header);
    return decode(channel, buffer, readable, header);
}
                                        |
                                        V
protected Object decode(Channel channel, ChannelBuffer buffer, int readable, byte[] header) throws IOException {
    // check magic number.
    //检查魔数，魔数占两个字节
    //如果当前的前两个字节不是魔数，会进入以下分支
    if (readable > 0 && header[0] != MAGIC_HIGH
            || readable > 1 && header[1] != MAGIC_LOW) {
        int length = header.length;
        //如果还有可读取数据，这里将会把buffer所有数据都读取到header数组中
        if (header.length < readable) {
            //扩容header数组为readable大小，并把header原来的数据拷贝到新的字节数组中
            header = Bytes.copyOf(header, readable);
            //将剩余的数据都写入到header中
            buffer.readBytes(header, length, readable - length);
        }
        //能够进入这个分支，说明第0个位置肯定不是魔数，所以从1开始
        for (int i = 1; i < header.length - 1; i++) {
            //循环寻找魔数开始的地方
            if (header[i] == MAGIC_HIGH && header[i + 1] == MAGIC_LOW) {
                //将读下标设置到魔数开始的位置
                //这里为什么要用buffer.readerIndex() - header.length呢？因为这个i是以header为基准
                //的下标并不是以buffer，所以需要让这个i加上（buffer.readerIndex() - header.length）的差值
                buffer.readerIndex(buffer.readerIndex() - header.length + i);
                //将魔数开始之前的数据用一个新的数组保存，然后赋值给header
                header = Bytes.copyOf(header, i);
                break;
            }
        }
        //调用父类，这个父类是TelnetCodec，用于处理telnet命令
        //比如输入Ctrl + C将会关闭通道，上下键会发送历史命令，感兴趣的自己去了解一下
        return super.decode(channel, buffer, readable, header);
    }
    // check length.
    //如果第一个字节是魔数开始的地方，如果当前可读的值少于16个字节，那么返回需要更多数据的标识
    if (readable < HEADER_LENGTH) {
        return DecodeResult.NEED_MORE_INPUT;
    }

    // get data length.
    //从第12个字节开始读取4个字节（int占四个字节），转换成int值
    //这个值标识了消息体携带的数据长度
    int len = Bytes.bytes2int(header, 12);
    //检查负载，如果超过了设置的大小，抛出异常
    checkPayload(channel, len);
    //数据长度加上协议头长度
    int tt = len + HEADER_LENGTH;
    //如果可读数据不够，那么提示需要更多的数据
    if (readable < tt) {
        return DecodeResult.NEED_MORE_INPUT;
    }

    // limit input stream.
    //创建流，流的读取开始为buffer.readerIndex() = 17，结束位置为17 + len（数据长度）
    ChannelBufferInputStream is = new ChannelBufferInputStream(buffer, len);

    try {
        //解析请求体
        return decodeBody(channel, is, header);
    } finally {
        if (is.available() > 0) {
            try {
                if (logger.isWarnEnabled()) {
                    logger.warn("Skip input stream " + is.available());
                }
                StreamUtils.skipUnusedStream(is);
            } catch (IOException e) {
                logger.warn(e.getMessage(), e);
            }
        }
    }
}
```
解析请求体

```
protected Object com.alibaba.dubbo.rpc.protocol.dubbo.DubboCodec#decodeBody(Channel channel, InputStream is, byte[] header) throws IOException {
    //flag包含了是否request请求，是否需要响应，是否为心跳事件，序列化id的标识
    //proto就是用于标识是什么序列化协议
    byte flag = header[2], proto = (byte) (flag & SERIALIZATION_MASK);
    // get request id.
    //在第四个字节以后的8个字节（long占用8个字节）为请求id/响应id
    long id = Bytes.bytes2long(header, 4);
    //是否是request请求，如果不是那就是response进入分支
    if ((flag & FLAG_REQUEST) == 0) {
        // decode response.
        //创建Response对象
        Response res = new Response(id);
        //是否是心跳，如果是，设置事件为心跳
        if ((flag & FLAG_EVENT) != 0) {
            res.setEvent(Response.HEARTBEAT_EVENT);
        }
        // get status.
        //响应状态，比如20标识ok
        byte status = header[3];
        res.setStatus(status);
        try {
            //内部通过序列化id获取了对应的序列化协议，默认是hessian2
            ObjectInput in = CodecSupport.deserialize(channel.getUrl(), is, proto);
            //如果响应是ok的
            if (status == Response.OK) {
                Object data;
                //是心跳事件吗？
                if (res.isHeartbeat()) {
                    //内部直接in.readObject()
                    data = decodeHeartbeatData(channel, in);
                    //是其他事件吗？目前这个版本好像就只有心跳哦
                } else if (res.isEvent()) {
                    //内部的反序列化直接调用的in.readObject()
                    data = decodeEventData(channel, in);
                } else {
                    //用于封装响应结果集
                    DecodeableRpcResult result;
                    //是否在当前io线程中进行解码？默认为true
                    if (channel.getUrl().getParameter(
                            Constants.DECODE_IN_IO_THREAD_KEY,
                            Constants.DEFAULT_DECODE_IN_IO_THREAD)) {
                        result = new DecodeableRpcResult(channel, res, is,
                                (Invocation) getRequestData(id), proto);
                        //响应体反序列化
                        result.decode();
                    } else {
                        //这种留到后面多线程中解析
                        result = new DecodeableRpcResult(channel, res,
                                new UnsafeByteArrayInputStream(readMessageData(is)),
                                (Invocation) getRequestData(id), proto);
                    }
                    data = result;
                }
                //DecodeableRpcResult
                res.setResult(data);
            } else {
                //响应不成功的，读取错误消息
                res.setErrorMessage(in.readUTF());
            }
        } catch (Throwable t) {
            if (log.isWarnEnabled()) {
                log.warn("Decode response failed: " + t.getMessage(), t);
            }
            //CLIENT_ERROR = 90
            res.setStatus(Response.CLIENT_ERROR);
            //异常消息
            res.setErrorMessage(StringUtils.toString(t));
        }
        return res;
    } else {
        // decode request.
        //创建Request
        Request req = new Request(id);
        //设置dubbo协议版本
        req.setVersion(Version.getProtocolVersion());
        //是否需要响应
        req.setTwoWay((flag & FLAG_TWOWAY) != 0);
        //是否是心跳
        if ((flag & FLAG_EVENT) != 0) {
            req.setEvent(Request.HEARTBEAT_EVENT);
        }
        try {
            Object data;
            //发序列化，默认使用hessian2
            ObjectInput in = CodecSupport.deserialize(channel.getUrl(), is, proto);
            //心跳
            if (req.isHeartbeat()) {
                //in.readObject()
                data = decodeHeartbeatData(channel, in);
            } else if (req.isEvent()) {
                //in.readObject()
                data = decodeEventData(channel, in);
            } else {
                //构建调用上下文，这个东西解析出简要调用的接口，方法，参数等信息
                DecodeableRpcInvocation inv;
                //是否在当前线程中解析
                if (channel.getUrl().getParameter(
                        Constants.DECODE_IN_IO_THREAD_KEY,
                        Constants.DEFAULT_DECODE_IN_IO_THREAD)) {
                    inv = new DecodeableRpcInvocation(channel, req, is, proto);
                    //解析请求体
                    inv.decode();
                } else {
                    //在线程池中解析
                    inv = new DecodeableRpcInvocation(channel, req,
                            new UnsafeByteArrayInputStream(readMessageData(is)), proto);
                }
                data = inv;
            }
            req.setData(data);
        } catch (Throwable t) {
            if (log.isWarnEnabled()) {
                log.warn("Decode request failed: " + t.getMessage(), t);
            }
            // bad request
            req.setBroken(true);
            req.setData(t);
        }
        return req;
    }
}



```
解析响应体（比如我们调用了远程的某个接口，然后远程服务返回数据）

```
public Object DecodeableRpcResult#decode(Channel channel, InputStream input) throws IOException {
        ObjectInput in = CodecSupport.getSerialization(channel.getUrl(), serializationType)
                .deserialize(channel.getUrl(), input);
    //读取一个字节，这个字节是用来标识响应结果类型的
    byte flag = in.readByte();
    switch (flag) {
        //如果响应结果为null
        case DubboCodec.RESPONSE_NULL_VALUE:
            break;
        case DubboCodec.RESPONSE_VALUE:
            try {
                //解析方法的返回类型
                Type[] returnType = RpcUtils.getReturnTypes(invocation);
                //根据接口方法返回类型反序列化值
                setValue(returnType == null || returnType.length == 0 ? in.readObject() :
                        (returnType.length == 1 ? in.readObject((Class<?>) returnType[0])
                                : in.readObject((Class<?>) returnType[0], returnType[1])));
            } catch (ClassNotFoundException e) {
                throw new IOException(StringUtils.toString("Read response data failed.", e));
            }
            break;
            //如果调用远程接口出错了
        case DubboCodec.RESPONSE_WITH_EXCEPTION:
            try {
                Object obj = in.readObject();
                //如果这个obj不是一场类型，嘻嘻，报错
                if (obj instanceof Throwable == false)
                    throw new IOException("Response data error, expect Throwable, but get " + obj);
                    //记录异常
                setException((Throwable) obj);
            } catch (ClassNotFoundException e) {
                throw new IOException(StringUtils.toString("Read response data failed.", e));
            }
            break;
            //响应null值，但是附带了其他的附加参数的
        case DubboCodec.RESPONSE_NULL_VALUE_WITH_ATTACHMENTS:
            try {
                //直接反序列化附加参数即可
                setAttachments((Map<String, String>) in.readObject(Map.class));
            } catch (ClassNotFoundException e) {
                throw new IOException(StringUtils.toString("Read response data failed.", e));
            }
            break;
            //正常调用还带有附加参数的
        case DubboCodec.RESPONSE_VALUE_WITH_ATTACHMENTS:
            try {
                //解析调用方法的返回值类型
                Type[] returnType = RpcUtils.getReturnTypes(invocation);
                //发序列化返回值
                setValue(returnType == null || returnType.length == 0 ? in.readObject() :
                        (returnType.length == 1 ? in.readObject((Class<?>) returnType[0])
                                : in.readObject((Class<?>) returnType[0], returnType[1])));
                //反序列化附加参数
                setAttachments((Map<String, String>) in.readObject(Map.class));
            } catch (ClassNotFoundException e) {
                throw new IOException(StringUtils.toString("Read response data failed.", e));
            }
            break;
            //调用远程接口发生异常，但附加了参数的
        case DubboCodec.RESPONSE_WITH_EXCEPTION_WITH_ATTACHMENTS:
            try {
                Object obj = in.readObject();
                if (obj instanceof Throwable == false)
                    throw new IOException("Response data error, expect Throwable, but get " + obj);
                    //先反序列化异常
                setException((Throwable) obj);
                //然后反序列化附加参数
                setAttachments((Map<String, String>) in.readObject(Map.class));
            } catch (ClassNotFoundException e) {
                throw new IOException(StringUtils.toString("Read response data failed.", e));
            }
            break;
        default:
            throw new IOException("Unknown result flag, expect '0' '1' '2', get " + flag);
    }
    //清理
    if (in instanceof Cleanable) {
        ((Cleanable) in).cleanup();
    }
    return this;
}
```
从上面解析响应体的数据来看，它的响应体结构大概就是 flag(用于表示是否有结果，有没有异常之类的，占用一个字节)+返回的结果/异常/附加参数

下一节，我们继续分析如果解析请求体


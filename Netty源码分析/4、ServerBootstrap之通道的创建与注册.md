## 一、ServerBootstrap概览

在第一节的例子中有以下代码

```
public void bind(int port) {
		//NIO线程组，Reactor线程组，一个用于接受客户端的请求，另一个用于进行SocketChannel的操作
		EventLoopGroup bossGroup = new NioEventLoopGroup();
		
		EventLoopGroup workerGroup = new NioEventLoopGroup();
		
		try {
			//用于NIO服务端启动的辅助类
			ServerBootstrap b = new ServerBootstrap();
			
			b.group(bossGroup, workerGroup)
			  .channel(NioServerSocketChannel.class)
			  .option(ChannelOption.SO_BACKLOG, 1024)
			  .childHandler(new ChildChannelHandler());
			
			//绑定端口，同步等待成功，一直等待到绑定端口成功，返回一个ChannelFuture,类似JDK中的java.util.concurrent.Future
			//用于异步的通知会调
			ChannelFuture f = b.bind(port).sync();
			
			//进行阻塞，等待服务器链路关闭，就退出
			f.channel().closeFuture().sync();
		
		
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			
			//优雅退出，释放线程资源
			bossGroup.shutdownGracefully();
			workerGroup.shutdownGracefully();
			
		}
		
	}
```

首先来看下ServerBootstrap的类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/5d9fbb8a3752fc6983a6d138dc5048f3.png)

下面是AbstractBootstrap的抽象类定义


```
public abstract class AbstractBootstrap<B extends AbstractBootstrap<B, C>, C extends Channel> implements Cloneable {
    //线程组
    private volatile EventLoopGroup group;
    //socket地址
    private volatile SocketAddress localAddress;
    //这两个还不清楚，但是第一个可以猜出这是给通道设置属性的map
    //需要往后看才能知道，我们继续
    private final Map<ChannelOption<?>, Object> options = new LinkedHashMap<ChannelOption<?>, Object>();
    private final Map<AttributeKey<?>, Object> attrs = new LinkedHashMap<AttributeKey<?>, Object>();
    //通道处理
    private volatile ChannelHandler handler;
    。。。。。。
    
}
```
ServerBootstrap

```
public final class ServerBootstrap extends AbstractBootstrap<ServerBootstrap, ServerChannel> {

    private static final InternalLogger logger = InternalLoggerFactory.getInstance(ServerBootstrap.class);
    //通道工厂
    private volatile ServerChannelFactory<? extends ServerChannel> channelFactory;
    //子通道操作参数
    private final Map<ChannelOption<?>, Object> childOptions = new LinkedHashMap<ChannelOption<?>, Object>();
    private final Map<AttributeKey<?>, Object> childAttrs = new LinkedHashMap<AttributeKey<?>, Object>();
    //子事件循环组
    private volatile EventLoopGroup childGroup;
    //子通道处理器
    private volatile ChannelHandler childHandler;
    
    。。。。。。
    
}
```
此外除了类的成员参数的定义，方法请自行大致的浏览一遍

然后可以看到ServerBootstrap继承AbstractBootstrap时指定了对应的泛型类型，一个是ServerBootstrap，另一个是ServerChannel，ServerBootstrap就不说了，这个ServerChannel还是要看下的

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ea20599ff491ab44af6daa4d772e22a4.png)

与之相关的接口结合代码注释大致浏览一遍

io.netty.bootstrap.ServerBootstrap.ServerBootstrapAcceptor：这个类是ServerBootstrap的内部类，它继承了ChannelHandlerAdapter，ChannelHandlerAdapter又实现了ChannelHandler接口，如果大家都适用过netty的话，那么这个ChannelHandler接口就是一个观察者，它对通道的激活，读取，写等事件感兴趣

io.netty.bootstrap.ServerBootstrap.ServerBootstrapChannelFactory：这个也是ServerBootstrap的内部类，它实现了ServerChannelFactory接口，其泛型为ServerChannel的子类

```
T newChannel(EventLoop eventLoop, EventLoopGroup childGroup);
```
从接口上来看，它就是用于创建ServerChannel的工厂

## 二、ServerBootstrap的启动

下面开始进入细节代码

### 2.1 ServerBootstrap参数准备

```
ServerBootstrap b = new ServerBootstrap();
			
b.group(bossGroup, workerGroup)
  .channel(NioServerSocketChannel.class)
  .option(ChannelOption.SO_BACKLOG, 1024)
  .childHandler(new ChildChannelHandler());
```
ServerBootstrap的默认构造器没有做任何事情

> group方法

```
public ServerBootstrap ServerBootstrap.group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
        //调用父类的group方法，将parentGroup传递进去
        //(*1*)
        super.group(parentGroup);
        if (childGroup == null) {
            throw new NullPointerException("childGroup");
        }
        if (this.childGroup != null) {
            throw new IllegalStateException("childGroup set already");
        }
        //设置到成员变量中
        this.childGroup = childGroup;
        return this;
}

//(*1*)
public B AbstractBootstrap<B extends AbstractBootstrap<B, C>, C extends Channel>.group(EventLoopGroup group) {
        if (group == null) {
            throw new NullPointerException("group");
        }
        if (this.group != null) {
            throw new IllegalStateException("group set already");
        }
        //设置到成员变量中
        this.group = group;
        return (B) this;
}
```

> channel

```
//这个参数在例子中，我们设置的是NioServerSocketChannel
public ServerBootstrap ServerBootstrap.channel(Class<? extends ServerChannel> channelClass) {
        if (channelClass == null) {
            throw new NullPointerException("channelClass");
        }
        //ServerBootstrapChannelFactory在上面看过，它就是一个构件通道的工厂，很显然，这里就是构建NioServerSocketChannel的工厂
        //(*1*)
        return channelFactory(new ServerBootstrapChannelFactory<ServerChannel>(channelClass));
}

//(*1*)
public ServerBootstrap ServerBootstrap.channelFactory(ServerChannelFactory<? extends ServerChannel> channelFactory) {
        if (channelFactory == null) {
            throw new NullPointerException("channelFactory");
        }
        if (this.channelFactory != null) {
            throw new IllegalStateException("channelFactory set already");
        }
        //设置到成员变量中
        this.channelFactory = channelFactory;
        return this;
}
```

> option

```
//我们在例子中传递进来的是 ChannelOption.SO_BACKLOG 和 1024
//options  Map<ChannelOption<?>, Object>
public <T> B AbstractBootstrap<B extends AbstractBootstrap<B, C>, C extends Channel>.option(ChannelOption<T> option, T value) {
        if (option == null) {
            throw new NullPointerException("option");
        }
        if (value == null) {
            synchronized (options) {
                //如果值为空，移除之
                options.remove(option);
            }
        } else {
            synchronized (options) {
                //添加属性
                options.put(option, value);
            }
        }
        return (B) this;
}
```
> childHandler

```
//这个childHandler参数设置的是我们自己定义的一个通道处理ChildChannelHandler
public ServerBootstrap childHandler(ChannelHandler childHandler) {
        if (childHandler == null) {
            throw new NullPointerException("childHandler");
        }
        this.childHandler = childHandler;
        return this;
}
```
上面提到一个自定义的通道处理器，它的继承结构如下：

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/1f98e05a6b15c7052c05b35b4ab0fdf3.png)

在上面有提到ChannelHandler接口其实就是一个观察者，当发生通道事件的时候，它会对对应感兴趣的事件作出响应。但我们继承这个类有点特殊，它继承了一个叫ChannelInitializer的抽象类，这个类它有一个抽象方法

```
protected abstract void initChannel(C ch) throws Exception;
```
然后它实现了channelRegistered（即通道注册方法），具体做了什么先不管，但是我们能够知道它在通道发生注册事件的时候会被调用。

下面是ServerBootstrap属性设置部分的时序图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/240c522bae5cfab5c807a1b584e3dbc2.png)

下面是ServerBootstrap的对象状态关系图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/4331a129c8226415fbe3faa918bfc459.png)

### 2.2 端口绑定

好了，一切准备参数已经就绪，那么解析来就开始绑定端口，接收请求了

```
//绑定端口，同步等待成功，一直等待到绑定端口成功，返回一个ChannelFuture,类似JDK中的java.util.concurrent.Future
//用于异步的通知会调
ChannelFuture f = b.bind(port).sync();

//进行阻塞，等待服务器链路关闭，就退出
f.channel().closeFuture().sync();
```
> ServerBootstrap.bind

```
public ChannelFuture io.netty.bootstrap.AbstractBootstrap.bind(int inetPort) {
        return bind(new InetSocketAddress(inetPort));
}
                                |
                                V
public ChannelFuture io.netty.bootstrap.AbstractBootstrap.bind(SocketAddress localAddress) {
        validate();
        if (localAddress == null) {
            throw new NullPointerException("localAddress");
        }
        return doBind(localAddress);
}
                                |
                                V
private ChannelFuture io.netty.bootstrap.AbstractBootstrap.doBind(final SocketAddress localAddress) {
        //初始化和注册通道
        final ChannelFuture regFuture = initAndRegister();
        final Channel channel = regFuture.channel();
        if (regFuture.cause() != null) {
            return regFuture;
        }

        final ChannelPromise promise;
        if (regFuture.isDone()) {
            promise = channel.newPromise();
            //绑定端口
            doBind0(regFuture, channel, localAddress, promise);
        } else {
            // Registration future is almost always fulfilled already, but just in case it's not.
            promise = new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE);
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    doBind0(regFuture, channel, localAddress, promise);
                }
            });
        }

        return promise;
}
```

对于上面的io.netty.bootstrap.AbstractBootstrap.doBind方法，我们分段进行分析，第一段是
initAndRegister方法

```
final ChannelFuture initAndRegister() {
        Channel channel;
        try {
            //创建通道，嘿嘿，这里一看就知道使用的就是上面我们分析过的ServerBootstrapChannelFactory创建的NioServerSocketChannel通道
            //(*1*)
            channel = createChannel();
        } catch (Throwable t) {
            return VoidChannel.INSTANCE.newFailedFuture(t);
        }

        try {
            //(*3*)
            //初始化通道
            init(channel);
        } catch (Throwable t) {
            channel.unsafe().closeForcibly();
            return channel.newFailedFuture(t);
        }

        ChannelPromise regFuture = channel.newPromise();
        channel.unsafe().register(regFuture);
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }
        return regFuture;
    }
    
    //(*1*)
    Channel io.netty.bootstrap.ServerBootstrap.createChannel() {
        //group()方法获取的父事件loop组中的事件执行器，next方法目前是轮询获取事件执行器的方法
        EventLoop eventLoop = group().next();
        //(*2*)
        return channelFactory().newChannel(eventLoop, childGroup);
    }
    
    //(*2*)
    public T newChannel(EventLoop eventLoop, EventLoopGroup childGroup) {
            try {
                //我们在例子中指定的通道是NioServerSocketChannel
                Constructor<? extends T> constructor = clazz.getConstructor(EventLoop.class, EventLoopGroup.class);
                return constructor.newInstance(eventLoop, childGroup);
            } catch (Throwable t) {
                throw new ChannelException("Unable to create Channel from class " + clazz, t);
            }
    }
    
```
上面创建了一个NioServerSocketChannel，我们来研究一下它的类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ffc68dfb2a0ac9ec52e9c26d9521ecc8.png)

- AttributeMap：用于存储和获取属性
- Channel：netty提供的接口，可以获取与其关联的EventLoop，通道配置对象ChannelConfig，读取，写，连接，获取ChannelPipeline，通道排管，从名字上就可以看出它相当于作用于通道的拦截器
- ServerChannel：它就定义了一个接口方法，用于获取EventLoopGroup
- AbstractChannel：bind,connect等实现，但是它在实现方法中调用的是DefaultChannelPipeline对应的bind，connect方法
- AbstractNioChannel：针对nio通道做些处理，比如设置为非阻塞，内部定义NioUnsafe的子类，用于针对nio的连接，读取，写实现。
- AbstractNioMessageChannel：主要定义了一个内部类NioMessageUnsafe，这个内部类实现了读取方法，AbstractNioMessageChannel实现了写方法，另外还定义两个抽象方法

```
/**
 * Read messages into the given array and return the amount which was read.
 */
protected abstract int doReadMessages(List<Object> buf) throws Exception;

/**
 * Write a message to the underlying {@link java.nio.channels.Channel}.
 *
 * @return {@code true} if and only if the message has been written
 */
protected abstract boolean doWriteMessage(Object msg, ChannelOutboundBuffer in) throws Exception;
```
- AbstractNioMessageServerChannel：实现了childEventLoopGroup方法，用于获取子事件循环组

下面看看NioServerSocketChannel的创建

```
//eventLoop：是父事件执行组中其中一个成员，childGroup这个是
public NioServerSocketChannel(EventLoop eventLoop, EventLoopGroup childGroup) {
        //(*1*newSocket()) SelectionKey.OP_ACCEPT是JDK nio包下的注册到selector的接收事件
        //newSocket()这个方法创建了一个JDK提供的nio的socket通道
        super(null, eventLoop, childGroup, newSocket(), SelectionKey.OP_ACCEPT);
        //创建一个默认的DefaultServerSocketChannelConfig配置类，这个类用于存储channal的配置信息
        //可以通过它获取到配置通道的一些配置信息，可以通过它向channal中设置属性
        //(*2*)
        config = new DefaultServerSocketChannelConfig(this, javaChannel().socket());
}
                            |
                            V
protected AbstractNioMessageServerChannel(
            Channel parent, EventLoop eventLoop, EventLoopGroup childGroup, SelectableChannel ch, int readInterestOp) {
        super(parent, eventLoop, ch, readInterestOp);
        this.childGroup = childGroup;
}
                            |
                            V
protected AbstractNioMessageChannel(
            Channel parent, EventLoop eventLoop, SelectableChannel ch, int readInterestOp) {
        super(parent, eventLoop, ch, readInterestOp);
}
                            |
                            V
protected AbstractNioChannel(Channel parent, EventLoop eventLoop, SelectableChannel ch, int readInterestOp) {
        super(parent, eventLoop);
        this.ch = ch;
        this.readInterestOp = readInterestOp;
        try {
            //配置为非阻塞通道
            ch.configureBlocking(false);
        } catch (IOException e) {
            try {
                ch.close();
            } catch (IOException e2) {
                if (logger.isWarnEnabled()) {
                    logger.warn(
                            "Failed to close a partially initialized socket.", e2);
                }
            }

            throw new ChannelException("Failed to enter non-blocking mode.", e);
        }
}
                            |
                            V
protected AbstractChannel(Channel parent, EventLoop eventLoop) {
        this.parent = parent;
        //确保eventLoop值不为null，确保eventLoop的类型为NioEventLoop
        this.eventLoop = validate(eventLoop);
        //NioMessageUnsafe
        unsafe = newUnsafe();
        //创建DefaultChannelPipeline
        pipeline = new DefaultChannelPipeline(this);
}

//(*1*)
private static ServerSocketChannel io.netty.channel.socket.nio.NioServerSocketChannel.newSocket() {
        try {
            //JDK提供的ServerSocketChannel
            return ServerSocketChannel.open();
        } catch (IOException e) {
            throw new ChannelException(
                    "Failed to open a server socket.", e);
        }
}

//(*2*)
//第二参数调用了javaChannel().socket()方法，实际上就是获取(*1*)中的ServerSocketChannel，然后获取其ServerSocket对象
public DefaultServerSocketChannelConfig(ServerSocketChannel channel, ServerSocket javaSocket) {
        //(*3*)
        super(channel);
        if (javaSocket == null) {
            throw new NullPointerException("javaSocket");
        }
        this.javaSocket = javaSocket;
}

 //(*3*)
 public DefaultChannelConfig(Channel channel) {
        if (channel == null) {
            throw new NullPointerException("channel");
        }
        this.channel = channel;

        if (channel instanceof ServerChannel || channel instanceof AbstractNioByteChannel) {
            // Server channels: Accept as many incoming connections as possible.
            // NIO byte channels: Implemented to reduce unnecessary system calls even if it's > 1.
            //                    See https://github.com/netty/netty/issues/2079
            // TODO: Add some property to ChannelMetadata so we can remove the ugly instanceof
            maxMessagesPerRead = 16;
        } else {
            maxMessagesPerRead = 1;
        }
}

```


初始化通道

```
void io.netty.bootstrap.ServerBootstrap.init(Channel channel) throws Exception {
        //获取通道操作，之前我们只设置了ChannelOption.SO_BACKLOG, 1024
        final Map<ChannelOption<?>, Object> options = options();
        synchronized (options) {
            //从可知创建是DefaultChannelConfig，然后将ServerBootstrap的options
            //转到与通道关联的DefaultChannelConfig上
            channel.config().setOptions(options);
        }
        //属性，我们在例子没有设置，所以这里也没有什么值
        final Map<AttributeKey<?>, Object> attrs = attrs();
        synchronized (attrs) {
            for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
                @SuppressWarnings("unchecked")
                AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
                //DefaultAttributeMap，将被包装成AttributeKey，与value组成的Attribute的对象
                channel.attr(key).set(e.getValue());
            }
        }
        //排管DefaultChannelPipeline
        ChannelPipeline p = channel.pipeline();
        //这里讲获取的处理器，是父处理器，我们之前设置的那个子处理器，所以这里为空
        if (handler() != null) {
            p.addLast(handler());
        }
        //当前子处理器
        final ChannelHandler currentChildHandler = childHandler;
        final Entry<ChannelOption<?>, Object>[] currentChildOptions;
        final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
        synchronized (childOptions) {
            currentChildOptions = childOptions.entrySet().toArray(newOptionArray(childOptions.size()));
        }
        synchronized (childAttrs) {
            currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(childAttrs.size()));
        }
        //向当前通道的排管中添加ChannelInitializer，在初始化通道的时候这个通道的排管中会添加一个ServerBootstrapAcceptor的
        //通道处理器，并将当前在ServerBootstrap设置的子处理器，操作参数，子属性都设置到这个ServerBootstrapAcceptor中
        //可以猜测，当这个通道接收每个请求时，将把currentChildHandler，currentChildOptions，currentChildAttrs设置到子通道中
        p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(Channel ch) throws Exception {
                ch.pipeline().addLast(new ServerBootstrapAcceptor(currentChildHandler, currentChildOptions,
                        currentChildAttrs));
            }
        });
}
```
上面的初始化通道代码中，netty给这个父通道（用于接口请求的ServerSocketChannal）添加了一个ServerBootstrapAcceptor，这是ServerBootstrap的内部类，它是一个通道处理器，它只读一个事件感兴趣，那就是读取事件

```
public void ServerBootstrapAcceptor.channelRead(ChannelHandlerContext ctx, Object msg) {
            //子通道，也就是ServerSocketChannal接收到的请求而打开的一个通道
            Channel child = (Channel) msg;
            //将我们自定义的子通道处理器设置进去
            child.pipeline().addLast(childHandler);
            //设置操作参数
            for (Entry<ChannelOption<?>, Object> e: childOptions) {
                try {
                    if (!child.config().setOption((ChannelOption<Object>) e.getKey(), e.getValue())) {
                        logger.warn("Unknown channel option: " + e);
                    }
                } catch (Throwable t) {
                    logger.warn("Failed to set a channel option: " + child, t);
                }
            }
            //设置属性
            for (Entry<AttributeKey<?>, Object> e: childAttrs) {
                child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
            }
            //注册通道到对应的执行器的selector中
            child.unsafe().register(child.newPromise());
}
```
ServerBootstrapAcceptor.channelRead点到为止，因为这不是我们的重点，后面发生通道read事件的时候，我们自然会看到这段代码

稍微总结下，上面将boss执行器组中某个事件执行器和子事件执行组作为参数构造了NioServerSocketChannel，然后将我们在ServerBootstrap设置的通道操作参数和属性转设到NioServerSocketChannel关联的DefaultChannelConfig中，添加通道处理器，如果有设置与NioServerSocketChannel相关的通道就添加，如果没有就不添加，最后添加ServerBootstrapAcceptor到父通道的排管中，将我们在ServerBootstrap设置的子通道处理器和子操作参数以及子属性设置到ServerBootstrapAcceptor中，它将在父通道发生读事件时，将这些参数设置到子通道中。

接下来开始接口的绑定，让我回到AbstractBootstrap的initAndRegister方法

```
final ChannelFuture AbstractBootstrap.initAndRegister() {
        Channel channel;
        try {
            //创建通道
            channel = createChannel();
        } catch (Throwable t) {
            return VoidChannel.INSTANCE.newFailedFuture(t);
        }

        try {
            //初始化通道
            init(channel);
        } catch (Throwable t) {
            channel.unsafe().closeForcibly();
            return channel.newFailedFuture(t);
        }
        //创建通道相关的Future，其实例为DefaultChannelPromise
        //当前通道将被作为参数与DefaultChannelPromise发生被动关联
        ChannelPromise regFuture = channel.newPromise();
        //注册当前通道到对应的事件循环执行器中，其刚兴趣的事件为接收事件
        //这里的unsafe在前面分析过，它是NioMessageUnsafe的实例
        //(*1*)
        channel.unsafe().register(regFuture);
        //如果发生异常
        if (regFuture.cause() != null) {
            //但是却注册成功
            if (channel.isRegistered()) {
                //关闭通道
                channel.close();
            } else {
                //强制关闭
                channel.unsafe().closeForcibly();
            }
        }

        return regFuture;
    }
    
//(*1*)
public final void io.netty.channel.AbstractChannel.AbstractUnsafe.register(final ChannelPromise promise) {
        //判断当前执行的线程是否是当前通道的事件循环执行器的里的线程，如果是直接注册，如果不是，异步提交到事件循环执行器中，让
        //执行器内部的线程池执行
        if (eventLoop.inEventLoop()) {
            register0(promise);
        } else {
            try {
                eventLoop.execute(new Runnable() {
                    @Override
                    public void run() {
                        //注册
                        //(*2*)
                        register0(promise);
                    }
                });
            } catch (Throwable t) {
                logger.warn(
                        "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                        AbstractChannel.this, t);
                //强制关闭，就是调用JDK对应的通道的关闭方法
                closeForcibly();
                //正常的通道，将会发生通知事件，但是closeFuture将直接抛错错误
                closeFuture.setClosed();
                //设置导致失败的异常
                promise.setFailure(t);
            }
        }
}

//(*2*)
private void io.netty.channel.AbstractChannel.AbstractUnsafe.register0(ChannelPromise promise) {
            try {
                // check if the channel is still open as it could be closed in the mean time when the register
                // call was outside of the eventLoop
                //确保ChannelPromise关联的通道没有被关闭，如果被关闭，那么就直接方法，在前面我知道netty会判断当前执行线程是否是事件循环执行器中的线程池的
                //的线程，如果不是将会进行异步提交到事件循环执行器中执行，那么就可能会出现通道被外部线程关闭的可能性
                if (!ensureOpen(promise)) {
                    return;
                }
                //(*3*)
                //do注册
                doRegister();
                //标记通道注册成功
                registered = true;
                //设置成功标识，其实就是给Future的result值，设置null的时候，netty自动给result赋值SUCCESS（具体的值无需关心，一个表示成功的Signal对象）
                //并将存在的阻塞线程唤醒，最后还会通知future监听器
                promise.setSuccess();
                //触发通道注册事件，通知通道处理器
                pipeline.fireChannelRegistered();
                //是否被激活，如果被激活将触发通道激活事件，通知通道处理器
                //这里所谓的激活就是调用JDK的通道的isBind方法，判断通道是否已经绑定到执行的端口
                //因为netty的注册方法是异步的，所以在注册的时候，通道可能已经被其他线程绑定了。
                if (isActive()) {
                    pipeline.fireChannelActive();
                }
            } catch (Throwable t) {
                // Close the channel directly to avoid FD leak.
                closeForcibly();
                closeFuture.setClosed();
                if (!promise.tryFailure(t)) {
                    logger.warn(
                            "Tried to fail the registration promise, but it is complete already. " +
                                    "Swallowing the cause of the registration failure:", t);
                }
            }
}

//(*3*)
protected void io.netty.channel.nio.AbstractNioChannel.doRegister() throws Exception {
        boolean selected = false;
        for (;;) {
            try {
                //注册通道
                //第一个参数就是jdk的多路复用器，第二参数表示感兴趣的事件，这里是0，后面会通过逻辑或的方式注册感兴趣的事件，最后一个
                //参数表示附件，这里的附件是netty的通道
                selectionKey = javaChannel().register(eventLoop().selector, 0, this);
                return;
            } catch (CancelledKeyException e) {
                //如果没有注册成功
                if (!selected) {
                    // Force the Selector to select now as the "canceled" SelectionKey may still be
                    // cached and not removed because no Select.select(..) operation was called yet.
                    //当前通道可能以前注册过，但是被取消了，还留在缓存中，那么通过selectNow方法来彻底取消掉的目的
                    //然后重新注册，selected被标记为true，也就是说一个通道进行注册只有两次的机会
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

回到doBind方法中

```
private ChannelFuture io.netty.bootstrap.AbstractBootstrap.doBind(final SocketAddress localAddress) {
        //初始化和注册，返回通道注册结果
        final ChannelFuture regFuture = initAndRegister();
        //获取通道
        final Channel channel = regFuture.channel();
        //如果注册时发生了异常，直接返回这个注册future
        if (regFuture.cause() != null) {
            return regFuture;
        }

        final ChannelPromise promise;
        //没有错误，并且已经注册完毕
        if (regFuture.isDone()) {
            //创建一个新的ChannelPromise，用于记录绑定的结果
            promise = channel.newPromise();
            doBind0(regFuture, channel, localAddress, promise);
        } else {
            // Registration future is almost always fulfilled already, but just in case it's not.
            //如果还没有注册成功，也就是还在注册中，那么直接创建一个DefaultChannelPromise，设置一个全局的事件执行器实例
            promise = new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE);
            //注册通道结果监听器
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    doBind0(regFuture, channel, localAddress, promise);
                }
            });
        }

        return promise;
}

```

注册

```
private static void io.netty.bootstrap.AbstractBootstrap#doBind0(
            final ChannelFuture regFuture, final Channel channel,
            final SocketAddress localAddress, final ChannelPromise promise) {

        // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
        // the pipeline in its channelRegistered() implementation.
        //又是通过事件执行器里的线程池进行异步执行，对了，我们现在创建的异步线程池内部使用的线程池是一个为每个任务都创建线程的执行的线程池
        //执行完线程就会消失，事件执行器任务的提交，我们稍后再分析，我们先看它是怎么进行绑定的
        channel.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                if (regFuture.isSuccess()) {
                    //(*1*)
                    //绑定，并且给这个通道注册通道发生错误时进行关闭通道的监听器
                    channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
                } else {
                    //表示失败
                    promise.setFailure(regFuture.cause());
                }
            }
        });
}

//(*1*)
public ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
        //又看到这个排管了，看来我们又必要研究一下这个排管了
        return pipeline.bind(localAddress, promise);
}
```
排管的概览介绍和绑定操作放在下一节分析，有必要的说明的是在通道绑定成功前，在例子中调用了ChannelFuture的sync方法，用于阻塞主线程。

```
public ChannelPromise io.netty.channel.DefaultChannelPromise#sync() throws InterruptedException {
        super.sync();
        return this;
}

public Promise<V> io.netty.util.concurrent.DefaultPromise#sync() throws InterruptedException {
        //(*1*)
        await();
        //(*2*)
        rethrowIfFailed();
        return this;
}

//(*1*)
public Promise<V> io.netty.util.concurrent.DefaultPromise#await() throws InterruptedException {
        //是否已经结束，表示结果已经返回
        //判断条件是result != null && result != UNCANCELLABLE;
        if (isDone()) {
            return this;
        }
        //如果线程被中断，判断中断异常
        if (Thread.interrupted()) {
            throw new InterruptedException(toString());
        }

        synchronized (this) {
            while (!isDone()) {
                //检查死锁，也就是判断当前线程是否是DefaultPromise所关联线程池中的线程
                //如果是，那么就表示发生了死锁
                checkDeadLock();
                //waiters自增，用于计数有多少被阻塞的线程
                incWaiters();
                try {
                    //调用Object的wait方法进行阻塞
                    wait();
                } finally {
                    //被唤醒后waiters自减
                    decWaiters();
                }
            }
        }
        return this;
}
```

绑定与排管将在下一节分析


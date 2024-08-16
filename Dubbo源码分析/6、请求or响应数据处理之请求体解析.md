上一小节我们分析了响应头的解析，现在我们继续
分析
解析请求体

```
public Object com.alibaba.dubbo.rpc.protocol.dubbo.DecodeableRpcInvocation#decode(Channel channel, InputStream input) throws IOException {
    //反序列化，默认使用hessian2
    ObjectInput in = CodecSupport.getSerialization(channel.getUrl(), serializationType)
            .deserialize(channel.getUrl(), input);
    //读取版本
    String dubboVersion = in.readUTF();
    request.setVersion(dubboVersion);
    //设置为附加参数
    setAttachment(Constants.DUBBO_VERSION_KEY, dubboVersion);
    //接口名
    setAttachment(Constants.PATH_KEY, in.readUTF());
    //接口版本
    setAttachment(Constants.VERSION_KEY, in.readUTF());
    //方法名
    setMethodName(in.readUTF());
    try {
        //参数
        Object[] args;
        //参数类型
        Class<?>[] pts;
        //获取字节码描述符，比如String,User,int =》 Ljava/lang/String;Lcom/test/User;I
        String desc = in.readUTF();
        if (desc.length() == 0) {
            //如果没有参数
            pts = DubboCodec.EMPTY_CLASS_ARRAY;
            args = DubboCodec.EMPTY_OBJECT_ARRAY;
        } else {
            //解析描述符，获取参数数组，假设接口参数是String,user,int
            //那么这个数组就是new Class[]{String.class, User.class, int.class}
            pts = ReflectUtils.desc2classArray(desc);
            args = new Object[pts.length];
            for (int i = 0; i < args.length; i++) {
                try {
                    //紧接着的就是参数
                    args[i] = in.readObject(pts[i]);
                } catch (Exception e) {
                    if (log.isWarnEnabled()) {
                        log.warn("Decode argument failed: " + e.getMessage(), e);
                    }
                }
            }
        }
        //记录参数类型
        setParameterTypes(pts);
        //解析附加参数
        Map<String, String> map = (Map<String, String>) in.readObject(Map.class);
        if (map != null && map.size() > 0) {
            Map<String, String> attachment = getAttachments();
            if (attachment == null) {
                attachment = new HashMap<String, String>();
            }
            attachment.putAll(map);
            setAttachments(attachment);
        }
        //decode argument ,may be callback
        //参数回调
        for (int i = 0; i < args.length; i++) {
            args[i] = decodeInvocationArgument(channel, this, pts, i, args[i]);
        }
        //设置参数
        setArguments(args);

    } catch (ClassNotFoundException e) {
        throw new IOException(StringUtils.toString("Read invocation data failed.", e));
    } finally {
        if (in instanceof Cleanable) {
            ((Cleanable) in).cleanup();
        }
    }
    return this;
}
```
上面的代码中，dubbo解析完参数后，还有一个比较复杂的动作，那就是进行参数的回调处理

```
//channel：dubbo封装的通道，url：提供者地址，clazz：当前参数类型，inv：当前请求的调用上下文，instid：参数位置，isRefer：是否提交回调服务
private static Object com.alibaba.dubbo.rpc.protocol.dubbo.CallbackServiceCodec#referOrdestroyCallbackService(Channel channel, URL url, Class<?> clazz, Invocation inv, int instid, boolean isRefer) {
    Object proxy = null;
    //构建调用者key => callback.service.proxy.channel实例的hacode.接口名.参数下标位置.invoker
    String invokerCacheKey = getServerSideCallbackInvokerCacheKey(channel, clazz.getName(), instid);
    //代理缓存key =》callback.service.proxy.channel实例的hacode.接口名.参数下标位置
    String proxyCacheKey = getServerSideCallbackServiceCacheKey(channel, clazz.getName(), instid);
    //从通道缓存获取已经代理的回调对象
    proxy = channel.getAttribute(proxyCacheKey);
    //callback.service.proxy.channel的hashcode.回调参数类型名.COUNT
    String countkey = getServerSideCountKey(channel, clazz.getName());
    if (isRefer) {
        //如果代理对象为空，那么重新生成一个
        if (proxy == null) {
            //构建回调地址callback://提供者的ip地址+端口/参数类型名?interface=参数类型名
            URL referurl = URL.valueOf("callback://" + url.getAddress() + "/" + clazz.getName() + "?" + Constants.INTERFACE_KEY + "=" + clazz.getName());
            //将提供者参数设置到referurl中并移除methods参数
            referurl = referurl.addParametersIfAbsent(url.getParameters()).removeParameter(Constants.METHODS_KEY);
            //判断回调参数是否超过上限，默认一个方法就一个回调参数
            if (!isInstancesOverLimit(channel, referurl, clazz.getName(), instid, true)) {
                @SuppressWarnings("rawtypes")
                //构建一个Invoker
                Invoker<?> invoker = new ChannelWrappedInvoker(clazz, channel, referurl, String.valueOf(instid));
                //对invoker进行代理，这个proxyFactory是生成的一个类，它的类名为ProxyFactory$Adaptive，由于默认的ProxyFactory上标注的@SPI默认名字
                //为javassist，所以最终调用的为JavassistProxyFactory
                proxy = proxyFactory.getProxy(invoker);
                //缓存代理对象
                channel.setAttribute(proxyCacheKey, proxy);
                //缓存Invoker
                channel.setAttribute(invokerCacheKey, invoker);
                //计数
                increaseInstanceCount(channel, countkey);

                //convert error fail fast .
                //ignore concurrent problem. 
                //channel.callback.invokers.key 记录历史invoker
                Set<Invoker<?>> callbackInvokers = (Set<Invoker<?>>) channel.getAttribute(Constants.CHANNEL_CALLBACK_KEY);
                if (callbackInvokers == null) {
                    //没有就创建一个
                    callbackInvokers = new ConcurrentHashSet<Invoker<?>>(1);
                    callbackInvokers.add(invoker);
                    //设值
                    channel.setAttribute(Constants.CHANNEL_CALLBACK_KEY, callbackInvokers);
                }
                logger.info("method " + inv.getMethodName() + " include a callback service :" + invoker.getUrl() + ", a proxy :" + invoker + " has been created.");
            }
        }
    } else {
        //如果refer不为true，并且存在回调代理
        if (proxy != null) {
            //从通道缓存中获取Invoker对象
            Invoker<?> invoker = (Invoker<?>) channel.getAttribute(invokerCacheKey);
            try {
                //获取invoker历史集合，移除最近使用的那一个invoker
                Set<Invoker<?>> callbackInvokers = (Set<Invoker<?>>) channel.getAttribute(Constants.CHANNEL_CALLBACK_KEY);
                if (callbackInvokers != null) {
                    callbackInvokers.remove(invoker);
                }
                //调用其销毁方法，由上面refer为true，proxy为null时的代码可知，它创建的是ChannelWrappedInvoker的实例，它的destroy方法为空方法
                invoker.destroy();
            } catch (Exception e) {
                logger.error(e.getMessage(), e);
            }
            // cancel refer, directly remove from the map
            //移除
            channel.removeAttribute(proxyCacheKey);
            channel.removeAttribute(invokerCacheKey);
            //计数减一
            decreaseInstanceCount(channel, countkey);
        }
    }
    return proxy;
}
```
我们继续来看下回调参数是怎么使用JavassistProxyFactory创建代理的

```
public <T> T AbstractProxyFactory#getProxy(Invoker<T> invoker) throws RpcException {
    return getProxy(invoker, false);
}
                                            |
                                            V
@Override
public <T> T AbstractProxyFactory#getProxy(Invoker<T> invoker, boolean generic) throws RpcException {
    Class<?>[] interfaces = null;
    //从参数中获取需要代理的接口
    String config = invoker.getUrl().getParameter("interfaces");
    if (config != null && config.length() > 0) {
        //逗号分割
        String[] types = Constants.COMMA_SPLIT_PATTERN.split(config);
        if (types != null && types.length > 0) {
            interfaces = new Class<?>[types.length + 2];
            //回调参数类型
            interfaces[0] = invoker.getInterface();
            //EchoService
            interfaces[1] = EchoService.class;
            for (int i = 0; i < types.length; i++) {
                interfaces[i + 1] = ReflectUtils.forName(types[i]);
            }
        }
    }
    if (interfaces == null) {
        interfaces = new Class<?>[]{invoker.getInterface(), EchoService.class};
    }
    //如果回调接口类型不是GenericService，并且又指定了要泛化调用，那么增加一个GenericService接口
    if (!invoker.getInterface().equals(GenericService.class) && generic) {
        int len = interfaces.length;
        Class<?>[] temp = interfaces;
        interfaces = new Class<?>[len + 1];
        System.arraycopy(temp, 0, interfaces, 0, len);
        interfaces[len] = GenericService.class;
    }
    //获取代理
    return getProxy(invoker, interfaces);
}
                                            |
                                            V
public <T> T JavassistProxyFactory#getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    //InvokerInvocationHandler实现了jdk的InvocationHandler接口，所以具体的操作逻辑就在这个方法里面
    return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
}
                                            |
                                            V
//以下为生成代理的具体逻辑
public static Proxy getProxy(ClassLoader cl, Class<?>... ics) {
    
    。。。。。。省略部分校验与从缓存获取代理的逻辑
    //代理计数器
    long id = PROXY_CLASS_COUNTER.getAndIncrement();
    String pkg = null;
    ClassGenerator ccp = null, ccm = null;
    try {
        //使用javassist创建一个类生成器
        ccp = ClassGenerator.newInstance(cl);
        //用于已经处理过的方法描述符
        Set<String> worked = new HashSet<String>();
        //结构方法对象
        List<Method> methods = new ArrayList<Method>();

        for (int i = 0; i < ics.length; i++) {
            //不能出现不同包的非public类
            if (!Modifier.isPublic(ics[i].getModifiers())) {
                String npkg = ics[i].getPackage().getName();
                if (pkg == null) {
                    pkg = npkg;
                } else {
                    if (!pkg.equals(npkg))
                        throw new IllegalArgumentException("non-public interfaces from different packages");
                }
            }
            //添加将要实现的接口
            ccp.addInterface(ics[i]);
            //循环接口方法
            for (Method method : ics[i].getMethods()) {
                //获取方法的描述符
                String desc = ReflectUtils.getDesc(method);
                //是否已经处理过了，处理过的跳过
                if (worked.contains(desc))
                    continue;
                //记录
                worked.add(desc);
                //方法所在位置下标
                int ix = methods.size();
                //方法返回值
                Class<?> rt = method.getReturnType();
                //方法参数
                Class<?>[] pts = method.getParameterTypes();
                //构建方法体java代码，主要是将方法参数存储到Object数组中，比如这个方法a(String a, Integer b, User user)
                //那么就可以构建一个三个参数的Object数组，Object args = new Obejct[]{a, b, user}
                StringBuilder code = new StringBuilder("Object[] args = new Object[").append(pts.length).append("];");
                for (int j = 0; j < pts.length; j++)
                    code.append(" args[").append(j).append("] = ($w)$").append(j + 1).append(";");
                code.append(" Object ret = handler.invoke(this, methods[" + ix + "], args);");
                //如果返回值为不是void，那么构建返回值
                if (!Void.TYPE.equals(rt))
                    code.append(" return ").append(asArgument(rt, "ret")).append(";");

                methods.add(method);
                //添加方法
                ccp.addMethod(method.getName(), method.getModifiers(), rt, pts, method.getExceptionTypes(), code.toString());
            }
        }

        if (pkg == null)
            //默认包名com.alibaba.dubbo.common.bytecode
            pkg = PACKAGE_NAME;

        // create ProxyInstance class.
        //代理类的类名，通常为com.alibaba.dubbo.common.bytecode.proxy.1(这个数组随着创建的代理对象原子递增)
        String pcn = pkg + ".proxy" + id;
        //设置类名
        ccp.setClassName(pcn);
        //添加成员变量，接口方法数组
        ccp.addField("public static java.lang.reflect.Method[] methods;");
        //添加实现了JDK的InvocationHandler的接口成员变量
        ccp.addField("private " + InvocationHandler.class.getName() + " handler;");
        //添加构造器
        ccp.addConstructor(Modifier.PUBLIC, new Class<?>[]{InvocationHandler.class}, new Class<?>[0], "handler=$1;");
        //添加默认构造器
        ccp.addDefaultConstructor();
        Class<?> clazz = ccp.toClass();
        //设置methods数组属性值
        clazz.getField("methods").set(null, methods.toArray(new Method[0]));

        // create Proxy class.
        //创建代理助手类
        String fcn = Proxy.class.getName() + id;
        ccm = ClassGenerator.newInstance(cl);
        ccm.setClassName(fcn);
        ccm.addDefaultConstructor();
        //设置继承的超类
        ccm.setSuperClass(Proxy.class);
        //添加方法
        ccm.addMethod("public Object newInstance(" + InvocationHandler.class.getName() + " h){ return new " + pcn + "($1); }");
        Class<?> pc = ccm.toClass();
        proxy = (Proxy) pc.newInstance();
    } catch (RuntimeException e) {
        throw e;
    } catch (Exception e) {
        throw new RuntimeException(e.getMessage(), e);
    } finally {
        // release ClassGenerator
        if (ccp != null)
            ccp.release();
        if (ccm != null)
            ccm.release();
        synchronized (cache) {
            if (proxy == null)
                cache.remove(key);
            else
                cache.put(key, new WeakReference<Proxy>(proxy));
            cache.notifyAll();
        }
    }
    return proxy;
}
```
为了便于理解，我们直接举个例子，假设我要为以下接口生成代理

```
public interface UserService {
	public void sayHello(String say, User user, int a);
}
```
生成的代理类

```
public class com.alibaba.dubbo.common.bytecode.proxy.1 implements UserService,EchoService {
    //接口方法对象
    public static java.lang.reflect.Method[] methods;
    //实现了JDK的InvocationHandler接口
    private java.lang.reflect.InvocationHandler handler;
    
    public com.alibaba.dubbo.common.bytecode.proxy.1() {
    }
    
    public com.alibaba.dubbo.common.bytecode.proxy.1(java.lang.reflect.InvocationHandler handler) {
        this.handler = handler;
    }
    
	public void sayHello(String say, User user, int a) {
	    Object[] args = new Object[3];
	    args[0] = say;
	    args[1] = user;
	    args[2] = a;
	    Object ret = handler.invoke(this, methods[0], args);
	    
	}
	
	public Object $echo(Object message) {
	    Object[] args = new Object[1];
	    args[0] = message;
	    Object ret = handler.invoke(this, methods[1], args);
	    return ret;
	}
}
```
代理助手类

```
public class com.alibaba.dubbo.common.bytecode.Proxy1 extends com.alibaba.dubbo.common.bytecode.Proxy {

    public com.alibaba.dubbo.common.bytecode.Proxy1() {
        
    }
    
    public Object newInstance(java.lang.reflect.InvocationHandler h) {
        return new com.alibaba.dubbo.common.bytecode.proxy.1(h);
    }
    
}
```
好了，现在我们来看下InvokerInvocationHandler的逻辑

```
public Object com.alibaba.dubbo.rpc.proxy.InvokerInvocationHandler#invoke(Object proxy, Method method, Object[] args) throws Throwable {
    //方法名
    String methodName = method.getName();
    //方法参数
    Class<?>[] parameterTypes = method.getParameterTypes();
    //如果方法是定义在Object中的，那么直接调用即可
    if (method.getDeclaringClass() == Object.class) {
        return method.invoke(invoker, args);
    }
    //toString方法，调用invoker的toString方法
    if ("toString".equals(methodName) && parameterTypes.length == 0) {
        return invoker.toString();
    }
    if ("hashCode".equals(methodName) && parameterTypes.length == 0) {
        return invoker.hashCode();
    }
    if ("equals".equals(methodName) && parameterTypes.length == 1) {
        return invoker.equals(args[0]);
    }
    //invoker的对象，我们在分析前面的回调参数代码中可以知道它是ChannelWrappedInvoker的示例，它继承于AbstractInvoker
    return invoker.invoke(new RpcInvocation(method, args)).recreate();
}
```
> ChannelWrappedInvoker

```
//Invocation 通常就是一种记录目标方法的一些信息，比如接口，方法，参数等，一般我把这种对象称为调用上下文
public Result com.alibaba.dubbo.rpc.protocol.AbstractInvoker#invoke(Invocation inv) throws RpcException {
    // if invoker is destroyed due to address refresh from registry, let's allow the current invoke to proceed
    //如果当前invoker已经被销毁，不如地址发生改变之类，弃用当前invoker
    if (destroyed.get()) {
        logger.warn("Invoker for service " + this + " on consumer " + NetUtils.getLocalHost() + " is destroyed, "
                + ", dubbo version is " + Version.getVersion() + ", this invoker should not be used any longer");
    }

    RpcInvocation invocation = (RpcInvocation) inv;
    invocation.setInvoker(this);
    //添加invoker的附加参数
    if (attachment != null && attachment.size() > 0) {
        invocation.addAttachmentsIfAbsent(attachment);
    }
    //从调用的本地线程上下文中获取附加参数
    Map<String, String> contextAttachments = RpcContext.getContext().getAttachments();
    if (contextAttachments != null && contextAttachments.size() != 0) {
        /**
         * invocation.addAttachmentsIfAbsent(context){@link RpcInvocation#addAttachmentsIfAbsent(Map)}should not be used here,
         * because the {@link RpcContext#setAttachment(String, String)} is passed in the Filter when the call is triggered
         * by the built-in retry mechanism of the Dubbo. The attachment to update RpcContext will no longer work, which is
         * a mistake in most cases (for example, through Filter to RpcContext output traceId and spanId and other information).
         */
        invocation.addAttachments(contextAttachments);
    }
    //检查当前方法是否需要异步执行，这种方法一般都是没有返回值的，就是让它跑下，相当于消费者的角色
    if (getUrl().getMethodParameter(invocation.getMethodName(), Constants.ASYNC_KEY, false)) {
        invocation.setAttachment(Constants.ASYNC_KEY, Boolean.TRUE.toString());
    }
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);

    try {
        //(*1*)
        return doInvoke(invocation);
    } catch (InvocationTargetException e) { // biz exception
        //异常处理
        Throwable te = e.getTargetException();
        if (te == null) {
            return new RpcResult(e);
        } else {
            if (te instanceof RpcException) {
                ((RpcException) te).setCode(RpcException.BIZ_EXCEPTION);
            }
            return new RpcResult(te);
        }
    } catch (RpcException e) {
        if (e.isBiz()) {
            return new RpcResult(e);
        } else {
            throw e;
        }
    } catch (Throwable e) {
        return new RpcResult(e);
    }
}


//(*1*)
protected Result com.alibaba.dubbo.rpc.protocol.dubbo.ChannelWrappedInvoker#doInvoke(Invocation invocation) throws Throwable {
    RpcInvocation inv = (RpcInvocation) invocation;
    // use interface's name as service path to export if it's not found on client side
    //设置接口路径，这里就是我们的回调参数类型
    inv.setAttachment(Constants.PATH_KEY, getInterface().getName());
    //这个回到参数下标
    inv.setAttachment(Constants.CALLBACK_SERVICE_KEY, serviceKey);

    try {
        //如果是异步调用
        if (getUrl().getMethodParameter(invocation.getMethodName(), Constants.ASYNC_KEY, false)) { // may have concurrency issue
            //发送需要回调的参数信息给消费者后就返回一个空的RpcResult
            currentClient.send(inv, getUrl().getMethodParameter(invocation.getMethodName(), Constants.SENT_KEY, false));
            return new RpcResult();
        }
        //不是异步的，获取超时时间
        int timeout = getUrl().getMethodParameter(invocation.getMethodName(), Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
        if (timeout > 0) {
            //同步调用，get()方法会发生阻塞
            return (Result) currentClient.request(inv, timeout).get();
        } else {
            //同步调用，get()方法会发生阻塞
            return (Result) currentClient.request(inv).get();
        }
    } catch (RpcException e) {
        throw e;
    } catch (TimeoutException e) {
        throw new RpcException(RpcException.TIMEOUT_EXCEPTION, e.getMessage(), e);
    } catch (RemotingException e) {
        throw new RpcException(RpcException.NETWORK_EXCEPTION, e.getMessage(), e);
    } catch (Throwable e) { // here is non-biz exception, wrap it.
        throw new RpcException(e.getMessage(), e);
    }
}

```
对于上面一段代码的同步调用，会做这么一个逻辑

```
public ResponseFuture com.alibaba.dubbo.remoting.exchange.support.header.HeaderExchangeChannel#request(Object request, int timeout) throws RemotingException {
    if (closed) {
        throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
    }
    // create request.
    Request req = new Request();
    req.setVersion(Version.getProtocolVersion());
    req.setTwoWay(true);
    req.setData(request);
    //创建DefaultFuture，这个future类似于JDK的Future，调用其get方法时会发生阻塞
    DefaultFuture future = new DefaultFuture(channel, req, timeout);
    try {
        channel.send(req);
    } catch (RemotingException e) {
        future.cancel();
        throw e;
    }
    return future;
}

```
下面看下这个DefaultFuture的构造器

```
public DefaultFuture(Channel channel, Request request, int timeout) {
    this.channel = channel;
    this.request = request;
    //请求的id
    this.id = request.getId();
    //超时时间
    this.timeout = timeout > 0 ? timeout : channel.getUrl().getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
    // put into waiting map.
    //全局的静态Map<请求id，DefaultFuture>
    FUTURES.put(id, this);
    //全局的静态Map<请求id, Channel>
    CHANNELS.put(id, channel);
}
```
com.alibaba.dubbo.remoting.exchange.support.DefaultFuture#FUTURES是一个全局静态map结合，请求id =》DefaultFuture，当远程服务处理完结果后会把这个id返回回来，然后从这里结合中获取DefaultFuture，最后把值解析设置进去，阻塞的线程被唤醒。

经过代码的分析可以发现方法的回调参数是远程服务器调用消费者来实现的，不过这也无可厚非，因为消费者才会创建具体的回调参数对象，提供者是没有这个对象的，当提供者的实现中使用到这个回调参数，那么就会回调消费者。好了既然已经明白了这个回调参数的意义，那么我就把长线拉回到解析请求体的代码中

构建好了request，netty触发了通道read事件，进入了以下方法

```
//ctx：当前通道处理器的上下文，msg：就是上面构建的dubbo的Request对象
public void com.alibaba.dubbo.remoting.transport.netty4.NettyServerHandler#channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    //构建NettyChannel，并存入缓存，ctx.channel() -》 NettyChannel
    //NettyChannel维护这netty的channel与url以及handler的关系
    //在第四小节分析服务导出的时候，我们知道这个handler被层层装饰过的
    NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);
    try {
        //调用dubbo自己通道接收处理
        handler.received(channel, msg);
    } finally {
        NettyChannel.removeChannelIfDisconnected(ctx.channel());
    }
}
```
对于这个handler，我们直接使用一张图来回顾一下它的装饰层级

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/985056ebaad2334ae144f5a1d758bc63.png)

```
public void com.alibaba.dubbo.remoting.transport.MultiMessageHandler#received(Channel channel, Object message) throws RemotingException {
    //很明显，我们传递的是一个Request对象
    //MultiMessage实现了Iterable接口，内部持有一个List集合
    if (message instanceof MultiMessage) {
        MultiMessage list = (MultiMessage) message;
        //循环处理消息
        for (Object obj : list) {
            handler.received(channel, obj);
        }
    } else {
        handler.received(channel, message);
    }
}
                                                        |
                                                        V
public void com.alibaba.dubbo.remoting.exchange.support.header.HeartbeatHandler#received(Channel channel, Object message) throws RemotingException {
    //更新通道的读取事件，便于心跳
    setReadTimestamp(channel);
    //是心跳请求吗？
    if (isHeartbeatRequest(message)) {
        //是的
        Request req = (Request) message;
        //需要回应吗？
        if (req.isTwoWay()) {
            //要的
            Response res = new Response(req.getId(), req.getVersion());
            res.setEvent(Response.HEARTBEAT_EVENT);
            //回应对方，您好啊！我还能跳
            channel.send(res);
            if (logger.isInfoEnabled()) {
                int heartbeat = channel.getUrl().getParameter(Constants.HEARTBEAT_KEY, 0);
                if (logger.isDebugEnabled()) {
                    logger.debug("Received heartbeat from remote channel " + channel.getRemoteAddress()
                            + ", cause: The channel has no data-transmission exceeds a heartbeat period"
                            + (heartbeat > 0 ? ": " + heartbeat + "ms" : ""));
                }
            }
        }
        return;
    }
    //是心跳响应吗？
    if (isHeartbeatResponse(message)) {
        //是的
        //因为已经更新通道的read时间，所以此处就打个debug日志即可
        if (logger.isDebugEnabled()) {
            logger.debug("Receive heartbeat response in thread " + Thread.currentThread().getName());
        }
        return;
    }
    handler.received(channel, message);
}
                                                        |
                                                        V
public void com.alibaba.dubbo.remoting.transport.dispatcher.all.AllChannelHandler#received(Channel channel, Object message) throws RemotingException {
    //获取线程池，这个线程池也是通过SPI获取的，默认是调用FixedThreadPool的getExecutor方法，对应的参数（线程个数等等）通过URL获取
    ExecutorService cexecutor = getExecutorService();
    try {
        //包装成Runnable，线程池调用
        cexecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
    } catch (Throwable t) {
        //TODO A temporary solution to the problem that the exception information can not be sent to the opposite end after the thread pool is full. Need a refactoring
        //fix The thread pool is full, refuses to call, does not return, and causes the consumer to wait for time out
    	if(message instanceof Request && t instanceof RejectedExecutionException){
    	    //如果是线程池满了，回复给对方，告诉对方失败原因
    		Request request = (Request)message;
    		if(request.isTwoWay()){
    			String msg = "Server side(" + url.getIp() + "," + url.getPort() + ") threadpool is exhausted ,detail msg:" + t.getMessage();
    			Response response = new Response(request.getId(), request.getVersion());
    			response.setStatus(Response.SERVER_THREADPOOL_EXHAUSTED_ERROR);
    			response.setErrorMessage(msg);
    			channel.send(response);
    			return;
    		}
    	}
        throw new ExecutionException(message, channel, getClass() + " error when process received event .", t);
    }
} 
                                                        |
                                                        V
public void com.alibaba.dubbo.remoting.transport.dispatcher.ChannelEventRunnable#run() {
    //接收到信息
    if (state == ChannelState.RECEIVED) {
        try {
            handler.received(channel, message);
        } catch (Exception e) {
            logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel
                    + ", message is " + message, e);
        }
    } else {
        switch (state) {
        //连接
        case CONNECTED:
            try {
                handler.connected(channel);
            } catch (Exception e) {
                logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel, e);
            }
            break;
        //短连
        case DISCONNECTED:
            try {
                handler.disconnected(channel);
            } catch (Exception e) {
                logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel, e);
            }
            break;
        //发送信息
        case SENT:
            try {
                handler.sent(channel, message);
            } catch (Exception e) {
                logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel
                        + ", message is " + message, e);
            }
        //发生错误
        case CAUGHT:
            try {
                handler.caught(channel, exception);
            } catch (Exception e) {
                logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel
                        + ", message is: " + message + ", exception is " + exception, e);
            }
            break;
        default:
            logger.warn("unknown state: " + state + ", message is " + message);
        }
    }

}
                                                        |
                                                        V
private void com.alibaba.dubbo.remoting.transport.DecodeHandler#decode(Object message) {
    //是否是Decodeable的实例
    if (message != null && message instanceof Decodeable) {
        try {
            //进行消息解码，在前面我们已经分析过它的解码了，完成解码的会将DecodeableRpcInvocation的属性hasDecoded置为true，到了这里就不会再次解码
            ((Decodeable) message).decode();
            if (log.isDebugEnabled()) {
                log.debug("Decode decodeable message " + message.getClass().getName());
            }
        } catch (Throwable e) {
            if (log.isWarnEnabled()) {
                log.warn("Call Decodeable.decode failed: " + e.getMessage(), e);
            }
        } // ~ end of catch
    } // ~ end of if
}
                                                        |
                                                        V
public void com.alibaba.dubbo.remoting.exchange.support.header.HeaderExchangeHandler#received(Channel channel, Object message) throws RemotingException {
    //更新read时间戳
    channel.setAttribute(KEY_READ_TIMESTAMP, System.currentTimeMillis());
    //创建HeaderExchangeChannel
    ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
    try {
        if (message instanceof Request) {
            // handle request.
            Request request = (Request) message;
            //处理事件，如果是只读时间，那么将会给通道设置只读属性
            if (request.isEvent()) {
                handlerEvent(channel, request);
            } else {
                //需要回应吗？
                if (request.isTwoWay()) {
                    //要的，那就构建响应，调用服务接口，返回响应对象
                    Response response = handleRequest(exchangeChannel, request);
                    //然后响应客户端
                    channel.send(response);
                } else {
                    //不需要回应的，自己默默调用完接口就完事了
                    handler.received(exchangeChannel, request.getData());
                }
            }
            //如果是回应，比如我们这边是客户端
            //那好，找到上次请求的时候，我们存储在com.alibaba.dubbo.remoting.exchange.support.DefaultFuture#FUTURES的Future
            //唤醒阻塞在Future上的线程，告诉它结果
            //上一节分析到了响应体的解析，对应响应就是在这里处理的
        } else if (message instanceof Response) {
            handleResponse(channel, (Response) message);
        } else if (message instanceof String) {
            //当前机器在此通信过程中充当客户端吗？
            if (isClientSide(channel)) {
                //是的，好吧，我不支持这种字符串类型的数据
                Exception e = new Exception("Dubbo client can not supported string message: " + message + " in channel: " + channel + ", url: " + channel.getUrl());
                logger.error(e.getMessage(), e);
            } else {
                //如果是服务端，那就使用telnet进行处理，然后向客户端发送消息
                String echo = handler.telnet(channel, (String) message);
                if (echo != null && echo.length() > 0) {
                    channel.send(echo);
                }
            }
        } else {
            //交由下一个处理器处理，至于能不能处理，我就不管了
            handler.received(exchangeChannel, message);
        }
    } finally {
        //要是通道被关闭了，那不好意思，移除它
        HeaderExchangeChannel.removeChannelIfDisconnected(channel);
    }
}
                                                        |
                                                        V
//这个方法是定义在DubboProtocol中的
public Object com.alibaba.dubbo.remoting.exchange.support.ExchangeHandlerAdapter#reply(ExchangeChannel channel, Object message) throws RemotingException {
        if (message instanceof Invocation) {
            Invocation inv = (Invocation) message;
            //获取上次进行导出时构建的invoker，注意此时的invoker是一个责任链
            Invoker<?> invoker = getInvoker(channel, inv);
            // need to consider backward-compatibility if it's a callback
            //请问，这是一个回调请求吗？我们再上面分析了参数回调的情况
            if (Boolean.TRUE.toString().equals(inv.getAttachments().get(IS_CALLBACK_SERVICE_INVOKE))) {
                //回调的那个方法是谁？
                String methodsStr = invoker.getUrl().getParameters().get("methods");
                boolean hasMethod = false;
                if (methodsStr == null || methodsStr.indexOf(",") == -1) {
                    hasMethod = inv.getMethodName().equals(methodsStr);
                } else {
                    String[] methods = methodsStr.split(",");
                    for (String method : methods) {
                        if (inv.getMethodName().equals(method)) {
                            hasMethod = true;
                            break;
                        }
                    }
                }
                //如果不存在那样的方法，那么发出警告，然后忽略本次调用
                if (!hasMethod) {
                    logger.warn(new IllegalStateException("The methodName " + inv.getMethodName()
                            + " not found in callback service interface ,invoke will be ignored."
                            + " please update the api interface. url is:"
                            + invoker.getUrl()) + " ,invocation is :" + inv);
                    return null;
                }
            }
            //在本地线程中记录远程客户端地址
            RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
            //调用我们的服务，这里会经过一系列的filter
            return invoker.invoke(inv);
        }
        throw new RemotingException(channel, "Unsupported request: "
                + (message == null ? null : (message.getClass().getName() + ": " + message))
                + ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress());
    }
```
内容有点多了，下一节分析invoker调用过程


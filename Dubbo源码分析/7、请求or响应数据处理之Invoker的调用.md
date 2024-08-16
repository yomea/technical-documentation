前面我们分析到请求体的解析，最后解析成一个Request，Request持有的值是一个Invocation，再结合在第4节服务的暴露，我们知道dubbo在暴露服务的协议中储存了一个Exporter

```
//group/接口名:version:port -> Exporter
Map<String, Exporter<?>> exporterMap
```
Exporter持有invoker

这个invoker最原始是在com.alibaba.dubbo.config.ServiceConfig#doExportUrlsFor1Protocol创建，代码如下：

```
//（*1*）
Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
//这个类主要是维护着invoker与ServiceConfig的关系
DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);、

//（*1*）
public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
    // TODO Wrapper cannot handle this scenario correctly: the classname contains '$'
    //这个Wrapper的处理逻辑就不再啰嗦了，简单的说就是用于提供方法信息，属性信息，并且通过传入对象和方法名，方法参数直接调用方法（不是反射调用哦）
    final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
    //创建Invoker对象
    return new AbstractProxyInvoker<T>(proxy, type, url) {
        @Override
        protected Object doInvoke(T proxy, String methodName,
                                  Class<?>[] parameterTypes,
                                  Object[] arguments) throws Throwable {
            return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
        }
    };
}
```
这个DelegateProviderMetaDataInvoker实例传入到com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper#export时又被一堆Filter给装饰了以便，形成了一个责任链

```
private static <T> Invoker<T> com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper#buildInvokerChain(final Invoker<T> invoker, String key, String group) {
    Invoker<T> last = invoker;
    //通过SPI获取到所有的复合条件的Filter
    List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);
    if (!filters.isEmpty()) {
        //构建责任链
        for (int i = filters.size() - 1; i >= 0; i--) {
            final Filter filter = filters.get(i);
            final Invoker<T> next = last;
            last = new Invoker<T>() {

                @Override
                public Class<T> getInterface() {
                    return invoker.getInterface();
                }

                @Override
                public URL getUrl() {
                    return invoker.getUrl();
                }

                @Override
                public boolean isAvailable() {
                    return invoker.isAvailable();
                }

                @Override
                public Result invoke(Invocation invocation) throws RpcException {
                    return filter.invoke(next, invocation);
                }

                @Override
                public void destroy() {
                    invoker.destroy();
                }

                @Override
                public String toString() {
                    return invoker.toString();
                }
            };
        }
    }
    return last;
}
```
这个链条目前是经过的过滤器是这样的
EchoFilter -> ClassLoaderFilter -> GenericFilter -> ContextFilter -> TraceFilter -> TimeoutFilter -> MonitorFilter -> ExceptionFilter

```
public Result EchoFilter#invoke(Invoker<?> invoker, Invocation inv) throws RpcException {
    //如果调用的接口是EchoService接口的$echo方法，那么获取第一个参数，直接返回
    //产生的效果就是客户端输入任何内容，最终都会返回相同的内容，这种一般就用于回声测试
    if (inv.getMethodName().equals(Constants.$ECHO) && inv.getArguments() != null && inv.getArguments().length == 1)
        return new RpcResult(inv.getArguments()[0]);
    return invoker.invoke(inv);
}
                                                    |
                                                    V
public Result ClassLoaderFilter#invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
    //将当前线程使用的类加载更换成加载invoker的类加载，避免因invoker的类加载存在只有它能加载的类，而当前线程的类加载器却无法加载的错误
    ClassLoader ocl = Thread.currentThread().getContextClassLoader();
    Thread.currentThread().setContextClassLoader(invoker.getInterface().getClassLoader());
    try {
        return invoker.invoke(invocation);
    } finally {
        Thread.currentThread().setContextClassLoader(ocl);
    }
}
                                                    |
                                                    V
public Result GenericFilter#invoke(Invoker<?> invoker, Invocation inv) throws RpcException {
    //检查当前调用的方式是否为泛化调用
    if (inv.getMethodName().equals(Constants.$INVOKE)
            && inv.getArguments() != null
            && inv.getArguments().length == 3
            && !invoker.getInterface().equals(GenericService.class)) {
            //第一个参数表示方法名
        String name = ((String) inv.getArguments()[0]).trim();
        //方法参数类型数组
        String[] types = (String[]) inv.getArguments()[1];
        //方法参数值数组
        Object[] args = (Object[]) inv.getArguments()[2];
        try {
            //从当前接口中获取对应方法签名
            Method method = ReflectUtils.findMethodByMethodSignature(invoker.getInterface(), name, types);
            //参数类型
            Class<?>[] params = method.getParameterTypes();
            //如果没有传递参数，那么构建一个空的参数数组
            if (args == null) {
                args = new Object[params.length];
            }
            //从附加参数中获取泛化key，这个key用于指定参数使用了什么样的序列化方式
            String generic = inv.getAttachment(Constants.GENERIC_KEY);

            if (StringUtils.isBlank(generic)) {
                generic = RpcContext.getContext().getAttachment(Constants.GENERIC_KEY);
            }
            //如果为空，那么尝试通过通用的方法进行value与类型的转换
            if (StringUtils.isEmpty(generic)
                    || ProtocolUtils.isDefaultGenericSerialization(generic)) {
                args = PojoUtils.realize(args, params, method.getGenericParameterTypes());
                //是否使用的java的序列化方式（ObjectInputStream）
            } else if (ProtocolUtils.isJavaGenericSerialization(generic)) {
                for (int i = 0; i < args.length; i++) {
                    if (byte[].class == args[i].getClass()) {
                        try {
                            UnsafeByteArrayInputStream is = new UnsafeByteArrayInputStream((byte[]) args[i]);
                            args[i] = ExtensionLoader.getExtensionLoader(Serialization.class)
                                    .getExtension(Constants.GENERIC_SERIALIZATION_NATIVE_JAVA)
                                    .deserialize(null, is).readObject();
                        } catch (Exception e) {
                            throw new RpcException("Deserialize argument [" + (i + 1) + "] failed.", e);
                        }
                    } else {
                        。。。。。。省略部分异常代码
                    }
                }
                //使用JavaBean序列化，这个没用过
            } else if (ProtocolUtils.isBeanGenericSerialization(generic)) {
                for (int i = 0; i < args.length; i++) {
                    if (args[i] instanceof JavaBeanDescriptor) {
                        args[i] = JavaBeanSerializeUtil.deserialize((JavaBeanDescriptor) args[i]);
                    } else {
                       。。。。。。省略部分异常代码
            }
            //调用接口获取结果
            Result result = invoker.invoke(new RpcInvocation(method, args, inv.getAttachments()));
            if (result.hasException()
                    && !(result.getException() instanceof GenericException)) {
                    //构建泛化调用异常
                return new RpcResult(new GenericException(result.getException()));
            }
            //将获取的结果按照原来的序列化方式进行序列化，然后返回
            if (ProtocolUtils.isJavaGenericSerialization(generic)) {
                try {
                    UnsafeByteArrayOutputStream os = new UnsafeByteArrayOutputStream(512);
                    ExtensionLoader.getExtensionLoader(Serialization.class)
                            .getExtension(Constants.GENERIC_SERIALIZATION_NATIVE_JAVA)
                            .serialize(null, os).writeObject(result.getValue());
                            /
                    return new RpcResult(os.toByteArray());
                } catch (IOException e) {
                    throw new RpcException("Serialize result failed.", e);
                }
            } else if (ProtocolUtils.isBeanGenericSerialization(generic)) {
                return new RpcResult(JavaBeanSerializeUtil.serialize(result.getValue(), JavaBeanAccessor.METHOD));
            } else {
                return new RpcResult(PojoUtils.generalize(result.getValue()));
            }
        } catch (NoSuchMethodException e) {
            throw new RpcException(e.getMessage(), e);
        } catch (ClassNotFoundException e) {
            throw new RpcException(e.getMessage(), e);
        }
    }
    //普通调用
    return invoker.invoke(inv);
}
                                                    |
                                                    V
//主要是进行一些调用上下文参数的操作
public Result ContextFilter#invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
    Map<String, String> attachments = invocation.getAttachments();
    //移除一些附加参数
    if (attachments != null) {
        attachments = new HashMap<String, String>(attachments);
        attachments.remove(Constants.PATH_KEY);
        attachments.remove(Constants.GROUP_KEY);
        attachments.remove(Constants.VERSION_KEY);
        attachments.remove(Constants.DUBBO_VERSION_KEY);
        attachments.remove(Constants.TOKEN_KEY);
        attachments.remove(Constants.TIMEOUT_KEY);
        attachments.remove(Constants.ASYNC_KEY);// Remove async property to avoid being passed to the following invoke chain.
    }
    //记录信息到上下文中
    RpcContext.getContext()
            .setInvoker(invoker)
            .setInvocation(invocation)
//                .setAttachments(attachments)  // merged from dubbox
            .setLocalAddress(invoker.getUrl().getHost(),
                    invoker.getUrl().getPort());

    // mreged from dubbox
    // we may already added some attachments into RpcContext before this filter (e.g. in rest protocol)
    if (attachments != null) {
        if (RpcContext.getContext().getAttachments() != null) {
            RpcContext.getContext().getAttachments().putAll(attachments);
        } else {
            RpcContext.getContext().setAttachments(attachments);
        }
    }

    if (invocation instanceof RpcInvocation) {
        ((RpcInvocation) invocation).setInvoker(invoker);
    }
    try {
        //调用下一个invoker
        RpcResult result = (RpcResult) invoker.invoke(invocation);
        // pass attachments to result
        result.addAttachments(RpcContext.getServerContext().getAttachments());
        return result;
    } finally {
        RpcContext.removeContext();
        RpcContext.getServerContext().clearAttachments();
    }
}
                                                    |
                                                    V
//这个过滤器主要用于向客户端发送接口调用耗时信息
public Result TraceFilter#invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
    //当前时间
    long start = System.currentTimeMillis();
    Result result = invoker.invoke(invocation);
    //调用接口后的时间
    long end = System.currentTimeMillis();
    if (tracers.size() > 0) {
        //获取需要发送日志的通道
        String key = invoker.getInterface().getName() + "." + invocation.getMethodName();
        Set<Channel> channels = tracers.get(key);
        if (channels == null || channels.isEmpty()) {
            //通过接口名尝试获取
            key = invoker.getInterface().getName();
            channels = tracers.get(key);
        }
        if (channels != null && !channels.isEmpty()) {
            for (Channel channel : new ArrayList<Channel>(channels)) {
                //如果通道还连接着
                if (channel.isConnected()) {
                    try {
                        int max = 1;
                        //获取发送日志的最大限制
                        Integer m = (Integer) channel.getAttribute(TRACE_MAX);
                        if (m != null) {
                            max = (int) m;
                        }
                        int count = 0;
                        //日志发送计数
                        AtomicInteger c = (AtomicInteger) channel.getAttribute(TRACE_COUNT);
                        if (c == null) {
                            c = new AtomicInteger();
                            channel.setAttribute(TRACE_COUNT, c);
                        }
                        count = c.getAndIncrement();
                        //如果没有超过计数，那么向客户端发送接口调用耗时日志
                        if (count < max) {
                            String prompt = channel.getUrl().getParameter(Constants.PROMPT_KEY, Constants.DEFAULT_PROMPT);
                            channel.send("\r\n" + RpcContext.getContext().getRemoteAddress() + " -> "
                                    + invoker.getInterface().getName()
                                    + "." + invocation.getMethodName()
                                    + "(" + JSON.toJSONString(invocation.getArguments()) + ")" + " -> " + JSON.toJSONString(result.getValue())
                                    + "\r\nelapsed: " + (end - start) + " ms."
                                    + "\r\n\r\n" + prompt);
                        }
                        //如果超过了发送限制，可以GG了
                        if (count >= max - 1) {
                            channels.remove(channel);
                        }
                    } catch (Throwable e) {
                        channels.remove(channel);
                        logger.warn(e.getMessage(), e);
                    }
                } else {
                    channels.remove(channel);
                }
            }
        }
    }
    return result;
}
                                                    |
                                                    V
public Result TimeoutFilter#invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
    //当前时间
    long start = System.currentTimeMillis();
    Result result = invoker.invoke(invocation);
    //调用接口耗时
    long elapsed = System.currentTimeMillis() - start;
    //如果调用接口耗时超过了指定的最大耗时时间，那么打印警告日志
    if (invoker.getUrl() != null
            && elapsed > invoker.getUrl().getMethodParameter(invocation.getMethodName(),
            "timeout", Integer.MAX_VALUE)) {
        if (logger.isWarnEnabled()) {
            logger.warn("invoke time out. method: " + invocation.getMethodName()
                    + " arguments: " + Arrays.toString(invocation.getArguments()) + " , url is "
                    + invoker.getUrl() + ", invoke elapsed " + elapsed + " ms.");
        }
    }
    return result;
}
                                                    |
                                                    V
public Result MonitorFilter#invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
    //在前面进行服务暴露时，如果发现使用了监控中心，那么就会在url中添加monitor参数
    if (invoker.getUrl().hasParameter(Constants.MONITOR_KEY)) {
        RpcContext context = RpcContext.getContext(); // provider must fetch context before invoke() gets called
        //获取host
        String remoteHost = context.getRemoteHost();
        long start = System.currentTimeMillis(); // record start timestamp
        //计数
        getConcurrent(invoker, invocation).incrementAndGet(); // count up
        try {
            //调用接口
            Result result = invoker.invoke(invocation); // proceed invocation chain
            //收集信息，向监控中心提交信息，感兴趣的自己去看
            collect(invoker, invocation, result, remoteHost, start, false);
            return result;
        } catch (RpcException e) {
            collect(invoker, invocation, null, remoteHost, start, true);
            throw e;
        } finally {
            getConcurrent(invoker, invocation).decrementAndGet(); // count down
        }
    } else {
        return invoker.invoke(invocation);
    }
}
                                                    |
                                                    V
//这个filter主要进行统一异常处理
public Result ExceptionFilter#invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
    try {
        Result result = invoker.invoke(invocation);
        //如果有异常，并且接口不是泛化接口调用
        if (result.hasException() && GenericService.class != invoker.getInterface()) {
            try {
                //获取异常
                Throwable exception = result.getException();

                // directly throw if it's checked exception
                //直接抛出检查异常
                if (!(exception instanceof RuntimeException) && (exception instanceof Exception)) {
                    return result;
                }
                // directly throw if the exception appears in the signature
                //如果这个异常是接口上定义的异常，直接抛出
                try {
                    Method method = invoker.getInterface().getMethod(invocation.getMethodName(), invocation.getParameterTypes());
                    Class<?>[] exceptionClassses = method.getExceptionTypes();
                    for (Class<?> exceptionClass : exceptionClassses) {
                        if (exception.getClass().equals(exceptionClass)) {
                            return result;
                        }
                    }
                } catch (NoSuchMethodException e) {
                    return result;
                }
                //如果是其他的异常，那么就是一些未检查的异常，打印错误日志
                // for the exception not found in method's signature, print ERROR message in server's log.
                logger.error("Got unchecked and undeclared exception which called by " + RpcContext.getContext().getRemoteHost()
                        + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName()
                        + ", exception: " + exception.getClass().getName() + ": " + exception.getMessage(), exception);

                // directly throw if exception class and interface class are in the same jar file.
                //如果异常来自相同的jar包，直接返回
                String serviceFile = ReflectUtils.getCodeBase(invoker.getInterface());
                String exceptionFile = ReflectUtils.getCodeBase(exception.getClass());
                if (serviceFile == null || exceptionFile == null || serviceFile.equals(exceptionFile)) {
                    return result;
                }
                // directly throw if it's JDK exception
                //JDK包下的异常，直接返回
                String className = exception.getClass().getName();
                if (className.startsWith("java.") || className.startsWith("javax.")) {
                    return result;
                }
                // directly throw if it's dubbo exception
                if (exception instanceof RpcException) {
                    return result;
                }

                // otherwise, wrap with RuntimeException and throw back to the client
                //其他类型，都封装成RuntimeException
                return new RpcResult(new RuntimeException(StringUtils.toString(exception)));
            } catch (Throwable e) {
                logger.warn("Fail to ExceptionFilter when called by " + RpcContext.getContext().getRemoteHost()
                        + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName()
                        + ", exception: " + e.getClass().getName() + ": " + e.getMessage(), e);
                return result;
            }
        }
        return result;
    } catch (RuntimeException e) {
        logger.error("Got unchecked and undeclared exception which called by " + RpcContext.getContext().getRemoteHost()
                + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName()
                + ", exception: " + e.getClass().getName() + ": " + e.getMessage(), e);
        throw e;
    }
}
```
接着就调用到了在com.alibaba.dubbo.config.ServiceConfig#doExportUrlsFor1Protocol创建的DelegateProviderMetaDataInvoker，继续调用到

```
public Result AbstractProxyInvoker#invoke(Invocation invocation) throws RpcException {
    try {
        return new RpcResult(doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments()));
    } catch (InvocationTargetException e) {
        return new RpcResult(e.getTargetException());
    } catch (Throwable e) {
        throw new RpcException("Failed to invoke remote proxy method " + invocation.getMethodName() + " to " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
                                                    |
                                                    V
public <T> Invoker<T> JavassistProxyFactory#getInvoker(T proxy, Class<T> type, URL url) {
    // TODO Wrapper cannot handle this scenario correctly: the classname contains '$'
    final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
    return new AbstractProxyInvoker<T>(proxy, type, url) {
        @Override
        protected Object doInvoke(T proxy, String methodName,
                                  Class<?>[] parameterTypes,
                                  Object[] arguments) throws Throwable {
            return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
        }
    };
}
```
Wrapper的作用就不在赘述了
最终调用的就是我们自己写的接口实现了

```
public User getUser(long userId) {
	
    return new User("小明", 18);
}
```
接口调用成功之后，dubbo需要将结果告诉客户端，这就是下一节需要分析的数据回应

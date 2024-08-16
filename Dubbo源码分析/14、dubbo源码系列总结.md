## 一、服务暴露过程

-》 创建ServiceBean，这个类实现了InitializingBean

-》 准备各种配置，保证服务暴露时不会缺失属性

-》 ServiceBean实现了ApplicationListener接口，在spring触发ContextRefreshEvent时，开始暴露服务。

-》 检查存根服务或者本地服务（现在已经属于废弃状态），所谓存根，就是一个接口的本地实现，用于提供缓存服务或者其他的操作，它的其中一个参数就是能够传入一个远程
代理实现，用于在缓存未命中时调用

-》 检查mock服务，但服务调用失败时，进行降级处理

-》 将配置参数与协议服务构建URL --> 比如：dubbo://192.168.56.1:20880/groupName/com.dubbo.service.UserService:1.0?anyhost=true&application=hello-world-app&bean.name=com.dubbo.service.UserService
&bind.ip=192.168.56.1&bind.port=20880&dubbo=2.0.2&generic=false&interface=com.dubbo.service.UserService&methods=sayHello
&pid=26392&side=provider&timestamp=1567943599518

-》 通过SPI获取代理工厂，默认的是（JavassistFactory），创建Invoker

-》 创建Exporter，用于记录已经暴露的Invoker，消费者可以通过对应的协议找到已经注册的服务提供接口实现，然后调用

-》 包装Exporter为ListenerExporterWrapper并设置监听器，在服务暴露与注销时触发监听器的调用

-》 使用ProtocolFilterWrapper创建Filter拦截链

-》 创建HeaderExchangeServer并在构造器中启动心跳，在netty的通道属性上记录通信时间，每隔固定时间发送心跳事件，心跳得到回应就会刷新通道时间，其他通信方式，比
如收到请求时，也会主动刷新通道时间，如果超过允许的心跳超时，那么关闭通道（如果是客户端，会尝试重新连接，调用Boostrap进行连接）

-》 创建ServerBoostrap，创建netty服务，添加解码器

-》 根据我们设置的协议创建对应的注册中心实例，比如redis，那么就会创建redis连接，通过hset设置提供者地址，hash键为gourpName+接口名+/+category名
（提供者的为provider）hash表的key值为提供者的url，value为注册时间

-》 设置成功后，发布以通道为hash键的注册事件，那么订阅了这个通道的redis客户端就可以收到消息，用于更新自己的提供者目录

-》 设置调度器，调度未注册成功，未注销成功，未订阅成功，未退订成功的

## 二、服务引入过程

-》 创建ReferenceBean，这个类也实现了InitializingBean

-》 准备好各种配置，检查存根服务或者本地服务（现在已经属于废弃状态），所谓存根，就是一个接口的本地实现，用于提供缓存服务或者其他的操作，
它的其中一个参数就是能够传入一个远程代理实现，用于在缓存未命中时调用

-》 检查mock服务，但服务调用失败时，进行降级处理

-》 暂时修改注册中心协议为registry，通过SPI获取registryProtocol，在registryProtocol协议的refer方法中恢复协议，假设是redis，那么协议url变回redis://xxxx

-》 根据设置的协议redis创建RedisRegistry，订阅groupname/接口名/providers，configurators，routers，当这些redis通道发生变化时会收到提醒

-》 根据不同的订阅目录，比如providers，configurators，routers对url进行分组，形成不同的List

-》 接下来就是和服务暴露差不多的逻辑了，创建ExchangeClient，利用netty创建与提供链接的nettyClient

下面是服务引入所创建的一些重要类
```
MockClusterInvoker
    invoker -> FailoverClusterInvoker(如果消费者设置了group，那么这个会变成MergeableClusterInvoker)
                    directory -> RegistryDirectory(具有动态更新提供者目录的功能)
                                    urlInvokerMap -> 这是一个Map，key为提供者url，值为InvokeDelegate

        
InvokeDelegate
    invoker -> ProtocolFilterWrapper
                invoker -> ListenerInvokerWrapper
                            invoker -> DubboInvoker
                                        clients -> 这是一个数组，它的元素个数取决于配置服务引入时设置了允许多少连接数，其实例为ReferenceCountExchangeClient
                                        
                                        
ReferenceCountExchangeClient
    client -> HeaderExchangeClient
                client -> NettyClient
                            channel -> NioSocketChannel
                            handler -> MultiMessageHandler
                                            handler -> HeartbeatHandler
                                                           handler -> AllChannelHandler
                                                                            handler -> DecodeHandler
                                                                                            handler -> HeaderExchangeChannel
                                                                                                            handler -> DubboProtocol#requestHandler(内部类)

```

## 三、dubbo请求协议头

协议头有16个字节（0 ~ 127）大小

0~15位：魔数 oxdabb

16位：是否为Request,1为request，0为response

17位：是否为twoWay，从源代码来看这个twoWay的意思好像表示是否需要响应的意思

18位：是否是事件，表示心跳事件，可能当前的请求或者响应来自心跳，不是接口消费请求

19~23位：序列化id，用于标识当前使用的数据序列化方式是什么，比如hessian2

24~31位：状态，比如ok（20）

32~95位：请求的id，从后面的源码中，我发现它会以key为请求id -> DefaultFuture存储在com.alibaba.dubbo.remoting.exchange.support.DefaultFuture#FUTURES中
而这个DefaultFuture类似于JDK的Future，调用其get方法会发生阻塞知道有结果处理完毕

96~127位：标识请求体或者响应体的数据长度

128及以后：数据，如果是响应，第一个字节保存调用结果，用于表示返回值是否null，是否有异常，是否有附带参数，是否正常调用返回了值

## 四、请求处理过程

-》 消费者端

-》 检查是否强制要求mock服务调用，如果不是强制要求并且需要mock服务调用的话，在调用远程接口失败时调用mock服务

-》 获取提供者，通过路由进行一波选择，然后再通过负载均衡获取提供者

-》 如果调用时发生错误（会对异常进行判断是否是业务异常，换句话说就是不是提供者那边抛出的业务错误，比如通道被关闭这种），那么进行换一个提供者进行重试（故障转移），重试次数默认2次，重试的时候会重新进行路由筛选（因为抛出了错误，那么就有可能是因为提供者下掉了，所以要重新筛选），然后再次根据负载均衡去获取一个invoker。

-》 获取提供者时会判断是否使用了粘性策略，换句话说就是使用上次筛选出来的提供者

-》 通过负载均衡获取提供者，默认使用随机策略，如果给提供者设置了权重，那么会通过权重来进行概率性计算

-》 获取到invoker之后可能内部定义了多个连接，即有多个客户端，那么通过轮询的方式获取其中一个

-》 执行filter链

-》 如果请求是异步的并且不需要接收响应的，那么返回一个空RpcResult对象即可，如果是请求是异步的并且需要提供者响应，那么将
首先构建一个Future（dubbo的future），然后以 请求id -》future的方式
注册到DefaultFuture#FUTURES中，最后将这个future设置到RpcContext中。用户可以通过这个RpcContext获取到这个future

-》 如果请求不是异步的，那么首先构建一个Future（dubbo的future），然后以 请求id -》future的方式注册到DefaultFuture#FUTURES中，然后调用future的get方法进行
等待，如果设置了等待时间，那么超过这个时间远程服务没有返回，那么会抛错

-》 创建Request，通过InternalEncoder进行编码

```
protected void com.alibaba.dubbo.rpc.protocol.dubbo.DubboCodec#encodeRequestData(Channel channel, ObjectOutput out, Object data, String version) throws IOException {
    RpcInvocation inv = (RpcInvocation) data;
    //写入协议版本
    out.writeUTF(version);
    //写入接口
    out.writeUTF(inv.getAttachment(Constants.PATH_KEY));
    //写入接口版本
    out.writeUTF(inv.getAttachment(Constants.VERSION_KEY));
    //写入方法名
    out.writeUTF(inv.getMethodName());
    //写入参数类型
    out.writeUTF(ReflectUtils.getDesc(inv.getParameterTypes()));
    Object[] args = inv.getArguments();
    if (args != null)
        for (int i = 0; i < args.length; i++) {
            //写入参数，如果存在回调参数，那么会在注册exporter，以便服务端进行回调时能够调用
            out.writeObject(encodeInvocationArgument(channel, inv, i));
        }
    //写入附加参数
    out.writeObject(inv.getAttachments());
}
```
                                        |
                                        |
                                        V

-》 服务提供者端

-》 通过InternalDecoder进行解码，读取16字节的协议头，解析魔数，不是魔数的数据将以telnet命令的方式进行解析

-》 直到找到魔数头为止，然后开始检查数据体是否超过负载，超过会抛错

-》 解析flag，1表示请求，0表示响应，解析协议类型

-》 此处我们讨论的是请求过程，那么这个flag就是1，默认序列化协议为hessian2，解析请求id

-》 创建Request对象，创建DecodeableRpcInvocation，然后调用DecodeableRpcInvocation#decode方法解析数据体

-》 解析请求接口版本，接口名，方法名，参数，参数类型

-》 处理参数回调，生成代理，在提供者的方法体调用了这个回调参数后会向消费端发送请求（和消费者请求提供者是一样的格式）

-》 触发netty的通道读事件

-》 经过一系列dubbo的通道处理，然后从对应的协议中的com.alibaba.dubbo.rpc.protocol.AbstractProtocol#exporterMap<String, Exporter<?>>中获取到导出的invoker

-》 调用dubbo的Filter链，其中有个GenericFilter，用于处理泛化调用的参数的，最终调用目标服务实现

-》 将返回结果包装成Response对象，然后再包装成RpcResult对象

-》 使用InternalEncoder编码器对返回值进行编码

-》 根据调用情况返回不同的请求体：
1、无异常，但返回值为null的，如果有附加参数，设置附加参数
2、无异常，有返回值的，如果有附加参数，设置附加参数
3、有异常的，如果有附加参数，设置附加参数

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
        //写入异常
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
                                        |
                                        |
                                        V
-》 消费端

-》 创建DecodeableRpcResult，根据服务端返回的参数类型进行解码

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
                //如果这个obj不是异常类型，嘻嘻，报错
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
解析返回值

```
public Object recreate() throws Throwable {
    if (exception != null) {
        //如果有异常，抛出
        throw exception;
    }
    return result;
}
```



## 类介绍

- ProviderConfig：提供者配置，用于作为父级配置，也可以指定为默认配置，为那些不
  被<provider>标签包裹的service标签提供默认配置
- ApplicationConfig：用于提供应用信息，比如版本，应用名，使用的JDK版本，所属组
  织，使用的注册中心，监控中心等
- ModuleConfig：模块信息
- RegistryConfig：注册信息，用于定义注册中心地址，密码，用户名，协议等
- MonitorConfig：监控中心，定义协议，地址等，监控中心用于获取服务调用信息，记录
  调用次数，健康状态
- ConsumerConfig：消费者配置，通常用于作为其他引入的父配置，或者作为默认配置
- ProtocolConfig：协议配置，用于定义提供者将暴露在什么端口，什么地址，使用什么
  协议进行通信
- ServiceBean：服务提供者bean，除了定义提供者配置外，它将合并所有配置，进行服务
  的暴露和注册
- ReferenceBean：服务引入bean，用于为消费者接口生成代理，提供远程调用的能力
- ProxyFactory：代理工厂，实现了有JavassistProxyFactory和JdkProxyFactory等，用于生成代理类
- ExtensionLoader：用于加载SPI
- Protocol：协议，比如有DubboProtocol实现类，用于通过什么样的协议去暴露服务，注册什么样的协议编码解码器
- Exchanger：定义了绑定，连接的能力
- NettyServer：维护中创建服务socket时的ServerBootstrap，netty的Channel，还有两个EventLoopGroup（Boss和Worker）
- InternalDecoder：解码器
- InternalEncoder：编码器
- NettyServerHandler：netty通道处理器，内部适配了dubbo自己的通道处理器
- MockClusterInvoker：提供mock服务的调用
- FailoverClusterInvoker：故障转移，比如提供者调不通的时候，会自动调用其他的提供者
- RegistryDirectory：注册目录，用于维护提供者目录


## 设计模式

- 代理模式：JavassistProxyFactory
- 工厂模式：JavassistProxyFactory
- 策略模式：SPI，根据url参数的指定获取不同的实现
- 适配器模式：NettyServerHandler适配dubbo自己的通道处理
- 责任链模式：filter
- 装饰器模式：各种Wrapper，ProtocolFilterWrapper，通道handler
- 观察者模式：ListenerExporterWrapper，在服务暴露与注销时调用监听器

## SPI

Service Provider Interface

通过ExtensionLoader加载META-INF/services/，META-INF/dubbo/，META-INF/dubbo/internal/下的服务。SPI服务实现类必须有@SPI注解，@SPI注解可以指定默认的自适应接口实现，

如果某个实现类上面标注了@Adaptive，那么这个类优先被选中为自适应的默认实现，如果没有找到标注为@Adaptive的实现类，那么会通过javasist进行动态生成一个接口实现，
这个动态生成的实现类会实现被@Adaptive标注的方法，没有被@Adaptive标注的方法默认实现为抛出不支持的异常，被@Adaptive标注的方法，必须要有URL参数或者对应传入的bean有
getUrl方法，否则抛错，@Adaptive注解上需要指定从URl中获取什么样的参数值的参数名，如果url上面没有指定对应参数名的值，那么使用@SPI注解上指定的默认扩展名去获取接口实现。

@Activate：这个注解主要用于标识每个SPI实现类被激活，比如我们使用的Filter，只有被标注了@Activate注解的才能被用于构建过滤器链。

@Extension：这个注解已被废弃，用来指定扩展实现的名字，通常我们在配置SPI实现的时候就会指定SPI扩展名
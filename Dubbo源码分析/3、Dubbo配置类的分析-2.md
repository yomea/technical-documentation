接着上一节的内容，上一节我们分析到了com.alibaba.dubbo.config.ServiceConfig#doExport方法，接下来我们继续分析com.alibaba.dubbo.config.ServiceConfig#doExportUrls方法

```
private void doExportUrls() {
    //获取需要注册的url
    List<URL> registryURLs = loadRegistries(true);
    for (ProtocolConfig protocolConfig : protocols) {
        doExportUrlsFor1Protocol(protocolConfig, registryURLs);
    }
}
```
> loadRegistries

```
protected List<URL> com.alibaba.dubbo.config.AbstractInterfaceConfig#loadRegistries(boolean provider) {
    //检查注册中心配置，如果没有设置，就会从系统属性，变量，配置文件中获取
    checkRegistry();
    List<URL> registryList = new ArrayList<URL>();
    if (registries != null && !registries.isEmpty()) {
        for (RegistryConfig config : registries) {
            //获取注册中心地址
            String address = config.getAddress();
            //未设置地址
            if (address == null || address.length() == 0) {
                //0.0.0.0
                address = Constants.ANYHOST_VALUE;
            }
            //从系统属性中获取注册中心地址
            String sysaddress = System.getProperty("dubbo.registry.address");
            if (sysaddress != null && sysaddress.length() > 0) {
                address = sysaddress;
            }
            if (address.length() > 0 && !RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(address)) {
                Map<String, String> map = new HashMap<String, String>();
                //读取ApplicationConfig中参数，可以通过@Parameter注解拒绝追加参数
                appendParameters(map, application);
                //读取注册中心配置的参数
                appendParameters(map, config);
                //路径
                map.put("path", RegistryService.class.getName());
                //设置dubbo协议版本
                map.put("dubbo", Version.getProtocolVersion());
                //记录时间戳
                map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
                if (ConfigUtils.getPid() > 0) {
                    //设置当前jvm进程的pid，ConfigUtils.getPid()使用了ManagementFactory获取jvm的运行参数
                    map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
                }
                //如果未设置协议参数，尝试检查是否存在RegistryFactory的spi
                if (!map.containsKey("protocol")) {
                    //RegistryFactory目前的实现类有多个，比如ZookeeperRegistryFactory，RedisRegistryFactory，DubboRegistryFactory，MulticastRegistryFactory
                    if (ExtensionLoader.getExtensionLoader(RegistryFactory.class).hasExtension("remote")) {
                        map.put("protocol", "remote");
                    } else {
                        map.put("protocol", "dubbo");
                    }
                }
                //(*1*)
                List<URL> urls = UrlUtils.parseURLs(address, map);
                for (URL url : urls) {
                    //添加参数 registry -》 协议
                    url = url.addParameter(Constants.REGISTRY_KEY, url.getProtocol());
                    //设置协议registry
                    url = url.setProtocol(Constants.REGISTRY_PROTOCOL);
                    //是否允许注册，或者允许订阅
                    if ((provider && url.getParameter(Constants.REGISTER_KEY, true))
                            || (!provider && url.getParameter(Constants.SUBSCRIBE_KEY, true))) {
                        registryList.add(url);
                    }
                }
            }
        }
    }
    return registryList;
}

//(*1*)
public static List<URL> com.alibaba.dubbo.common.utils.UrlUtils#parseURLs(String address, Map<String, String> defaults) {
    if (address == null || address.length() == 0) {
        return null;
    }
    //"\\s*[|;]+\\s*"
    String[] addresses = Constants.REGISTRY_SPLIT_PATTERN.split(address);
    if (addresses == null || addresses.length == 0) {
        return null; //here won't be empty
    }
    List<URL> registries = new ArrayList<URL>();
    for (String addr : addresses) {
        //(*2*)
        registries.add(parseURL(addr, defaults));
    }
    return registries;
}

//(*2*)
public static URL com.alibaba.dubbo.common.utils.UrlUtils#parseURL(String address, Map<String, String> defaults) {
    if (address == null || address.length() == 0) {
        return null;
    }
    String url;
    //如果存在://，那么可以认为这是一个标准的url
    if (address.indexOf("://") >= 0) {
        url = address;
    } else {
        //"\\s*[,]+\\s*"
        String[] addresses = Constants.COMMA_SPLIT_PATTERN.split(address);
        //获取第一个url
        url = addresses[0];
        if (addresses.length > 1) {
            StringBuilder backup = new StringBuilder();
            for (int i = 1; i < addresses.length; i++) {
                if (i > 1) {
                    backup.append(",");
                }
                //后面的地址使用逗号拼接
                backup.append(addresses[i]);
            }
            
            url += "?" + Constants.BACKUP_KEY + "=" + backup.toString();
        }
    }
    //获取协议
    String defaultProtocol = defaults == null ? null : defaults.get("protocol");
    //如果没有就设置默认的协议
    if (defaultProtocol == null || defaultProtocol.length() == 0) {
        defaultProtocol = "dubbo";
    }
    //获取用户名
    String defaultUsername = defaults == null ? null : defaults.get("username");
    //获取密码
    String defaultPassword = defaults == null ? null : defaults.get("password");
    //获取端口
    int defaultPort = StringUtils.parseInteger(defaults == null ? null : defaults.get("port"));
    //获取默认路径
    String defaultPath = defaults == null ? null : defaults.get("path");
    Map<String, String> defaultParameters = defaults == null ? null : new HashMap<String, String>(defaults);
    //移除上面获取过的参数值
    if (defaultParameters != null) {
        defaultParameters.remove("protocol");
        defaultParameters.remove("username");
        defaultParameters.remove("password");
        defaultParameters.remove("host");
        defaultParameters.remove("port");
        defaultParameters.remove("path");
    }
    //从url中获取协议，用户名，密码等
    URL u = URL.valueOf(url);
    boolean changed = false;
    String protocol = u.getProtocol();
    String username = u.getUsername();
    String password = u.getPassword();
    String host = u.getHost();
    int port = u.getPort();
    String path = u.getPath();
    Map<String, String> parameters = new HashMap<String, String>(u.getParameters());
    //如果没有从参数配置中获取
    if ((protocol == null || protocol.length() == 0) && defaultProtocol != null && defaultProtocol.length() > 0) {
        changed = true;
        protocol = defaultProtocol;
    }
    //如果没有从参数配置中获取
    if ((username == null || username.length() == 0) && defaultUsername != null && defaultUsername.length() > 0) {
        changed = true;
        username = defaultUsername;
    }
    //如果没有从参数配置中获取
    if ((password == null || password.length() == 0) && defaultPassword != null && defaultPassword.length() > 0) {
        changed = true;
        password = defaultPassword;
    }
    /*if (u.isAnyHost() || u.isLocalHost()) {
        changed = true;
        host = NetUtils.getLocalHost();
    }*/
    //如果没有从参数配置中获取
    if (port <= 0) {
        if (defaultPort > 0) {
            changed = true;
            port = defaultPort;
        } else {
            changed = true;
            port = 9090;
        }
    }
    //如果没有从参数配置中获取
    if (path == null || path.length() == 0) {
        if (defaultPath != null && defaultPath.length() > 0) {
            changed = true;
            path = defaultPath;
        }
    }
    //获取额外自定义参数
    if (defaultParameters != null && defaultParameters.size() > 0) {
        for (Map.Entry<String, String> entry : defaultParameters.entrySet()) {
            String key = entry.getKey();
            String defaultValue = entry.getValue();
            if (defaultValue != null && defaultValue.length() > 0) {
                String value = parameters.get(key);
                if (value == null || value.length() == 0) {
                    changed = true;
                    parameters.put(key, defaultValue);
                }
            }
        }
    }
    //如果发生参数的调整，那么重新new出一个url
    if (changed) {
        u = new URL(protocol, username, password, host, port, path, parameters);
    }
    //返回
    return u;
}
```

> doExportUrlsFor1Protocol

```
private void com.alibaba.dubbo.config.ServiceConfig#doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    //协议名
    String name = protocolConfig.getName();
    if (name == null || name.length() == 0) {   
        //没有就使用默认的协议名
        name = "dubbo";
    }

    Map<String, String> map = new HashMap<String, String>();
    //side -> provider
    map.put(Constants.SIDE_KEY, Constants.PROVIDER_SIDE);
    //dubbo -> 2.0.2(协议版本)
    map.put(Constants.DUBBO_VERSION_KEY, Version.getProtocolVersion());
    //timestamp -> 记录当前时间戳
    map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
    //获取当前虚拟机进程的pid
    if (ConfigUtils.getPid() > 0) {
        //pid -> 记录当前虚拟机的进程pid
        map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
    }
    //追加ApplicationConfig配置项参数
    appendParameters(map, application);
    //追加ModuleConfig配置项参数
    appendParameters(map, module);
    //追加application，DEFAULT_KEY = default，表示添加参数key时拼上前缀default
    appendParameters(map, provider, Constants.DEFAULT_KEY);
    //追加ProtocolConfig配置项参数
    appendParameters(map, protocolConfig);
    //追加当前ServiceBean的配置项参数
    appendParameters(map, this);
    
    //方法配置MethodConfig
    if (methods != null && !methods.isEmpty()) {
        for (MethodConfig method : methods) {
            //追加参数，参数前缀方法名
            appendParameters(map, method, method.getName());
            //一看这个retry就可以猜测这个MethodConfig肯定存在retry属性
            String retryKey = method.getName() + ".retry";
            //查看了MethodConfig的源码，发现retry属性被定为废弃的，所以这里要做些特殊处理，以便兼容
            if (map.containsKey(retryKey)) {
                //移除原来的
                String retryValue = map.remove(retryKey);
                //如果是false，那么转化成新的，key为方法名拼上retries，值为0
                if ("false".equals(retryValue)) {
                    map.put(method.getName() + ".retries", "0");
                }
            }
            //获取方法的参数配置
            List<ArgumentConfig> arguments = method.getArguments();
            if (arguments != null && !arguments.isEmpty()) {
                for (ArgumentConfig argument : arguments) {
                    // convert argument type
                    if (argument.getType() != null && argument.getType().length() > 0) {
                        //获取提供服务的接口的所有的方法对象
                        Method[] methods = interfaceClass.getMethods();
                        // visit all methods
                        if (methods != null && methods.length > 0) {
                            for (int i = 0; i < methods.length; i++) {
                                String methodName = methods[i].getName();
                                // target the method, and get its signature
                                //寻找到与配置的方法名相匹配的方法，当然即使是方法名匹配也不一定是匹配的，主要还要看参数
                                if (methodName.equals(method.getName())) {
                                    //获取接口方法的参数类型列表
                                    Class<?>[] argtypes = methods[i].getParameterTypes();
                                    // one callback in the method
                                    //参数位置
                                    if (argument.getIndex() != -1) {
                                        //参数是否匹配
                                        if (argtypes[argument.getIndex()].getName().equals(argument.getType())) {
                                            appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                                        } else {
                                            //
                                            throw new IllegalArgumentException("argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
                                        }
                                    } else {
                                        // multiple callbacks in the method
                                        for (int j = 0; j < argtypes.length; j++) {
                                            Class<?> argclazz = argtypes[j];
                                            if (argclazz.getName().equals(argument.getType())) {
                                                appendParameters(map, argument, method.getName() + "." + j);
                                                if (argument.getIndex() != -1 && argument.getIndex() != j) {
                                                    throw new IllegalArgumentException("argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
                                                }
                                            }
                                        }
                                    }
                                }
                            }
                        }
                    } else if (argument.getIndex() != -1) {
                        appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                    } else {
                        throw new IllegalArgumentException("argument config must set index or type attribute.eg: <dubbo:argument index='0' .../> or <dubbo:argument type=xxx .../>");
                    }

                }
            }
        } // end of methods for
    }
    //是否需要调用，这个值在dubbo解析interfaceName的时候会进行一次检查
    if (ProtocolUtils.isGeneric(generic)) {
        map.put(Constants.GENERIC_KEY, generic);
        map.put(Constants.METHODS_KEY, Constants.ANY_VALUE);
    } else {
        //从MANIFEST.MF获取版本->从源代码所在的jar名提取版本（像maven这样的打包工具就会提供版本）->使用ServerConfig配置的版本
        String revision = Version.getVersion(interfaceClass, version);
        if (revision != null && revision.length() > 0) {
            map.put("revision", revision);
        }
        //通过Wrapper工具类使用javasist生成包装对象，用于储存当前接口的一些元数据
        //(*1*)
        String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
        if (methods.length == 0) {
            logger.warn("NO method found in service interface " + interfaceClass.getName());
            map.put(Constants.METHODS_KEY, Constants.ANY_VALUE);
        } else {
            //用逗号分割
            map.put(Constants.METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
        }
    }
    //是否使用token
    if (!ConfigUtils.isEmpty(token)) {
        if (ConfigUtils.isDefault(token)) {
            map.put(Constants.TOKEN_KEY, UUID.randomUUID().toString());
        } else {
            map.put(Constants.TOKEN_KEY, token);
        }
    }
    //如果协议为本地协议，injvm，进行本地调用
    if (Constants.LOCAL_PROTOCOL.equals(protocolConfig.getName())) {
        protocolConfig.setRegister(false);
        map.put("notify", "false");
    }
    // export service
    //获取协议上下文路径，对于http请求，这里指定的上下路径是servlet的urlPattern
    String contextPath = protocolConfig.getContextpath();
    if ((contextPath == null || contextPath.length() == 0) && provider != null) {
        contextPath = provider.getContextpath();
    }
    //获取配置的主机地址，对于multicast（广播），会主动打开socket去连接，然后获取ip地址
    String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
    //获取端口
    Integer port = this.findConfigedPorts(protocolConfig, name, map);
    //创建url（注意，这个URL是dubbo自己定义的一个model，不是JDK的URL）
    URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);
    //通过SPI获取配置工厂实现，目前有两个实现，一个是补充参数的AbsentConfiguratorFactory
    //OverrideConfiguratorFactory
    if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
            .hasExtension(url.getProtocol())) {
        url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                //getConfigurator这个方法传入的url参数表示配置url，configure方法的url参数是被配置的url，它的参数可能会被配置url改变
                .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
    }
    //服务范围
    String scope = url.getParameter(Constants.SCOPE_KEY);
    // don't export when none is configured
    //为none时，不会进行服务的暴露
    if (!Constants.SCOPE_NONE.toString().equalsIgnoreCase(scope)) {

        // export to local if the config is not remote (export to remote only when config is remote)
        //如果不是remote，那就是local，进行本地暴露
        if (!Constants.SCOPE_REMOTE.toString().equalsIgnoreCase(scope)) {
            exportLocal(url);
        }
        // export to remote if the config is not local (export to local only when config is local)
        //远程暴露
        if (!Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope)) {
            if (logger.isInfoEnabled()) {
                logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
            }
            if (registryURLs != null && !registryURLs.isEmpty()) {
                for (URL registryURL : registryURLs) {
                    url = url.addParameterIfAbsent(Constants.DYNAMIC_KEY, registryURL.getParameter(Constants.DYNAMIC_KEY));
                    //获取监控中心的url
                    URL monitorUrl = loadMonitor(registryURL);
                    if (monitorUrl != null) {
                        //给提供者url添加监控中心地址，monitorUrl.toFullString()方法会构建成一个形入 协议名://ip地址:端口号?key1=value1&key2=value2的url地址
                        url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
                    }
                    if (logger.isInfoEnabled()) {
                        logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
                    }

                    // For providers, this is used to enable custom proxy to generate invoker
                    String proxy = url.getParameter(Constants.PROXY_KEY);
                    if (StringUtils.isNotEmpty(proxy)) {
                        registryURL = registryURL.addParameter(Constants.PROXY_KEY, proxy);
                    }
                    //代理工厂，生成Invoker，这个proxyFactory是通过spi获取，javassistProxyFactory，JDKProxyFactory
                    //关于SPI的分析，我们会在后面的章节单独分析
                    //ProxyFactory的注解@SPI注解的默认的name是javassist，当然此处的proxyFactory实例其实dubbo用javassist生成的
                    //但是内部方法实现依然获取的是默认的代理工厂，所以我们就用JavassistProxyFactory进行分析
                    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
                    //静态代理，DelegateProviderMetaDataInvoker也是实现了Invoker接口，主要维护invoker与服务配置之间的关系
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
                    //这个protocol也是一个spi，此处的实例实际上是dubbo生成的一个对象，内部通过ExtensionLoader获取到RegistryProtocol
                    Exporter<?> exporter = protocol.export(wrapperInvoker);
                    //记录暴露的服务
                    exporters.add(exporter);
                }
            } else {
                Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
                DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                Exporter<?> exporter = protocol.export(wrapperInvoker);
                exporters.add(exporter);
            }
        }
    }
    //记录当前服务提供者暴露的地址
    this.urls.add(url);
}
```
上面的标号（*1*）中将一个接口进行了包装，生成了一个Wrapper的子类，用于调用方法，获取属性，方法，设置属性，假设我们有以下几个类

> 将要被包装的一个类

```
public class UserServiceImpl implements UserService {

	private String a;

	@Override
	public void sayHello() {
		System.out.println("hello world!!!");

	}

	public String getA() {
		return a;
	}

	public void setA(String a) {
		this.a = a;
	}
}
```
> 测试main方法

```
public static void main(String[] args) {
    Wrapper wrapper = Wrapper.getWrapper(UserServiceImpl.class);
    wrapper.setPropertyValue(new UserServiceImpl(), "a", "hello world!");
}
```

那么这个通过javasist生成代码如下：

```
public class Wrapper0 {

    //属性名
    public static String[] pns;

    //属性名-》属性类型
    public static java.util.Map pts;

    //pulic 方法名，通过getMethods()获取的方法的方法名
    public static String[] mns;

    //当前类定义的public的方法名，通过getMethods()获取，最后进行的筛选
    public static String[] dmns;

    //参数类型，com.dubbo.service.impl.UserServiceImpl中就只有一个sayHello方法，这个mts0就表示这个方法的参数
    //如果有多个方法，那么又会多出一个public static Class[] mts1;
    public static Class[] mts0;

    public String[] getPropertyNames(){
        return pns;
    }

    public boolean hasProperty(String n){
        return pts.containsKey(n);
    }

    public Class getPropertyType(String n){
        return (Class)pts.get(n);
    }
    public String[] getMethodNames(){
        return mns;
    }

    public String[] getDeclaredMethodNames(){
        return dmns;
    }

    //o：UserServiceImpl对象，n：属性名，v：属性值
    public void setPropertyValue(Object o, String n, Object v){
        com.dubbo.service.impl.UserServiceImpl w;
        try{
            w = ((com.dubbo.service.impl.UserServiceImpl)o);
        }catch(Throwable e){
            throw new IllegalArgumentException(e);
        }
        if( n.equals("a") ){
            w.setA((java.lang.String)v);
            return;
        }
        throw new com.alibaba.dubbo.common.bytecode.NoSuchPropertyException("Not found property \""+n+"\" filed or setter method in class com.dubbo.service.impl.UserServiceImpl.");
    }

    //o：UserServiceImpl对象，n：属性名
    public Object getPropertyValue(Object o, String n){
        com.dubbo.service.impl.UserServiceImpl w;
        try{
            w = ((com.dubbo.service.impl.UserServiceImpl)o);
        }catch(Throwable e){
            throw new IllegalArgumentException(e); }
        if( n.equals("a") ){
            //$w表示包装类型，如果a是基本类型，比如int，那么会变成这样的语句
            //return (java.lang.Integer)w.getA();
            //如果本身就是对象类型，那么什么也不做
            //return ($w) w.getA();
            return w.getA();
        }
        throw new com.alibaba.dubbo.common.bytecode.NoSuchPropertyException("Not found property \""+n+"\" filed or setter method in class com.dubbo.service.impl.UserServiceImpl.");
    }

    //o：UserServiceImpl对象，n：方法名，p：方法参数类型数组，v：方法参数
    public Object invokeMethod(Object o, String n, Class[] p, Object[] v) throws java.lang.reflect.InvocationTargetException{
        com.dubbo.service.impl.UserServiceImpl w;
        try{
            w = ((com.dubbo.service.impl.UserServiceImpl)o);
        }catch(Throwable e){
            throw new IllegalArgumentException(e);
        }
        try{
            if( "sayHello".equals( n )  &&  p.length == 0 ) {
                w.sayHello();
                return null;
            }
            if( "getA".equals( n )  &&  p.length == 0 ) {
                return w.getA(); }
            if( "setA".equals( n )  &&  p.length == 1 ) {
                w.setA((java.lang.String)v[0]); return null;
            }
        } catch(Throwable e) {
            throw new java.lang.reflect.InvocationTargetException(e);
        }
        throw new com.alibaba.dubbo.common.bytecode.NoSuchMethodException("Not found method \""+n+"\" in class com.dubbo.service.impl.UserServiceImpl.");
    }


}

```
可以看到dubbo为UserServiceImpl生成了一个可以调用其方法，设置属性，获取属性，方法，继承Wrapper的包装对象，话说为啥要单独生成一个类，为什么不能直接通过放射的方式去调用呢？其实反射调用会耗费性能（不过虚拟机会做优化），没有直接调用来的快。

接下来，我恩继续分析上面提到的Invoker的代理生成，dubbo默认使用的代理工厂是JavassistProxyFactory，那么我们就以它为例，分析它是怎么创建invoker的

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/f9432fd247b921f7155e561d5e2ba85b.png)

从类图上，dubbo认为每个服务是一个node，并且提供一些获取服务信息的方法

> 创建Invoker

```
//proxy：接口实现类，type：接口，url：服务提供地址
public <T> Invoker<T> com.alibaba.dubbo.rpc.proxy.javassist.JavassistProxyFactory#getInvoker(T proxy, Class<T> type, URL url) {
    // TODO Wrapper cannot handle this scenario correctly: the classname contains '$'
    //这个Wrapper我们在上面用一个mian方法进行了分析，它就是通过javassist动态生成了一个继承了Wrapper的类，用于调用方法，设置属性，获取属性，方法信息
    final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
    //创建一个匿名内部类，继承AbstractProxyInvoker
    return new AbstractProxyInvoker<T>(proxy, type, url) {
        @Override
        protected Object doInvoke(T proxy, String methodName,
                                  Class<?>[] parameterTypes,
                                  Object[] arguments) throws Throwable {
            //通过Wrapper进行调用
            return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
        }
    };
}
```
一切都准备就绪了，那么下一节我开始进入服务的暴露




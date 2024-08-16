本源码版本为2.6.7

以下为dubbo的提供者配置方式

```
<!-- 提供方应用信息，用于计算依赖关系 -->
<dubbo:application name="hello-world-app"  />

<!-- 使用multicast广播注册中心暴露服务地址 -->
<dubbo:registry address="multicast://224.5.6.7:1234" />

<!-- 用dubbo协议在20880端口暴露服务 -->
<dubbo:protocol name="dubbo" port="20880" />

<!-- 声明需要暴露的服务接口 -->
<dubbo:service interface="com.dubbo.service.UserService" ref="userService" />

<!-- 和本地bean一样实现服务 -->
<bean id="userService" class="com.dubbo.service.impl.UserServiceImpl" />

```


dubbo的标签都是以dubbo作为命名空间的，对应的处理器是DubboNamespaceHandler

```
public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }

```
从上面可以看到，dubbo对应的所有的标签基本上都是使用的DubboBeanDefinitionParser解析器(以前的版本用的都是DubboBeanDefinitionParser)，他们都继承了一个叫做AbstractConfig的抽象类

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/4c0d0e9c44ffb21d83a3cb3ba8055ba4.png)

读源码在我看来，最忌讳的就是一上来就扎进源码细节中，因为这样容易迷失的代码的海洋中，我们应当先大致了解一些类的作用，比如上面的DubboBeanDefinitionParser，我们去分析的它的代码其实不难，不外乎就是解析标签，然后设置值，但是具体设置的这些config到底是干啥的呢？所以我们应先大致了解一些这些config，从名字上来看就是一些承载配置信息的bean，好了，我们先来分析下各类配置的祖宗AbstractConfig（ps：在未完整看完代码之前，我们分析某个类是干什么用的难免会遇到确确实实看不懂是用来干什么的类，这个时候别慌，先mark一下，对于这种类只能通过局部代码一步步了解）

AbstractConfig的静态常量，乍一看，好像不是很理解是啥，这个时候我们从它的名字入手，然后可以利用idea的Find useges功能查看在哪里使用到了

```
//最大长度，如果查找哪里使用了它，发现在本类的checkProperty方法中使用了，这个方法比较简单，所以很容看懂
//可以知道这个长度是用于限制字符串属性值的最大长度的
private static final int MAX_LENGTH = 200;

//最大路径长度，本类的checkPathLength方法在使用，也是检查某个属性值的最大长度
private static final int MAX_PATH_LENGTH = 200;

//这个属性从名字上看是用于匹配名字（指的啥名字），在看看哪里使用了，在本类的checkName方法中使用了
//用于正则匹配某个属性值（注意不是属性名）
private static final Pattern PATTERN_NAME = Pattern.compile("[\\-._0-9a-zA-Z]+");

//多名字匹配，这个正则与上面的PATTERN_NAME相比，只是多了一个英文逗号，说明，这是一个匹配用逗号分割的多name字符串
private static final Pattern PATTERN_MULTI_NAME = Pattern.compile("[,\\-._0-9a-zA-Z]+");

//匹配方法名
private static final Pattern PATTERN_METHOD_NAME = Pattern.compile("[a-zA-Z][0-9a-zA-Z]*");

//匹配路径名，结合我们在配置文件中配置，我们当前可以猜测它匹配的是路径名，比如上面的注册中心地址
//这个常量和MAX_PATH_LENGTH在checkPathName方法中被一起使用，你懂的
private static final Pattern PATTERN_PATH = Pattern.compile("[/\\-$._0-9a-zA-Z]+");

//匹配名字是否有symbol（符号）比如端口：8080
private static final Pattern PATTERN_NAME_HAS_SYMBOL = Pattern.compile("[:*,\\s/\\-._0-9a-zA-Z]+");

//匹配key，那这个key是啥呢？在本类的checkKey被调用，checkKey方法被其他的配置类调用
//比如com.alibaba.dubbo.config.AbstractReferenceConfig#setVersion，所以这个key可以大致认为是goup，version这类值
private static final Pattern PATTERN_KEY = Pattern.compile("[*,\\-._0-9a-zA-Z]+");
//legacy表示遗产的意思，应该指的是历史遗留属性
private static final Map<String, String> legacyProperties = new HashMap<String, String>();
//后缀
private static final String[] SUFFIXES = new String[]{"Config", "Bean"};
```

从上面的代码可以看到，dubbo提供了很多用于检查配置信息的的静态常量，我先大致了解以下他们的作用，至于正不正确，我们在后面慢慢了解，下面我们再看看到它的静态块，可以大胆猜测这里设置的是历史遗留属性与新属性的映射或者是新属性与旧属性的映射，这个属性在本类的appendProperties方法中使用到了

```
static {
        legacyProperties.put("dubbo.protocol.name", "dubbo.service.protocol");
        legacyProperties.put("dubbo.protocol.host", "dubbo.service.server.host");
        legacyProperties.put("dubbo.protocol.port", "dubbo.service.server.port");
        legacyProperties.put("dubbo.protocol.threads", "dubbo.service.max.thread.pool.size");
        legacyProperties.put("dubbo.consumer.timeout", "dubbo.service.invoke.timeout");
        legacyProperties.put("dubbo.consumer.retries", "dubbo.service.max.retry.providers");
        legacyProperties.put("dubbo.consumer.check", "dubbo.service.allow.no.provider");
        legacyProperties.put("dubbo.service.url", "dubbo.service.address");

        // this is only for compatibility 这只是为了兼容
        //这里是注册了一个虚拟机退出时的钩子函数，无外乎就是关闭一下资源，做些其他的操作
        Runtime.getRuntime().addShutdownHook(DubboShutdownHook.getDubboShutdownHook());
}
```

下面我们来分析下方法（看不懂的只能先跳过，一步一步来）

```
protected static void com.alibaba.dubbo.config.AbstractConfig#appendProperties(AbstractConfig config) {
        if (config == null) {
            return;
        }
        //我们了解过AbstractConfig的子类，这个getTagName是获取某个类的simpleName然后去掉包含Config和Bean的后缀，比如ProtocolConfig，那么它拼成的
        //名字为dubbo.protocol.
        String prefix = "dubbo." + getTagName(config.getClass()) + ".";
        //获取配置类的所有public方法
        Method[] methods = config.getClass().getMethods();
        for (Method method : methods) {
            try {
                String name = method.getName();
                //寻找set方法
                if (name.length() > 3 && name.startsWith("set") && Modifier.isPublic(method.getModifiers())
                        && method.getParameterTypes().length == 1 && isPrimitive(method.getParameterTypes()[0])) {
                        
                    //将驼峰命名的属性转成点号分割的字符串，比如serverHost -> server.host
                    String property = StringUtils.camelToSplitName(name.substring(3, 4).toLowerCase() + name.substring(4), ".");

                    String value = null;
                    //配置id（由第一节的标签解析可知，它是配置bean在spring中的beanName）
                    if (config.getId() != null && config.getId().length() > 0) {
                        //拼接
                        String pn = prefix + config.getId() + "." + property;
                        value = System.getProperty(pn);
                        if (!StringUtils.isBlank(value)) {
                            logger.info("Use System Property " + pn + " to config dubbo");
                        }
                    }
                    //如果从上面中没有获取到值，那么去掉configId尝试
                    if (value == null || value.length() == 0) {
                        String pn = prefix + property;
                        value = System.getProperty(pn);
                        if (!StringUtils.isBlank(value)) {
                            logger.info("Use System Property " + pn + " to config dubbo");
                        }
                    }
                    //如果还是为空，那么尝试从配置对象的getter中获取
                    if (value == null || value.length() == 0) {
                        Method getter;
                        try {
                            getter = config.getClass().getMethod("get" + name.substring(3));
                        } catch (NoSuchMethodException e) {
                            try {
                                getter = config.getClass().getMethod("is" + name.substring(3));
                            } catch (NoSuchMethodException e2) {
                                getter = null;
                            }
                        }
                        if (getter != null) {
                            //还是为空
                            if (getter.invoke(config) == null) {
                                //从配置文件中查找，从系统属性和系统变量中寻找Constants.DUBBO_PROPERTIES_KEY的路径
                                //所以如果我们需要定义这些属性，我们可以在java启动的时候设置系统属性，指定配置文件的路径
                                if (config.getId() != null && config.getId().length() > 0) {
                                    value = ConfigUtils.getProperty(prefix + config.getId() + "." + property);
                                }
                                if (value == null || value.length() == 0) {
                                    value = ConfigUtils.getProperty(prefix + property);
                                }
                                if (value == null || value.length() == 0) {
                                    //还是没有，那么获取历史key
                                    String legacyKey = legacyProperties.get(prefix + property);
                                    if (legacyKey != null && legacyKey.length() > 0) {
                                        //转换用遗留key获取到的值，比如dubbo.service.max.retry.providers的值会减一
                                        //dubbo.service.allow.no.provider的值进行取反，从这个方法来看，我们在上面提到的
                                        //legacyProperties的这个map集合是新属性名与旧属性名的映射
                                        value = convertLegacyValue(legacyKey, ConfigUtils.getProperty(legacyKey));
                                    }
                                }

                            }
                        }
                    }
                    if (value != null && value.length() > 0) {
                        //如果获取到了值，那么经过转换后set到配置对象中
                        method.invoke(config, convertPrimitive(method.getParameterTypes()[0], value));
                    }
                }
            } catch (Exception e) {
                logger.error(e.getMessage(), e);
            }
        }
    }
```

继续分析其他的方法

```
protected static void com.alibaba.dubbo.config.AbstractConfig#appendParameters(Map<String, String> parameters, Object config, String prefix) {
        if (config == null) {
            return;
        }
        Method[] methods = config.getClass().getMethods();
        for (Method method : methods) {
            try {
                String name = method.getName();
                //getter方法
                if ((name.startsWith("get") || name.startsWith("is"))
                        && !"getClass".equals(name)
                        && Modifier.isPublic(method.getModifiers())
                        && method.getParameterTypes().length == 0
                        && isPrimitive(method.getReturnType())) {
                    //从方法上面获取@Parameter注解
                    Parameter parameter = method.getAnnotation(Parameter.class);
                    //如果getter的返回值是Object或者这个注解标注为排除，那么跳过
                    if (method.getReturnType() == Object.class || parameter != null && parameter.excluded()) {
                        continue;
                    }
                    //要么get，要么is
                    int i = name.startsWith("get") ? 3 : 2;
                    //驼峰命名转为.号分割字符串，比如serverHost -> server.host
                    String prop = StringUtils.camelToSplitName(name.substring(i, i + 1).toLowerCase() + name.substring(i + 1), ".");
                    String key;
                    //从注解中获取key的名字
                    if (parameter != null && parameter.key().length() > 0) {
                        key = parameter.key();
                    } else {
                        //如果注解未指定，那么使用getter方法中的驼峰转点的命名
                        key = prop;
                    }
                    //调用getter方法
                    Object value = method.invoke(config);
                    String str = String.valueOf(value).trim();
                    if (value != null && str.length() > 0) {
                        //是否需要编码
                        if (parameter != null && parameter.escaped()) {
                            str = URL.encode(str);
                        }
                        //是否拼接，具体做什么的，先不管，后面再看
                        if (parameter != null && parameter.append()) {
                            String pre = parameters.get(Constants.DEFAULT_KEY + "." + key);
                            if (pre != null && pre.length() > 0) {
                                str = pre + "," + str;
                            }
                            pre = parameters.get(key);
                            if (pre != null && pre.length() > 0) {
                                str = pre + "," + str;
                            }
                        }
                        if (prefix != null && prefix.length() > 0) {
                            key = prefix + "." + key;
                        }
                        //以当前指定的key设值
                        parameters.put(key, str);
                        //如果是必须的值，如果没有值，那么将会抛错
                    } else if (parameter != null && parameter.required()) {
                        throw new IllegalStateException(config.getClass().getSimpleName() + "." + key + " == null");
                    }
                    //getParameters方法，获取属性映射
                } else if ("getParameters".equals(name)
                        && Modifier.isPublic(method.getModifiers())
                        && method.getParameterTypes().length == 0
                        && method.getReturnType() == Map.class) {
                    Map<String, String> map = (Map<String, String>) method.invoke(config, new Object[0]);
                    if (map != null && map.size() > 0) {
                        String pre = (prefix != null && prefix.length() > 0 ? prefix + "." : "");
                        for (Map.Entry<String, String> entry : map.entrySet()) {
                            parameters.put(pre + entry.getKey().replace('-', '.'), entry.getValue());
                        }
                    }
                }
            } catch (Exception e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
        }
    }
```
继续看看添加属性（appendAttributes）方法

```
protected static void appendAttributes(Map<Object, Object> parameters, Object config, String prefix) {
        if (config == null) {
            return;
        }
        Method[] methods = config.getClass().getMethods();
        for (Method method : methods) {
            try {
                String name = method.getName();
                if ((name.startsWith("get") || name.startsWith("is"))
                        && !"getClass".equals(name)
                        && Modifier.isPublic(method.getModifiers())
                        && method.getParameterTypes().length == 0
                        && isPrimitive(method.getReturnType())) {
                    Parameter parameter = method.getAnnotation(Parameter.class);
                    if (parameter == null || !parameter.attribute())
                        continue;
                    String key;
                    parameter.key();
                    if (parameter.key().length() > 0) {
                        key = parameter.key();
                    } else {
                        int i = name.startsWith("get") ? 3 : 2;
                        key = name.substring(i, i + 1).toLowerCase() + name.substring(i + 1);
                    }
                    //调用配置对象的getter方法
                    Object value = method.invoke(config);
                    if (value != null) {
                        if (prefix != null && prefix.length() > 0) {
                            key = prefix + "." + key;
                        }
                        //设置值
                        parameters.put(key, value);
                    }
                }
            } catch (Exception e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
        }
    }
```
好了，AbstractConfig就分析这么多，其他的方法，请自行了解。

dubbo大部分配置类都只是配置信息的承载体，没有什么特殊的操作，对于这些没有什么特殊操作的类，我比较需要注意的是它构造器内是否有特殊操作，其他的方法可以大致了解一下，具体的内容在看源码的过程中具体分析，这里就不进行分析了，自行去了解一遍就可以了。

我们要重点了解的就是那些实现了spring特殊接口的类，比如InitializingBean，ApplicationContextAware等等
首先我们来分析提供接口服务的bean------------------------》ServiceBean

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/7bcb0a5b254600e20de9477297f40b13.png)

从类图中可以看到ServiceBean实现了spring的
- InitializingBean bean创建-》填充属性-》初始化
- DisposableBean 容器关闭时，进行销毁
- ApplicationContextAware 在填充属性后在postProcessBeforeInitialization中设置，会把ApplicationContext实例设置到对应实现了当前接口的bean中
- ApplicationListener<ContextRefreshedEvent> 对ContextRefreshedEvent感兴趣的监听器
- BeanNameAware 属性填充后设置beanName
- ApplicationEventPublisherAware 在postProcessBeforeInitialization中设置

我们按照spring创建bean的顺序去分析这些接口对应的方法

> BeanNameAware

```
public void setBeanName(String name) {
    this.beanName = name;
}
```
没啥复杂的逻辑，仅仅是设置beanName

> ApplicationEventPublisherAware

```
public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
    this.applicationEventPublisher = applicationEventPublisher;
}
```
也没啥特殊的逻辑，仅仅是设置事件发送者（spring中是ApplicationContext）

> ApplicationContextAware

```
public void setApplicationContext(ApplicationContext applicationContext) {
    this.applicationContext = applicationContext;
    //向SpringExtensionFactory的静态变量contexts添加ApplicationContext对象
    //并向ApplicationContext注册关闭事件监听器，用于在容器关闭时关闭dubbo的资源
    SpringExtensionFactory.addApplicationContext(applicationContext);
    //向ApplicationContext注册ServiceBean（它实现了ApplicationListener接口，并对ContextRefreshedEvent感兴趣）
    //其实想serviceBean这种由spring管理的ApplicationListener会在spring执行bean的初始化后置处理中自动注册
    //如果spring的监听器的注册不是set，是list是不是这里就会重复
    supportedApplicationListener = addApplicationListener(applicationContext, this);
}
```
> InitializingBean

```
public void afterPropertiesSet() throws Exception {
    //如果没有设置提供者，那么需要查找提供者配置
    if (getProvider() == null) {
        //获取所有的提供者配置
        Map<String, ProviderConfig> providerConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProviderConfig.class, false, false);
        //找到了
        if (providerConfigMap != null && providerConfigMap.size() > 0) {
            //获取协议配置
            Map<String, ProtocolConfig> protocolConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProtocolConfig.class, false, false);
            //如果没有获取到协议配置并且存在超过一个提供者配置
            if ((protocolConfigMap == null || protocolConfigMap.size() == 0)
                    && providerConfigMap.size() > 1) { // backward compatibility
                List<ProviderConfig> providerConfigs = new ArrayList<ProviderConfig>();
                for (ProviderConfig config : providerConfigMap.values()) {
                    //获取默认的提供者配置
                    if (config.isDefault() != null && config.isDefault().booleanValue()) {
                        providerConfigs.add(config);
                    }
                }
                //将默认的提供者转换成协议配置
                //并把转换好的协议配置设置到当前serviceBean对象中
                if (!providerConfigs.isEmpty()) {
                    setProviders(providerConfigs);
                }
            } else {
                ProviderConfig providerConfig = null;
                for (ProviderConfig config : providerConfigMap.values()) {
                    if (config.isDefault() == null || config.isDefault().booleanValue()) {
                        //在存在协议配置的情况下，如果出现多个默认的提供者配置，那么抛出错误
                        if (providerConfig != null) {
                            throw new IllegalStateException("Duplicate provider configs: " + providerConfig + " and " + config);
                        }
                        providerConfig = config;
                    }
                }
                if (providerConfig != null) {
                    //设置一个提供者
                    setProvider(providerConfig);
                }
            }
        }
    }
    //如果没有设置ApplicationConfig
    if (getApplication() == null
            && (getProvider() == null || getProvider().getApplication() == null)) {
        //查找ApplicationConfig
        Map<String, ApplicationConfig> applicationConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ApplicationConfig.class, false, false);
        if (applicationConfigMap != null && applicationConfigMap.size() > 0) {
            ApplicationConfig applicationConfig = null;
            //不能出现多个默认的应用配置
            for (ApplicationConfig config : applicationConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    if (applicationConfig != null) {
                        throw new IllegalStateException("Duplicate application configs: " + applicationConfig + " and " + config);
                    }
                    applicationConfig = config;
                }
            }
            if (applicationConfig != null) {
                setApplication(applicationConfig);
            }
        }
    }
    //设置模块
    if (getModule() == null
            && (getProvider() == null || getProvider().getModule() == null)) {
        Map<String, ModuleConfig> moduleConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ModuleConfig.class, false, false);
        if (moduleConfigMap != null && moduleConfigMap.size() > 0) {
            ModuleConfig moduleConfig = null;
            for (ModuleConfig config : moduleConfigMap.values()) {
                //不能出现多个默认模块
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    if (moduleConfig != null) {
                        throw new IllegalStateException("Duplicate module configs: " + moduleConfig + " and " + config);
                    }
                    moduleConfig = config;
                }
            }
            if (moduleConfig != null) {
                setModule(moduleConfig);
            }
        }
    }
    //设置注册配置，可以多个
    if ((getRegistries() == null || getRegistries().isEmpty())
            && (getProvider() == null || getProvider().getRegistries() == null || getProvider().getRegistries().isEmpty())
            && (getApplication() == null || getApplication().getRegistries() == null || getApplication().getRegistries().isEmpty())) {
        Map<String, RegistryConfig> registryConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, RegistryConfig.class, false, false);
        if (registryConfigMap != null && registryConfigMap.size() > 0) {
            List<RegistryConfig> registryConfigs = new ArrayList<RegistryConfig>();
            for (RegistryConfig config : registryConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    registryConfigs.add(config);
                }
            }
            if (registryConfigs != null && !registryConfigs.isEmpty()) {
                super.setRegistries(registryConfigs);
            }
        }
    }
    //设置监控中心配置，只能设置一个
    if (getMonitor() == null
            && (getProvider() == null || getProvider().getMonitor() == null)
            && (getApplication() == null || getApplication().getMonitor() == null)) {
        Map<String, MonitorConfig> monitorConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, MonitorConfig.class, false, false);
        if (monitorConfigMap != null && monitorConfigMap.size() > 0) {
            MonitorConfig monitorConfig = null;
            for (MonitorConfig config : monitorConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    if (monitorConfig != null) {
                        throw new IllegalStateException("Duplicate monitor configs: " + monitorConfig + " and " + config);
                    }
                    monitorConfig = config;
                }
            }
            if (monitorConfig != null) {
                setMonitor(monitorConfig);
            }
        }
    }
    //协议配置，可以多协议
    if ((getProtocols() == null || getProtocols().isEmpty())
            && (getProvider() == null || getProvider().getProtocols() == null || getProvider().getProtocols().isEmpty())) {
        Map<String, ProtocolConfig> protocolConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProtocolConfig.class, false, false);
        if (protocolConfigMap != null && protocolConfigMap.size() > 0) {
            List<ProtocolConfig> protocolConfigs = new ArrayList<ProtocolConfig>();
            for (ProtocolConfig config : protocolConfigMap.values()) {
                if (config.isDefault() == null || config.isDefault().booleanValue()) {
                    protocolConfigs.add(config);
                }
            }
            if (protocolConfigs != null && !protocolConfigs.isEmpty()) {
                super.setProtocols(protocolConfigs);
            }
        }
    }
    //设置路径，如果beanName是接口名的前缀，那么路径设置为beanName
    if (getPath() == null || getPath().length() == 0) {
        if (beanName != null && beanName.length() > 0
                && getInterface() != null && getInterface().length() > 0
                && beanName.startsWith(getInterface())) {
            setPath(beanName);
        }
    }
    //是否延时暴露
    if (!isDelay()) {
        export();
    }
}
```

> ApplicationListener

当springrefresh完成时，会触发ContextRefreshedEvent事件

```
public void onApplicationEvent(ContextRefreshedEvent event) {
    //如果当前service被延时暴露并且还没有暴露并且允许暴露
    if (isDelay() && !isExported() && !isUnexported()) {
        if (logger.isInfoEnabled()) {
            logger.info("The service ready on spring started. service: " + getInterface());
        }
        //暴露
        export();
    }
}
```
export

```
public void com.alibaba.dubbo.config.spring.ServiceBean#export() {
    //(*1*)
    super.export();
    // Publish ServiceBeanExportedEvent
    //发布ServiceBeanExportedEvent事件
    publishExportEvent();
}

//(*1*)
public synchronized void com.alibaba.dubbo.config.ServiceConfig#export() {
    //提供者配置不为空
    if (provider != null) {
        if (export == null) {
            //是否暴露
            export = provider.getExport();
        }
        if (delay == null) {
            //延时时间
            delay = provider.getDelay();
        }
    }
    //不暴露，直接返回
    if (export != null && !export) {
        return;
    }

    if (delay != null && delay > 0) {
        //通过调度线程池调度
        delayExportExecutor.schedule(new Runnable() {
            @Override
            public void run() {
                doExport();
            }
        }, delay, TimeUnit.MILLISECONDS);
    } else {
        //暴露
        doExport();
    }
}
```
doExport

```
protected synchronized void com.alibaba.dubbo.config.ServiceConfig#doExport() {
    //取消暴露
    if (unexported) {
        throw new IllegalStateException("Already unexported!");
    }
    //已暴露
    if (exported) {
        return;
    }
    //标记为已暴露，即使后面的暴露发生错误
    exported = true;
    //必须指定接口名
    if (interfaceName == null || interfaceName.length() == 0) {
        throw new IllegalStateException("<dubbo:service interface=\"\" /> interface not allow null!");
    }
    //设置默认值，如果ProviderConfig为空，那么调用com.alibaba.dubbo.config.AbstractConfig#appendProperties方法
    //appendProperties方法将会从将属性拼接成dubbo.provider.[prefix].[beaName].thread.pool去系统属性，系统变量，指定配置文件中，自身的getter方法中找
    checkDefault();
    
    。。。。。。省略部分配置的判断与设置
    
    //引用的实现如果是泛化服务，那么设置接口类为泛化服务，所谓泛化就是在没有提供者接口jar包依赖的情况进行调用
    //只需提供对应的接口名，方法名，参数类型数组，参数即可
    if (ref instanceof GenericService) {
        //修改成GenericService接口，但是interfaceName不变，因为提供者需要根据这个名字才知道要调用哪个接口
        //GenericService有一个方法，方法有三个参数，第一个参数为目标接口方法名，第二参数是目标方法方法类型数组
        //第三个参数是目标方法参数值数组
        interfaceClass = GenericService.class;
        if (StringUtils.isEmpty(generic)) {
            //指定为泛化调用，这个generic如果设置为字符串true，那么在服务端进行参数的解析时使用的是PojoUtil助手类进行通用参数转换
            //除此之外，这个generic可以可以指定其他的两个值
            //bean和nativejava，分别表示通过dubbo的JavaBeanDescriptor与JDK的ObjectInputStream进行解码
            generic = Boolean.TRUE.toString();
        }
    } else {
        try {
            //如果不是泛化调用，解析成class对象
            interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
                    .getContextClassLoader());
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
        //检查当前类是否为接口，并且确实存在指定的要暴露的方法，否则抛错
        checkInterfaceAndMethods(interfaceClass, methods);
        //检查是否指定了接口的实现类，并且这个实现类必须是要暴露接口的实例
        checkRef();
        //非泛化调用
        generic = Boolean.FALSE.toString();
    }
    
    。。。。。。省略本地接口实现与存根实现的检查
    //如果ApplicationConfig为空，创建一个，属性从系统属性，变量，配置文件中获取
    checkApplication();
    //如果RegistryConfig为空，创建一个，属性从系统属性，变量，配置文件中获取
    checkRegistry();
    //如果RegistryConfig为空，创建一个，属性从系统属性，变量，配置文件中获取
    checkProtocol();
    //追加属性，或者覆盖属性，属性从系统属性，变量，配置文件中获取
    appendProperties(this);
    //检查是否存在存根服务，所谓存根服务就是接口的本地实现，每次在调用远程接口前都会调用这个存根服务，并且要求这个存根服务的构造参数
    //必须可以传入远程接口代理，通常用于可以缓存数据的接口，先从缓存中查看是否已经存在数据，如果没有再从传进来的远程代理调用远程接口
    checkStub(interfaceClass);
    //检查mock服务，当远程接口调用失败时调用
    //(*1*)
    checkMock(interfaceClass);
    //路径设置为接口名
    if (path == null || path.length() == 0) {
        path = interfaceName;
    }
    //暴露url
    doExportUrls();
    //getUniqueServiceName()生成唯一服务名：{group}/serviceInterface:{version}
    ProviderModel providerModel = new ProviderModel(getUniqueServiceName(), this, ref);
    //注册到ApplicationModel的静态变量ConcurrentMap<String, ProviderModel> providedServices中
    //getUniqueServiceName() -> providerModel
    ApplicationModel.initProviderModel(getUniqueServiceName(), providerModel);
}

(*1*)
void com.alibaba.dubbo.config.AbstractInterfaceConfig#checkMock(Class<?> interfaceClass) {
    if (ConfigUtils.isEmpty(mock)) {
        return;
    }
    //标准化mock字符串
    //(*2*)
    String normalizedMock = MockInvoker.normalizeMock(mock);
    //return
    if (normalizedMock.startsWith(Constants.RETURN_PREFIX)) {
        //截取掉return
        normalizedMock = normalizedMock.substring(Constants.RETURN_PREFIX.length()).trim();
        try {
            //解析mock值
            //(*3*)
            MockInvoker.parseMockValue(normalizedMock);
        } catch (Exception e) {
            throw new IllegalStateException("Illegal mock return in <dubbo:service/reference ... " +
                    "mock=\"" + mock + "\" />");
        }
        //throw
    } else if (normalizedMock.startsWith(Constants.THROW_PREFIX)) {
        //截取掉throw
        normalizedMock = normalizedMock.substring(Constants.THROW_PREFIX.length()).trim();
        if (ConfigUtils.isNotEmpty(normalizedMock)) {
            try {
                //(*4*)
                MockInvoker.getThrowable(normalizedMock);
            } catch (Exception e) {
                throw new IllegalStateException("Illegal mock throw in <dubbo:service/reference ... " +
                        "mock=\"" + mock + "\" />");
            }
        }
    } else {
        //(*5*)
        MockInvoker.getMockObject(normalizedMock, interfaceClass);
    }
}

//(*2*)
public static String com.alibaba.dubbo.rpc.support.MockInvoker#normalizeMock(String mock) {
    if (mock == null) {
        return mock;
    }

    mock = mock.trim();

    if (mock.length() == 0) {
        return mock;
    }
    //等于return
    if (Constants.RETURN_KEY.equalsIgnoreCase(mock)) {
        //returnnull
        return Constants.RETURN_PREFIX + "null";
    }
    //true default fail force
    if (ConfigUtils.isDefault(mock) || "fail".equalsIgnoreCase(mock) || "force".equalsIgnoreCase(mock)) {
        return "default";
    }
    //fail:
    if (mock.startsWith(Constants.FAIL_PREFIX)) {
        //截取掉fail:
        mock = mock.substring(Constants.FAIL_PREFIX.length()).trim();
    }
    //force:
    if (mock.startsWith(Constants.FORCE_PREFIX)) {
        //截取掉force:
        mock = mock.substring(Constants.FORCE_PREFIX.length()).trim();
    }
    //return throw
    if (mock.startsWith(Constants.RETURN_PREFIX) || mock.startsWith(Constants.THROW_PREFIX)) {
        mock = mock.replace('`', '"');
    }

    return mock;
}

//(*3*) 解析return类型的mock
public static Object parseMockValue(String mock, Type[] returnTypes) throws Exception {
    Object value = null;
    if ("empty".equals(mock)) {
        //返回类型
        value = ReflectUtils.getEmptyObject(returnTypes != null && returnTypes.length > 0 ? (Class<?>) returnTypes[0] : null);
    } else if ("null".equals(mock)) {
        //null
        value = null;
    } else if ("true".equals(mock)) {
        value = true;
    } else if ("false".equals(mock)) {
        value = false;
    } else if (mock.length() >= 2 && (mock.startsWith("\"") && mock.endsWith("\"")
            || mock.startsWith("\'") && mock.endsWith("\'"))) {
            //截取引号内容
        value = mock.subSequence(1, mock.length() - 1);
    } else if (returnTypes != null && returnTypes.length > 0 && returnTypes[0] == String.class) {
        value = mock;
    } else if (StringUtils.isNumeric(mock)) {
        value = JSON.parse(mock);
    } else if (mock.startsWith("{")) {
        //json解析成map对象
        value = JSON.parseObject(mock, Map.class);
    } else if (mock.startsWith("[")) {
        //解析为list
        value = JSON.parseObject(mock, List.class);
    } else {
        //其他的直接设置
        value = mock;
    }
    //如果指定了返回值类型
    if (returnTypes != null && returnTypes.length > 0) {
        //将值转化成对应指定的返回值类型，returnTypes[0]：值类型，returnTypes[1]：泛型类型
        value = PojoUtils.realize(value, (Class<?>) returnTypes[0], returnTypes.length > 1 ? returnTypes[1] : null);
    }
    //返回
    return value;
}

 //(*4*)
 public static Throwable com.alibaba.dubbo.rpc.support.MockInvoker#getThrowable(String throwstr) {
    //从缓存中获取
    Throwable throwable = throwables.get(throwstr);
    if (throwable != null) {
        return throwable;
    }
    //反射构建异常对象
    try {
        Throwable t;
        Class<?> bizException = ReflectUtils.forName(throwstr);
        Constructor<?> constructor;
        constructor = ReflectUtils.findConstructor(bizException, String.class);
        t = (Throwable) constructor.newInstance(new Object[]{"mocked exception for service degradation."});
        if (throwables.size() < 1000) {
            throwables.put(throwstr, t);
        }
        return t;
    } catch (Exception e) {
        throw new RpcException("mock throw error :" + throwstr + " argument error.", e);
    }
}

//(*5*)
public static Object com.alibaba.dubbo.rpc.support.MockInvoker#getMockObject(String mockService, Class serviceType) {
    //mock为true，default，force，fail
    if (ConfigUtils.isDefault(mockService)) {
        //接口拼上Mock，比如提供的服务叫UserService，那么其mock服务为UserServiceMock
        mockService = serviceType.getName() + "Mock";
    }

    Class<?> mockClass = ReflectUtils.forName(mockService);
    if (!serviceType.isAssignableFrom(mockClass)) {
        throw new IllegalStateException("The mock class " + mockClass.getName() +
                " not implement interface " + serviceType.getName());
    }

    try {
        //创建mock服务，用于测试
        return mockClass.newInstance();
    } catch (InstantiationException e) {
        throw new IllegalStateException("No default constructor from mock class " + mockClass.getName(), e);
    } catch (IllegalAccessException e) {
        throw new IllegalStateException(e);
    }
}
```
篇幅有点长了，我们在下一个小节分析doExportUrls()方法












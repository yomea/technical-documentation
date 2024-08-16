本源码版本为2.6.7

比如有以下配置

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
dubbo提供的标签解析的命名空间为DubboBeanDefinitionParser

```
@Override
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
解析方法

```
public BeanDefinition com.alibaba.dubbo.config.spring.schema.DubboBeanDefinitionParser#parse(Element element, ParserContext parserContext) {
        return parse(element, parserContext, beanClass, required);
}
                                                                |
                                                                V
private static BeanDefinition com.alibaba.dubbo.config.spring.schema.DubboBeanDefinitionParser#parse(Element element, ParserContext parserContext, Class<?> beanClass, boolean required) {
    //创建bean定义
    RootBeanDefinition beanDefinition = new RootBeanDefinition();
    //将传递进来的bean设置进去，dubbo根据不同标签设置不同的bean
    beanDefinition.setBeanClass(beanClass);
    //设置为非懒加载
    beanDefinition.setLazyInit(false);
    //获取beanName
    String id = element.getAttribute("id");
    //如果为空，并且是必须的，那么需要生成beanName
    if ((id == null || id.length() == 0) && required) {
        String generatedBeanName = element.getAttribute("name");
        if (generatedBeanName == null || generatedBeanName.length() == 0) {
            //如果是协议配置，那么将beanName设置成dubbo
            if (ProtocolConfig.class.equals(beanClass)) {
                generatedBeanName = "dubbo";
            } else {
                generatedBeanName = element.getAttribute("interface");
            }
        }
        if (generatedBeanName == null || generatedBeanName.length() == 0) {
            //设置为bean的className
            generatedBeanName = beanClass.getName();
        }
        id = generatedBeanName;
        int counter = 2;
        //是否已经包含这个beanName，包含的进行递增计数
        while (parserContext.getRegistry().containsBeanDefinition(id)) {
            id = generatedBeanName + (counter++);
        }
    }
    if (id != null && id.length() > 0) {
        //重复的beanName
        if (parserContext.getRegistry().containsBeanDefinition(id)) {
            throw new IllegalStateException("Duplicate spring bean id " + id);
        }
        //注册这个BeanDefinition
        parserContext.getRegistry().registerBeanDefinition(id, beanDefinition);
        //设置这个beanName到对应的bean中
        beanDefinition.getPropertyValues().addPropertyValue("id", id);
    }
    //上面是公用方法设置，我感觉可以单独抽成一个方法
    //下面是针对不同标签的特殊解析
    
    //解析protocol标签时进入
    if (ProtocolConfig.class.equals(beanClass)) {
        //循环所有已经注册的BeanDefinition
        for (String name : parserContext.getRegistry().getBeanDefinitionNames()) {
            BeanDefinition definition = parserContext.getRegistry().getBeanDefinition(name);
            //寻找拥有protocol属性的BeanDefinition
            PropertyValue property = definition.getPropertyValues().getPropertyValue("protocol");
            if (property != null) {
                Object value = property.getValue();
                //对于属性值直接为ProtocolConfig类型切对应的协议名和当前的id一样的，那么重新设置为spring的RuntimeBeanReference类型
                //便于spring能够自动注入当前创建的ProtocolConfig
                if (value instanceof ProtocolConfig && id.equals(((ProtocolConfig) value).getName())) {
                    definition.getPropertyValues().addPropertyValue("protocol", new RuntimeBeanReference(id));
                }
            }
        }
        //解析service标签时会进入这个方法
    } else if (ServiceBean.class.equals(beanClass)) {
        //获取service标签上的class属性，这个class指定的是当前service的实现类
        String className = element.getAttribute("class");
        if (className != null && className.length() > 0) {
            //创建一个BeanDefinition
            RootBeanDefinition classDefinition = new RootBeanDefinition();
            classDefinition.setBeanClass(ReflectUtils.forName(className));
            classDefinition.setLazyInit(false);
            //解析property标签，也就是给这个服务实现了adddPropertyValue
            parseProperties(element.getChildNodes(), classDefinition);
            //给当前ServiceBean添加ref属性值
            beanDefinition.getPropertyValues().addPropertyValue("ref", new BeanDefinitionHolder(classDefinition, id + "Impl"));
        }
        //内部可以配置多个service标签，provider标签提供内部多个提供者的默认配置
    } else if (ProviderConfig.class.equals(beanClass)) {
        //(*1*)
        parseNested(element, parserContext, ServiceBean.class, true, "service", "provider", id, beanDefinition);
        //消费者默认配置
    } else if (ConsumerConfig.class.equals(beanClass)) {
        //(*2*)
        parseNested(element, parserContext, ReferenceBean.class, false, "reference", "consumer", id, beanDefinition);
    }
    Set<String> props = new HashSet<String>();
    ManagedMap parameters = null;
    for (Method setter : beanClass.getMethods()) {
        String name = setter.getName();
        //寻找setter方法
        if (name.length() > 3 && name.startsWith("set")
                && Modifier.isPublic(setter.getModifiers())
                && setter.getParameterTypes().length == 1) {
            Class<?> type = setter.getParameterTypes()[0];
            String propertyName = name.substring(3, 4).toLowerCase() + name.substring(4);
            //驼峰命名转换成短横线命名，一般java中的属性名映射到xml元素的属性时都是以短横线分割，比如factoryBean -> factory-bean
            String property = StringUtils.camelToSplitName(propertyName, "-");
            props.add(property);
            Method getter = null;
            try {
                //获取getter方法
                getter = beanClass.getMethod("get" + name.substring(3), new Class<?>[0]);
            } catch (NoSuchMethodException e) {
                try {
                    getter = beanClass.getMethod("is" + name.substring(3), new Class<?>[0]);
                } catch (NoSuchMethodException e2) {
                }
            }
            //忽略不是public或者类型不和setter参数匹配的方法
            if (getter == null
                    || !Modifier.isPublic(getter.getModifiers())
                    || !type.equals(getter.getReturnType())) {
                continue;
            }
            //属性名为parameters，解析<parameter key='' value='' hide='true'></parameter>
            //如果hide属性为true，那么在put到parameters的时候会在key前面添加.号
            if ("parameters".equals(property)) {
                parameters = parseParameters(element.getChildNodes(), beanDefinition);
                
                //解析method标签，生成对应的配置对象为MethodConfig
            } else if ("methods".equals(property)) {
                parseMethods(id, element.getChildNodes(), beanDefinition, parserContext);
                
                //argument 对应的配置对象为ArgumentConfig
            } else if ("arguments".equals(property)) {
                parseArguments(id, element.getChildNodes(), beanDefinition, parserContext);
            } else {
                //处理其他的属性
                String value = element.getAttribute(property);
                if (value != null) {
                    value = value.trim();
                    if (value.length() > 0) {
                        //属性为registry并且值为N/A（不可用）
                        if ("registry".equals(property) && RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(value)) {
                            //设置注册地址不可用
                            RegistryConfig registryConfig = new RegistryConfig();
                            registryConfig.setAddress(RegistryConfig.NO_AVAILABLE);
                            beanDefinition.getPropertyValues().addPropertyValue(property, registryConfig);
                        } else if ("registry".equals(property) && value.indexOf(',') != -1) {
                            //解析value值，多个值逗号分割，设置到registries属性中，ManageList<RuntimeBeanReference>
                            parseMultiRef("registries", value, beanDefinition, parserContext);
                        } else if ("provider".equals(property) && value.indexOf(',') != -1) {
                            //设置多个提供者引用，具体对一个的setProviders实际上是从这些ProviderConfig配置中获取
                            //ProtocolConfig，这个方法被废弃，所以这个配置已经不建议使用，请使用下面的protocols协议配置
                            parseMultiRef("providers", value, beanDefinition, parserContext);
                        } else if ("protocol".equals(property) && value.indexOf(',') != -1) {
                            //设置多个协议配置引用
                            parseMultiRef("protocols", value, beanDefinition, parserContext);
                        } else {
                            Object reference;
                            //是否为简单类型
                            if (isPrimitive(type)) {
                                //对于以下组合的属性和值，将当前value设置为null
                                if ("async".equals(property) && "false".equals(value)
                                        || "timeout".equals(property) && "0".equals(value)
                                        || "delay".equals(property) && "0".equals(value)
                                        || "version".equals(property) && "0.0.0".equals(value)
                                        || "stat".equals(property) && "-1".equals(value)
                                        || "reliable".equals(property) && "false".equals(value)) {
                                    // backward compatibility for the default value in old version's xsd
                                    value = null;
                                }
                                reference = value;
                                //协议属性
                            } else if ("protocol".equals(property)
                                    //检查是否存在Protocol接口的实现，dubbo通过@SPI来获取实现类，这个value此时表示spi的扩展名
                                    //查看Protocol接口，可以看到接口被@SPI注解修饰，默认的扩展名为dubbo
                                    && ExtensionLoader.getExtensionLoader(Protocol.class).hasExtension(value)
                                    //不包含或者包含的对应bean不是ProtocolConfig
                                    && (!parserContext.getRegistry().containsBeanDefinition(value)
                                    || !ProtocolConfig.class.getName().equals(parserContext.getRegistry().getBeanDefinition(value).getBeanClassName()))) {
                                    //如果当前标签使用的是dubbo:provider标签配置协议，那么提示使用dubbo:protocol代替
                                if ("dubbo:provider".equals(element.getTagName())) {
                                    logger.warn("Recommended replace <dubbo:provider protocol=\"" + value + "\" ... /> to <dubbo:protocol name=\"" + value + "\" ... />");
                                }
                                // backward compatibility
                                //因为上面检测不存在value指定的ProtocolConfig，那么为了向后兼容，就创建一个
                                ProtocolConfig protocol = new ProtocolConfig();
                                protocol.setName(value);
                                reference = protocol;
                                //返回时调用方法
                            } else if ("onreturn".equals(property)) {
                                int index = value.lastIndexOf(".");
                                String returnRef = value.substring(0, index);
                                String returnMethod = value.substring(index + 1);
                                //对应方法的bean引用
                                reference = new RuntimeBeanReference(returnRef);
                                //当返回时调用的方法
                                beanDefinition.getPropertyValues().addPropertyValue("onreturnMethod", returnMethod);
                            } else if ("onthrow".equals(property)) {
                                int index = value.lastIndexOf(".");
                                String throwRef = value.substring(0, index);
                                String throwMethod = value.substring(index + 1);
                                reference = new RuntimeBeanReference(throwRef);
                                //抛出异常时的处理方法
                                beanDefinition.getPropertyValues().addPropertyValue("onthrowMethod", throwMethod);
                            } else if ("oninvoke".equals(property)) {
                                int index = value.lastIndexOf(".");
                                String invokeRef = value.substring(0, index);
                                String invokeRefMethod = value.substring(index + 1);
                                reference = new RuntimeBeanReference(invokeRef);
                                //调用时的处理方法
                                beanDefinition.getPropertyValues().addPropertyValue("oninvokeMethod", invokeRefMethod);
                            } else {
                                if ("ref".equals(property) && parserContext.getRegistry().containsBeanDefinition(value)) {
                                    BeanDefinition refBean = parserContext.getRegistry().getBeanDefinition(value);
                                    if (!refBean.isSingleton()) {
                                        throw new IllegalStateException("The exported service ref " + value + " must be singleton! Please set the " + value + " bean scope to singleton, eg: <bean id=\"" + value + "\" scope=\"singleton\" ...>");
                                    }
                                }
                                reference = new RuntimeBeanReference(value);
                            }
                            //设置应用
                            beanDefinition.getPropertyValues().addPropertyValue(propertyName, reference);
                        }
                    }
                }
            }
        }
    }
    //处理没有getter方法的属性，这些属性都被认为是自定义属性
    NamedNodeMap attributes = element.getAttributes();
    int len = attributes.getLength();
    for (int i = 0; i < len; i++) {
        Node node = attributes.item(i);
        String name = node.getLocalName();
        if (!props.contains(name)) {
            if (parameters == null) {
                parameters = new ManagedMap();
            }
            String value = node.getNodeValue();
            parameters.put(name, new TypedStringValue(value, String.class));
        }
    }
    if (parameters != null) {
        //添加自定义属性
        beanDefinition.getPropertyValues().addPropertyValue("parameters", parameters);
    }
    return beanDefinition;
}

//(*1*)
//element, parserContext, ServiceBean.class, true, "service", "provider", id, beanDefinition
private static void parseNested(Element element, ParserContext parserContext, Class<?> beanClass, boolean required, String tag, String property, String ref, BeanDefinition beanDefinition) {
    NodeList nodeList = element.getChildNodes();
    if (nodeList != null && nodeList.getLength() > 0) {
        boolean first = true;
        for (int i = 0; i < nodeList.getLength(); i++) {
            Node node = nodeList.item(i);
            if (node instanceof Element) {
                //service标签
                if (tag.equals(node.getNodeName())
                        || tag.equals(node.getLocalName())) {
                    //是否是读取到的第一个service标签
                    if (first) {
                        first = false;
                        //解析当前provider标签，判断当前的提供组是否为默认的，默认配置可以使用在那些没有被provider标签包裹的提供者
                        //当时不能出现多个默认配置，否者在服务启动的时候就会报错
                        String isDefault = element.getAttribute("default");
                        //是否为默认的提供者
                        if (isDefault == null || isDefault.length() == 0) {
                            beanDefinition.getPropertyValues().addPropertyValue("default", "false");
                        }
                    }
                    //递归调用DubboBeanDefinitionParser的parse，继续解析service标签
                    BeanDefinition subDefinition = parse((Element) node, parserContext, beanClass, required);
                    if (subDefinition != null && ref != null && ref.length() > 0) {
                        //serviceBean引用ProviderConfig配置
                        subDefinition.getPropertyValues().addPropertyValue(property, new RuntimeBeanReference(ref));
                    }
                }
            }
        }
    }
}

//(*2*)
//element, parserContext, ReferenceBean.class, false, "reference", "consumer", id, beanDefinition
private static void parseNested(Element element, ParserContext parserContext, Class<?> beanClass, boolean required, String tag, String property, String ref, BeanDefinition beanDefinition) {
    NodeList nodeList = element.getChildNodes();
    if (nodeList != null && nodeList.getLength() > 0) {
        boolean first = true;
        for (int i = 0; i < nodeList.getLength(); i++) {
            Node node = nodeList.item(i);
            if (node instanceof Element) {
                //解析标签reference
                if (tag.equals(node.getNodeName())
                        || tag.equals(node.getLocalName())) {
                    if (first) {
                        first = false;
                        //解析consumer标签上，判断当前的消费是否是默认的，用于那些不被consumer标签包裹的，游离的消费者
                        //ConsumerConfig默认配置只能有一个，不能出现多个，否在消费者启动的时候就会抛出错误
                        String isDefault = element.getAttribute("default");
                        if (isDefault == null || isDefault.length() == 0) {
                            beanDefinition.getPropertyValues().addPropertyValue("default", "false");
                        }
                    }
                    //创建ReferenceBean对象
                    BeanDefinition subDefinition = parse((Element) node, parserContext, beanClass, required);
                    if (subDefinition != null && ref != null && ref.length() > 0) {
                        //引用consumer
                        subDefinition.getPropertyValues().addPropertyValue(property, new RuntimeBeanReference(ref));
                    }
                }
            }
        }
    }
}

```

dubbo的标签解析就到这里就结束了，接下来就是具体分析配置类了
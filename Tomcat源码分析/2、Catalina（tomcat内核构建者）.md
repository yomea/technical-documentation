<p>
&nbsp;&nbsp;&nbsp;&nbsp;从上一节中，我们发现Bootstrap的加载和启动方法实际调用的是Catalina对象的加载和启动方法，而Catalina使用的catalina加载器加载，本节就来分析下Catalina的load和start方法，到底做了什么？首先我们看到Catalina的load方法
</p>


```
public void load() {
        //如果已经加载过了，那么不再重新加载
        if (loaded) {
            return;
        }
        loaded = true;

        long t1 = System.nanoTime();
        
        //从名字上看是初始化目录，但实际上这个方法里面只是检验一下
        //系统属性中是否有指定java.io.tmpdir的目录，如果没有，那么打印错误日志
        initDirs();

        // Before digester - it may be needed
        //初始化JNDI相关的包目录和用于初始化的InitialContextFactory实现类
        //设置到了系统属性中，以便随时可以获取
        initNaming();

        // Create and execute our Digester
        //创建Digester，这个类继承了DefaultHandler2类，这个类是用于sax解析xml使用的处理器
        Digester digester = createStartDigester();

       。。。。。。省略读取配置文件的逻辑代码
    }
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;创建Digester,首先来看下它的静态块做了什么
</p>


```
static {
        //从系统中获取定义在系统属性中的属性资源类名
        String className = System.getProperty("org.apache.tomcat.util.digester.PROPERTY_SOURCE");
        IntrospectionUtils.PropertySource source = null;
        if (className != null) {
            //设置两个加载器，一个是加载Digester的加载器，另一个是tomcat设置
            //在线程上下文的加载器，通常是common加载器
            ClassLoader[] cls = new ClassLoader[] { Digester.class.getClassLoader(),
                    Thread.currentThread().getContextClassLoader() };
            //加载属性资源对象
            for (int i = 0; i < cls.length; i++) {
                try {
                    Class<?> clazz = Class.forName(className, true, cls[i]);
                    source = (IntrospectionUtils.PropertySource)
                            clazz.getConstructor().newInstance();
                    break;
                } catch (Throwable t) {
                    ExceptionUtils.handleThrowable(t);
                    LogFactory.getLog("org.apache.tomcat.util.digester.Digester")
                            .error("Unable to load property source[" + className + "].", t);
                }
            }
        }
        if (source != null) {
            propertySource = source;
            propertySourceSet = true;
        }
        if (Boolean.getBoolean("org.apache.tomcat.util.digester.REPLACE_SYSTEM_PROPERTIES")) {
            replaceSystemProperties();
        }
    }
```
在创建Digester时，tomcat首先准备属性资源，以便后面在解析配置文件的时候能够进行占位符的替换，Digester继承了DefaultHandler2，所以Digester是用于sax解析xml的处理器，以下就是Digester的类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/a0d406ff6d52930216e31ea3597ba8f1.png)

- PropertySource：提供属性资源的对象，getProperty(String)
- rules：规则集合，tomcat目前的唯一实现类RulesBase，实现的基础集合使用的是一个HashMap，从类图中可以看到
- rule：具体的规则对象，实现的规则类型有ObjectCreateRule，CallMethodRule，CallParamRule等

以下就是创建Digester规程，可以看到tomcat给这个创建的Digester添加了许多的规则

```
protected Digester createStartDigester() {
        long t1=System.currentTimeMillis();
        // Initialize the digester
        Digester digester = new Digester();
        digester.setValidating(false);
        digester.setRulesValidation(true);
        HashMap<Class<?>, List<String>> fakeAttributes = new HashMap<>();
        ArrayList<String> attrs = new ArrayList<>();
        attrs.add("className");
        fakeAttributes.put(Object.class, attrs);
        digester.setFakeAttributes(fakeAttributes);
        digester.setUseContextClassLoader(true);
        /**
         * digester  ruleBase-->List<rule> , cache->Map<pattern, List<rule>>
         */
        // Configure the actions we will be using
        //
        digester.addObjectCreate("Server",
                                 "org.apache.catalina.core.StandardServer",
                                 "className");
        digester.addSetProperties("Server");
        digester.addSetNext("Server",
                            "setServer",
                            "org.apache.catalina.Server");

        。。。。。。省略大量添加规则的逻辑
        return (digester);

    }
```

上面一堆的代码，添加了一堆的rule，那么这些rule是做什么用的呢？为啥为何一个用于xml解析的解析器进行关联呢？下面我们先来看看rule与ruleSet的类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/db50c06ec605dac62edcfee4f34c0903.png)

rule暂时的作用先不讨论，但ruleSet我们可以从当前的代码中看出，它就是用于批量注册rule到digester的规则集，每个具体的规则集实现基本上都是硬编码的，当然如果让你自己实现，你可以写成从某个配置读取，这样具有一定的动态添加规则的效果。

到这里xml的处理器已经创建好了，规则也已经设置好了，那么我们就可以开始解析配置文件了，一下就是load方法中创建digester之后的逻辑


```
        InputSource inputSource = null;
        InputStream inputStream = null;
        File file = null;
        try {
            try {
                //获取配置文件
                /**
                 *    protected File configFile() {
                 *      //configFile = "conf/server.xml"
                 *      File file = new File(configFile);
                 *      if (!file.isAbsolute()) {
                 *           file = new File(Bootstrap.getCatalinaBase(), configFile);
                 *      }
                 *       return (file);
                 *
                 *  }
                 */
                 file = configFile();
                inputStream = new FileInputStream(file);
                inputSource = new InputSource(file.toURI().toURL().toString());
            } catch (Exception e) {
                if (log.isDebugEnabled()) {
                    log.debug(sm.getString("catalina.configFail", file), e);
                }
            }
            //如果没有，那么就从类加载器的加载路径获取
            if (inputStream == null) {
                try {
                    inputStream = getClass().getClassLoader()
                        .getResourceAsStream(getConfigFile());
                    inputSource = new InputSource
                        (getClass().getClassLoader()
                         .getResource(getConfigFile()).toString());
                } catch (Exception e) {
                    if (log.isDebugEnabled()) {
                        log.debug(sm.getString("catalina.configFail",
                                getConfigFile()), e);
                    }
                }
            }

            // This should be included in catalina.jar
            // Alternative: don't bother with xml, just create it manually.
            //如果还是没有就加载加载器路径下的嵌入的配置文件server-embed.xml
            if (inputStream == null) {
                try {
                    inputStream = getClass().getClassLoader()
                            .getResourceAsStream("server-embed.xml");
                    inputSource = new InputSource
                    (getClass().getClassLoader()
                            .getResource("server-embed.xml").toString());
                } catch (Exception e) {
                    if (log.isDebugEnabled()) {
                        log.debug(sm.getString("catalina.configFail",
                                "server-embed.xml"), e);
                    }
                }
            }

            //如果都没有，那么就只能打印日志并返回
            if (inputStream == null || inputSource == null) {
                if  (file == null) {
                    log.warn(sm.getString("catalina.configFail",
                            getConfigFile() + "] or [server-embed.xml]"));
                } else {
                    log.warn(sm.getString("catalina.configFail",
                            file.getAbsolutePath()));
                    if (file.exists() && !file.canRead()) {
                        log.warn("Permissions incorrect, read permission is not allowed on the file.");
                    }
                }
                return;
            }

            try {
                inputSource.setByteStream(inputStream);
                //将当前Catalina对象压入digester的堆栈
                digester.push(this);
                //开始解析配置文件
                digester.parse(inputSource);
            } catch (SAXParseException spe) {
                log.warn("Catalina.start using " + getConfigFile() + ": " +
                        spe.getMessage());
                return;
            } catch (Exception e) {
                log.warn("Catalina.start using " + getConfigFile() + ": " , e);
                return;
            }
        } finally {
            if (inputStream != null) {
                try {
                    inputStream.close();
                } catch (IOException e) {
                    // Ignore
                }
            }
        }
        //给server对象设置Catalina对象
        getServer().setCatalina(this);
        //设置基础路径
        getServer().setCatalinaHome(Bootstrap.getCatalinaHomeFile());
        getServer().setCatalinaBase(Bootstrap.getCatalinaBaseFile());

        // Stream redirection
        //重定向输出流
        initStreams();

        // Start the new server
        try {
            //开始启动服务
            getServer().init();
        } catch (LifecycleException e) {
            if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
                throw new java.lang.Error(e);
            } else {
                log.error("Catalina.start", e);
            }
        }

    。。。。。。省略打印启动日志代码
```

接下来我们就来看看解析配置文件的逻辑，前面说过Digester是继承了sax解析xml的处理器类DefaultHandler2，那么有三个方法是我们需要着重进行分析的，它们都来自同一个接口ContentHandler


```
//解析引擎解析到某个元素的开始标签时调用
public void startElement (String uri, String localName,
                              String qName, Attributes atts)
        throws SAXException;
        
//解析引擎解析到标签的结尾开始调用
public void endElement (String uri, String localName,
                            String qName)
        throws SAXException;
        
//解析引擎解析到标签体文本时开始调用
public void characters (char ch[], int start, int length)
        throws SAXException;
```

下面就是Disgester重写的startElement方法


```
//namespaceURI命名空间uri，localName标签的本地名，比如<aop:config>，localName就是aop，qName就是config
//list就是解析的当前标签的属性列表
public void startElement(String namespaceURI, String localName, String qName, Attributes list)
            throws SAXException {
        boolean debug = log.isDebugEnabled();

        if (saxLog.isDebugEnabled()) {
            saxLog.debug("startElement(" + namespaceURI + "," + localName + "," + qName + ")");
        }

        // Parse system properties
        //更新属性，解析${}占位符，替换成真正的属性值
        list = updateAttributes(list);

        // Save the body text accumulated for our surrounding element
        //bodyTexts是一个堆栈，Disgester的成员变量，bodyText是StringBuilder对象，也是成员变量
        //bodyText在characters方法中添加
        bodyTexts.push(bodyText);
        bodyText = new StringBuilder();

        // the actual element name is either in localName or qName, depending
        // on whether the parser is namespace aware
        String name = localName;
        if ((name == null) || (name.length() < 1)) {
            name = qName;
        }

        // Compute the current matching rule
        //match就是标签的名字用/进行连接的字符串
        StringBuilder sb = new StringBuilder(match);
        if (match.length() > 0) {
            sb.append('/');
        }
        sb.append(name);
        match = sb.toString();
        if (debug) {
            log.debug("  New match='" + match + "'");
        }

        // Fire "begin" events for all relevant rules
        //匹配规则组，就是从rulesBase中的HashMap中查找
        List<Rule> rules = getRules().match(namespaceURI, match);
        //压栈保存
        matches.push(rules);
        //循环调用规则的begin方法
        if ((rules != null) && (rules.size() > 0)) {
            for (int i = 0; i < rules.size(); i++) {
                try {
                    Rule rule = rules.get(i);
                    if (debug) {
                        log.debug("  Fire begin() for " + rule);
                    }
                    //调用规则的begin方法
                    rule.begin(namespaceURI, name, list);
                } catch (Exception e) {
                    log.error("Begin event threw exception", e);
                    throw createSAXException(e);
                } catch (Error e) {
                    log.error("Begin event threw error", e);
                    throw e;
                }
            }
        } else {
            if (debug) {
                log.debug("  No rules found matching '" + match + "'.");
            }
        }

    }
```

下面是解析标签体的普通文本调用的方法

```
public void characters(char buffer[], int start, int length) throws SAXException {

        if (saxLog.isDebugEnabled()) {
            saxLog.debug("characters(" + new String(buffer, start, length) + ")");
        }
        //连接标签内的文本
        bodyText.append(buffer, start, length);

    }
```

看完了解析开始标签和解析标签体文本的逻辑，我们继续看下解析结束标签的逻辑


```
public void endElement(String namespaceURI, String localName, String qName)
            throws SAXException {

        boolean debug = log.isDebugEnabled();

        if (debug) {
            if (saxLog.isDebugEnabled()) {
                saxLog.debug("endElement(" + namespaceURI + "," + localName + "," + qName + ")");
            }
            log.debug("  match='" + match + "'");
            log.debug("  bodyText='" + bodyText + "'");
        }

        // Parse system properties
        //更新标签体内容，解析${}占位符，用属性资源中的值进行替换
        bodyText = updateBodyText(bodyText);

        // the actual element name is either in localName or qName, depending
        // on whether the parser is namespace aware
        String name = localName;
        if ((name == null) || (name.length() < 1)) {
            name = qName;
        }

        // Fire "body" events for all relevant rules
        //在解析开始标签的时候如果寻找了对应的规则，那么会被压入matches堆栈
        //这里讲这些被找到的规则弹出
        List<Rule> rules = matches.pop();
        if ((rules != null) && (rules.size() > 0)) {
            String bodyText = this.bodyText.toString();
            for (int i = 0; i < rules.size(); i++) {
                try {
                    Rule rule = rules.get(i);
                    if (debug) {
                        log.debug("  Fire body() for " + rule);
                    }
                    //调用规则的body方法
                    rule.body(namespaceURI, name, bodyText);
                } catch (Exception e) {
                    log.error("Body event threw exception", e);
                    throw createSAXException(e);
                } catch (Error e) {
                    log.error("Body event threw error", e);
                    throw e;
                }
            }
        } else {
            if (debug) {
                log.debug("  No rules found matching '" + match + "'.");
            }
            if (rulesValidation) {
                log.warn("  No rules found matching '" + match + "'.");
            }
        }

        // Recover the body text from the surrounding element
        //恢复上一元素的文本内容
        bodyText = bodyTexts.pop();

        // Fire "end" events for all relevant rules in reverse order
        //调用当前元素匹配的规则的end方法
        if (rules != null) {
            for (int i = 0; i < rules.size(); i++) {
                int j = (rules.size() - i) - 1;
                try {
                    Rule rule = rules.get(j);
                    if (debug) {
                        log.debug("  Fire end() for " + rule);
                    }
                    rule.end(namespaceURI, name);
                } catch (Exception e) {
                    log.error("End event threw exception", e);
                    throw createSAXException(e);
                } catch (Error e) {
                    log.error("End event threw error", e);
                    throw e;
                }
            }
        }

        // Recover the previous match expression
        //恢复上一元素的匹配表达式
        int slash = match.lastIndexOf('/');
        //如果当前解析的标签是host，那么其表达式是Server/Service/Engine/Host
        //上一元素的表达式为Server/Service/Engine
        if (slash >= 0) {
            match = match.substring(0, slash);
        } else {
            match = "";
        }

    }
```

从上面的代码中我可以看到，规则就是用于辅助解析xml内容的一种解析方式，下面我们挑选一下规则进行分析

> ObjectCreateRule

首先看到begin方法

```
 public void begin(String namespace, String name, Attributes attributes)
            throws Exception {

        // Identify the name of the class to instantiate
        //类名，在构建Digester时，构建的其中一个规则中，tomcat传入了一个类名：org.apache.catalina.core.StandardServer
        String realClassName = className;
        //如果传入了属性名，那么可以通过属性名指定的属性值进行覆盖传入的值
        if (attributeName != null) {
            String value = attributes.getValue(attributeName);
            if (value != null) {
                realClassName = value;
            }
        }
        if (digester.log.isDebugEnabled()) {
            digester.log.debug("[ObjectCreateRule]{" + digester.match +
                    "}New " + realClassName);
        }

        if (realClassName == null) {
            throw new NullPointerException("No class name specified for " +
                    namespace + " " + name);
        }

        // Instantiate the new object and push it on the context stack
        //在前面的分析中，我知道加载Catalina的加载器是Catalina加载器，并且tomcat将线程的上下文的加载器设置成了
        //Catalina，所以加载这些xml指定的类也是通过这个加载器加载的。
        Class<?> clazz = digester.getClassLoader().loadClass(realClassName);
        Object instance = clazz.getConstructor().newInstance();
        //压入栈，注意这个栈中在创建digester的时候已经压入过一个Catalina的实例
        digester.push(instance);
    }
```
然后再看body方法，当前这个规则没有对这个方法进行重写，沿用的是父类Rule的默认方法，这个方法没有做任何事情，是个空方法

最后我直接看到end方法


```
public void end(String namespace, String name) throws Exception {
        //弹出当前规则创建的对象
        Object top = digester.pop();
        if (digester.log.isDebugEnabled()) {
            digester.log.debug("[ObjectCreateRule]{" + digester.match +
                    "} Pop " + top.getClass().getName());
        }

    }
```
上面的end方法只是弹出它创建的对象，之后就打印了以下debug的日志，然后就没有然后了。

下面我们继续看些别的规则吧

> SetPropertiesRule

SetPropertiesRule只重写了begin方法，其他的都是默认的空方法


```
public void begin(String namespace, String theName, Attributes attributes)
            throws Exception {

        // Populate the corresponding properties of the top object
        //获取栈顶数据，如果连着上面的ObjectCreateRule的话，那么这里获取的就是上面创建的对象的引用
        Object top = digester.peek();
        。。。。。。省略部分日志打印逻辑
        for (int i = 0; i < attributes.getLength(); i++) {
            //获取属性名
            String name = attributes.getLocalName(i);
            if ("".equals(name)) {
                name = attributes.getQName(i);
            }
            String value = attributes.getValue(i);
            
           。。。。。。省略部分日志代码
           
           //排除不需要进行操作的属性，在创建Digester的时候存入的那个属性是className，所以如果这个属性是className
           //那么就会被跳过，如果是其他属性那么就会调用堆栈中弹出来的对象的属性设置方法（setter方法）设置值
            if (!digester.isFakeAttribute(top, name)
                    && !IntrospectionUtils.setProperty(top, name, value)
                    && digester.getRulesValidation()) {//如果设置失败的话，问是否需要规则教养，如果需要那么打印警告日志
                digester.log.warn("[SetPropertiesRule]{" + digester.match +
                        "} Setting property '" + name + "' to '" +
                        value + "' did not find a matching property.");
            }
        }

    }
```
从上面的代码来看，这个SetPropertiesRule规则只是用于设置属性的规则。

> SetNextRule

这个规则只重写了end方法

```
public void end(String namespace, String name) throws Exception {

        // Identify the objects to be used
        //获取栈顶元素
        Object child = digester.peek(0);
        //获取栈顶的下一个元素
        Object parent = digester.peek(1);
        if (digester.log.isDebugEnabled()) {
            if (parent == null) {
                digester.log.debug("[SetNextRule]{" + digester.match +
                        "} Call [NULL PARENT]." +
                        methodName + "(" + child + ")");
            } else {
                digester.log.debug("[SetNextRule]{" + digester.match +
                        "} Call " + parent.getClass().getName() + "." +
                        methodName + "(" + child + ")");
            }
        }

        // Call the specified method
        //digester.addSetNext("Server", "setServer", "org.apache.catalina.Server");
        //调用指定方法，假设tomcat指定的方法为setServer，对应的类型是org.apache.catalina.Server
        //那么这里调用的就是Catalina的setService方法，将创建的StandardServer对象设置给了Catalina对象
        IntrospectionUtils.callMethod1(parent, methodName,
                child, paramType, digester.getClassLoader());

    }
```

> CallMethodRule

begin方法

```
public void begin(String namespace, String name, Attributes attributes)
            throws Exception {

        // Push an array to capture the parameter values if necessary
        //参数个数
        if (paramCount > 0) {
            //构建参数数组
            Object parameters[] = new Object[paramCount];
            for (int i = 0; i < parameters.length; i++) {
                parameters[i] = null;
            }
            //将参数数组入栈
            digester.pushParams(parameters);
        }

    }
```

body方法

```
 public void body(String namespace, String name, String bodyText)
            throws Exception {
        //如果没有参数的话，将标签体的文本保存下来
        if (paramCount == 0) {
            this.bodyText = bodyText.trim();
        }

    }
```

end方法

```
public void end(String namespace, String name) throws Exception {

        // Retrieve or construct the parameter values array
        Object parameters[] = null;
        if (paramCount > 0) {
            //弹出参数
            parameters = (Object[]) digester.popParams();

        
            //如果对应的方法有一个参数，但是xml中没有指定参数，那么之后返回
            if (paramCount == 1 && parameters[0] == null) {
                return;
            }
        //如果定义参数类型
        } else if (paramTypes != null && paramTypes.length != 0) {

            // In the case where the parameter for the method
            // is taken from the body text, but there is no
            // body text included in the source XML file,
            // skip the method call
            if (bodyText == null) {
                return;
            }
            //那么将标签体的内容作为参数
            parameters = new Object[1];
            parameters[0] = bodyText;
        }

        // Construct the parameter values array we will need
        // We only do the conversion if the param value is a String and
        // the specified paramType is not String.
        Object paramValues[] = new Object[paramTypes.length];
        for (int i = 0; i < paramTypes.length; i++) {
            // convert nulls and convert stringy parameters
            // for non-stringy param types
            if(
                parameters[i] == null ||
                 (parameters[i] instanceof String &&
                   !String.class.isAssignableFrom(paramTypes[i]))) {
                //进行参数类型转换
                paramValues[i] =
                        IntrospectionUtils.convert((String) parameters[i], paramTypes[i]);
            } else {
                paramValues[i] = parameters[i];
            }
        }

        // Determine the target object for the method call
        Object target;
        //通过偏移量从堆栈中获取对象
        if (targetOffset >= 0) {
            target = digester.peek(targetOffset);
        } else {
            target = digester.peek( digester.getCount() + targetOffset );
        }

        //。。。。。。省略判断和打印日志代码
        //调用方法
        Object result = IntrospectionUtils.callMethodN(target, methodName,
                paramValues, paramTypes);
        //处理方法的返回值
        processMethodCallResult(result);
    }
```
> CallParamRule

begin

```
public void begin(String namespace, String name, Attributes attributes)
            throws Exception {

        Object param = null;

        if (attributeName != null) {
            //获取指定属性值
            param = attributes.getValue(attributeName);

        } else if(fromStack) {
            //从指定的栈下标中获取参数
            param = digester.peek(stackIndex);
            //。。。。。。省略部分代码
        }

        if(param != null) {
            //弹出参数
            //设置参数，这里没有进行null判断，并且参数直接强转为Object的数组，这个规则应该是联合其他的规则使用的
            Object parameters[] = (Object[]) digester.peekParams();
            parameters[paramIndex] = param;
        }
    }
```
body

```
public void body(String namespace, String name, String bodyText)
            throws Exception {
        //如果没有指定属性，并且没有指定从栈中获取参数，那么将标签体的值作为参数
        if (attributeName == null && !fromStack) {
            // We must wait to set the parameter until end
            // so that we can make sure that the right set of parameters
            // is at the top of the stack
            if (bodyTextStack == null) {
                bodyTextStack = new ArrayStack<>();
            }
            bodyTextStack.push(bodyText.trim());
        }

    }
```
end

```
public void end(String namespace, String name) {
        //如果确实使用了标签体的内容作为参数，那么需要将值塞进去，以便其他的规则使用参数
        if (bodyTextStack != null && !bodyTextStack.empty()) {
            // what we do now is push one parameter onto the top set of parameters
            Object parameters[] = (Object[]) digester.peekParams();
            parameters[paramIndex] = bodyTextStack.pop();
        }
    }
```

> ConnectorCreateRule


```
/**
 * @param namespace xml标签的命名空间
 * @param name 标签名
 * @param attributes 当前标签的属性列表
 */
public void begin(String namespace, String name, Attributes attributes)
            throws Exception {
        //上一个创建的对象是StandardService，所以tomcat的规则对顺序的依赖还是比较强的
        Service svc = (Service)digester.peek();
        Executor ex = null;
        //获取Connector标签的executor属性，如果有值的话就用这个值去Service实例中循环查找
        //名字相同的线程池
        if ( attributes.getValue("executor")!=null ) {
            ex = svc.getExecutor(attributes.getValue("executor"));
        }
        //构建连接器
        //(*1*)
        Connector con = new Connector(attributes.getValue("protocol"));
        if (ex != null) {
            //设置Service给的线程池到协议处理中
            setExecutor(con, ex);
        }
        //获取ssl 实现类名
        String sslImplementationName = attributes.getValue("sslImplementationName");
        if (sslImplementationName != null) {
            setSSLImplementationName(con, sslImplementationName);
        }
        //入栈
        digester.push(con);
    }
    
     //(*1*)
    public Connector(String protocol) {
        //根据指定的协议，设置对应实现的协议类名
        //(*2*)
        setProtocol(protocol);
        // Instantiate protocol handler
        ProtocolHandler p = null;
        try {
            //创建协议处理器
            Class<?> clazz = Class.forName(protocolHandlerClassName);
            p = (ProtocolHandler) clazz.getConstructor().newInstance();
        } catch (Exception e) {
            log.error(sm.getString(
                    "coyoteConnector.protocolHandlerInstantiationFailed"), e);
        } finally {
            this.protocolHandler = p;
        }
        //设置uri编码
        if (Globals.STRICT_SERVLET_COMPLIANCE) {
            uriCharset = StandardCharsets.ISO_8859_1;
        } else {
            uriCharset = StandardCharsets.UTF_8;
        }
    }
    
    //(*2*)
    public void setProtocol(String protocol) {
        //判断是否支持apr连接器
        boolean aprConnector = AprLifecycleListener.isAprAvailable() &&
                AprLifecycleListener.getUseAprConnector();
        //一般我们单独的tomcat使用的就是Http11NioProtocol
        if ("HTTP/1.1".equals(protocol) || protocol == null) {
            if (aprConnector) {
                setProtocolHandlerClassName("org.apache.coyote.http11.Http11AprProtocol");
            } else {
                setProtocolHandlerClassName("org.apache.coyote.http11.Http11NioProtocol");
            }
            //用于与Apache服务器通信协议
        } else if ("AJP/1.3".equals(protocol)) {
            if (aprConnector) {
                setProtocolHandlerClassName("org.apache.coyote.ajp.AjpAprProtocol");
            } else {
                setProtocolHandlerClassName("org.apache.coyote.ajp.AjpNioProtocol");
            }
        } else {
            setProtocolHandlerClassName(protocol);
        }
    }

```
未实现body方法

```
public void end(String namespace, String name) throws Exception {
        //弹出Connector对象
        digester.pop();
    }
```


tomcat在Catalina中创建Disgester对象时，加入了大量的规则，我们现在来分析下tomcat在规则都准备什么内容


```
//解析Server标签时创建StandardServer对象并压入栈
digester.addObjectCreate("Server",
                                 "org.apache.catalina.core.StandardServer",
                                 "className");
        //设置Server标签上的所有属性到上面创建的server对象中
        digester.addSetProperties("Server");
        //解析Server标签时，将创建的Server对象设置到上一个栈对象Catalina中
        //调用其setServer方法
        digester.addSetNext("Server",
                            "setServer",
                            "org.apache.catalina.Server");
        //解析到Server下的GlobalNamingResources标签时，创建GlobalNamingResources对象
        digester.addObjectCreate("Server/GlobalNamingResources",
                                 "org.apache.catalina.deploy.NamingResourcesImpl");
        //填充属性
        digester.addSetProperties("Server/GlobalNamingResources");
        //将NamingResourcesImpl对象设置到StandardServer对象中
        digester.addSetNext("Server/GlobalNamingResources",
                            "setGlobalNamingResources",
                            "org.apache.catalina.deploy.NamingResourcesImpl");
        //解析到Server下的Listener时创建Listener对象，由标签中的className属性指定
        digester.addObjectCreate("Server/Listener",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties("Server/Listener");
        //给Server对象添加生命周期监听器
        digester.addSetNext("Server/Listener",
                            "addLifecycleListener",
                            "org.apache.catalina.LifecycleListener");
        //解析Server标签下的Service标签时，创建StandardService对象
        digester.addObjectCreate("Server/Service",
                                 "org.apache.catalina.core.StandardService",
                                 "className");
        //填充属性
        digester.addSetProperties("Server/Service");
        //将StandardService对象添加到Server对象中
        digester.addSetNext("Server/Service",
                            "addService",
                            "org.apache.catalina.Service");
        //解析Server/Service/Listener标签时创建Listener对象，具体的实现类由className属性决定
        digester.addObjectCreate("Server/Service/Listener",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties("Server/Service/Listener");
        //将监听器添加到Service对象中
        digester.addSetNext("Server/Service/Listener",
                            "addLifecycleListener",
                            "org.apache.catalina.LifecycleListener");

        //Executor，解析线程池
        digester.addObjectCreate("Server/Service/Executor",
                         "org.apache.catalina.core.StandardThreadExecutor",
                         "className");
        digester.addSetProperties("Server/Service/Executor");
        //将线程池添加到Service对象中
        digester.addSetNext("Server/Service/Executor",
                            "addExecutor",
                            "org.apache.catalina.Executor");

        //创建Connector，设置线程池，设置协议
        digester.addRule("Server/Service/Connector",
                         new ConnectorCreateRule());
        //将Connector标签的所有属性设置到Connector对象中，排除executor和sslImplementationName属性
        digester.addRule("Server/Service/Connector",
                         new SetAllPropertiesRule(new String[]{"executor", "sslImplementationName"}));
        //将Connector对象设置到Service中
        digester.addSetNext("Server/Service/Connector",
                            "addConnector",
                            "org.apache.catalina.connector.Connector");
        //创建SSLHostConfig对象
        digester.addObjectCreate("Server/Service/Connector/SSLHostConfig",
                                 "org.apache.tomcat.util.net.SSLHostConfig");
        digester.addSetProperties("Server/Service/Connector/SSLHostConfig");
        digester.addSetNext("Server/Service/Connector/SSLHostConfig",
                "addSslHostConfig",
                "org.apache.tomcat.util.net.SSLHostConfig");
        //SSL证书
        digester.addRule("Server/Service/Connector/SSLHostConfig/Certificate",
                         new CertificateCreateRule());
        digester.addRule("Server/Service/Connector/SSLHostConfig/Certificate",
                         new SetAllPropertiesRule(new String[]{"type"}));
        digester.addSetNext("Server/Service/Connector/SSLHostConfig/Certificate",
                            "addCertificate",
                            "org.apache.tomcat.util.net.SSLHostConfigCertificate");
        //create OpenSSLConf
        digester.addObjectCreate("Server/Service/Connector/SSLHostConfig/OpenSSLConf",
                                 "org.apache.tomcat.util.net.openssl.OpenSSLConf");
        digester.addSetProperties("Server/Service/Connector/SSLHostConfig/OpenSSLConf");
        digester.addSetNext("Server/Service/Connector/SSLHostConfig/OpenSSLConf",
                            "setOpenSslConf",
                            "org.apache.tomcat.util.net.openssl.OpenSSLConf");
        //创建OpenSSLConfCmd
        digester.addObjectCreate("Server/Service/Connector/SSLHostConfig/OpenSSLConf/OpenSSLConfCmd",
                                 "org.apache.tomcat.util.net.openssl.OpenSSLConfCmd");
        digester.addSetProperties("Server/Service/Connector/SSLHostConfig/OpenSSLConf/OpenSSLConfCmd");
        digester.addSetNext("Server/Service/Connector/SSLHostConfig/OpenSSLConf/OpenSSLConfCmd",
                            "addCmd",
                            "org.apache.tomcat.util.net.openssl.OpenSSLConfCmd");
        //创建Connector的监听器
        digester.addObjectCreate("Server/Service/Connector/Listener",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties("Server/Service/Connector/Listener");
        digester.addSetNext("Server/Service/Connector/Listener",
                            "addLifecycleListener",
                            "org.apache.catalina.LifecycleListener");
        //升级协议，比如websocket
        digester.addObjectCreate("Server/Service/Connector/UpgradeProtocol",
                                  null, // MUST be specified in the element
                                  "className");
        digester.addSetProperties("Server/Service/Connector/UpgradeProtocol");
        digester.addSetNext("Server/Service/Connector/UpgradeProtocol",
                            "addUpgradeProtocol",
                            "org.apache.coyote.UpgradeProtocol");

        // Add RuleSets for nested elements
        //添加规则集，对于全局命名资源，主要是添加了上下文环境
        digester.addRuleSet(new NamingRuleSet("Server/GlobalNamingResources/"));
        digester.addRuleSet(new EngineRuleSet("Server/Service/"));
        digester.addRuleSet(new HostRuleSet("Server/Service/Engine/"));
        digester.addRuleSet(new ContextRuleSet("Server/Service/Engine/Host/"));
        //准备集群结果集，添加一些sessionid生成器等等
        addClusterRuleSet(digester, "Server/Service/Engine/Host/Cluster/");
        digester.addRuleSet(new NamingRuleSet("Server/Service/Engine/Host/Context/"));

        // When the 'engine' is found, set the parentClassLoader.
        digester.addRule("Server/Service/Engine",
                         new SetParentClassLoaderRule(parentClassLoader));
        addClusterRuleSet(digester, "Server/Service/Engine/Cluster/");
```

上面代码中有出现添加监听器的，默认添加的监听器有这些


```
  <Listener className="org.apache.catalina.startup.VersionLoggerListener"/>
 
  <!-- Security listener. Documentation at /docs/config/listeners.html
  <Listener className="org.apache.catalina.security.SecurityListener" />
  -->
  
  <!--APR library loader. Documentation at /docs/apr.html -->
  <Listener SSLEngine="on" className="org.apache.catalina.core.AprLifecycleListener"/>
  
  <!-- Prevent memory leaks due to use of particular java/javax APIs-->
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener"/>
  
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener"/>
  
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener"/>
```
我们再着重研究一下以下几个规则集EngineRuleSet，HostRuleSet，ContextRuleSet

> EngineRuleSet


```
//prefix = "Server/Service/"
 public void addRuleInstances(Digester digester) {
        //创建StandardEngine，在创建Engine的时候会创建一个基础的管道阀StandardEngineValve
        //后面添加的管道阀都在基础管道阀的前面
        digester.addObjectCreate(prefix + "Engine",
                                 "org.apache.catalina.core.StandardEngine",
                                 "className");
        //设置属性
        digester.addSetProperties(prefix + "Engine");
        //创建EngineConfig生命周期监听器并add到Engine对象中
        digester.addRule(prefix + "Engine",
                         new LifecycleListenerRule
                         ("org.apache.catalina.startup.EngineConfig",
                          "engineConfigClass"));
        //将Engine设置到Service对象中
        digester.addSetNext(prefix + "Engine",
                            "setContainer",
                            "org.apache.catalina.Engine");

        //Cluster configuration start
        digester.addObjectCreate(prefix + "Engine/Cluster",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Engine/Cluster");
        //添加cluster
        digester.addSetNext(prefix + "Engine/Cluster",
                            "setCluster",
                            "org.apache.catalina.Cluster");
        //Cluster configuration end
        //构建监听器
        digester.addObjectCreate(prefix + "Engine/Listener",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Engine/Listener");
        //将监听器add到Engine中
        digester.addSetNext(prefix + "Engine/Listener",
                            "addLifecycleListener",
                            "org.apache.catalina.LifecycleListener");

        //添加Realm的规则集，用于创建Realm，Realm一般用于用户认证之类的
        digester.addRuleSet(new RealmRuleSet(prefix + "Engine/"));
        //解析并创建管道阀
        digester.addObjectCreate(prefix + "Engine/Valve",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Engine/Valve");
        //添加管道阀
        digester.addSetNext(prefix + "Engine/Valve",
                            "addValve",
                            "org.apache.catalina.Valve");

    }
```

> HostRuleSet


```
//prefix = "Server/Service/Engine/"
public void addRuleInstances(Digester digester) {
        //创建StandardHost对象，构建对象时添加基础管道阀StandardHostValve
        digester.addObjectCreate(prefix + "Host",
                                 "org.apache.catalina.core.StandardHost",
                                 "className");
        //设置属性
        digester.addSetProperties(prefix + "Host");
        //将Engine的类加载器设置到Host中
        digester.addRule(prefix + "Host",
                         new CopyParentClassLoaderRule());
        //给host对象添加HostConfig生命周期监听器
        digester.addRule(prefix + "Host",
                         new LifecycleListenerRule
                         ("org.apache.catalina.startup.HostConfig",
                          "hostConfigClass"));
        //将创建的host对象添加到Engine对象中
        digester.addSetNext(prefix + "Host",
                            "addChild",
                            "org.apache.catalina.Container");
        //调用Host的addAlias方法，以标签体的文本作为参数传递，设置别名
        digester.addCallMethod(prefix + "Host/Alias",
                               "addAlias", 0);

        //Cluster configuration start，集群配置开始
        digester.addObjectCreate(prefix + "Host/Cluster",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Host/Cluster");
        digester.addSetNext(prefix + "Host/Cluster",
                            "setCluster",
                            "org.apache.catalina.Cluster");
        //Cluster configuration end
        //host监听器开始
        digester.addObjectCreate(prefix + "Host/Listener",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Host/Listener");
        digester.addSetNext(prefix + "Host/Listener",
                            "addLifecycleListener",
                            "org.apache.catalina.LifecycleListener");
        //Realm认证
        digester.addRuleSet(new RealmRuleSet(prefix + "Host/"));
        //添加管道阀
        digester.addObjectCreate(prefix + "Host/Valve",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Host/Valve");
        digester.addSetNext(prefix + "Host/Valve",
                            "addValve",
                            "org.apache.catalina.Valve");

    }
```

> ContextRuleSet


```
//prefix = "Server/Service/Engine/Host/"
public void addRuleInstances(Digester digester) {

        if (create) {
            //创建StandardContext对象，并在创建时设置基础管道阀StandardContextValve
            digester.addObjectCreate(prefix + "Context",
                    "org.apache.catalina.core.StandardContext", "className");
            //给StandardContext对象设置属性
            digester.addSetProperties(prefix + "Context");
        } else {
            //给StandardContext对象设置属性
            digester.addRule(prefix + "Context", new SetContextPropertiesRule());
        }

        if (create) {
            //给context添加ContextConfig声明周期
            digester.addRule(prefix + "Context",
                             new LifecycleListenerRule
                                 ("org.apache.catalina.startup.ContextConfig",
                                  "configClass"));
            //作为子容器添加到Host容器中
            digester.addSetNext(prefix + "Context",
                                "addChild",
                                "org.apache.catalina.Container");
        }
        //创建监听器
        digester.addObjectCreate(prefix + "Context/Listener",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Context/Listener");
        //添加生命周期监听器
        digester.addSetNext(prefix + "Context/Listener",
                            "addLifecycleListener",
                            "org.apache.catalina.LifecycleListener");
        //创建WebappLoader，用于web项目的加载器
        digester.addObjectCreate(prefix + "Context/Loader",
                            "org.apache.catalina.loader.WebappLoader",
                            "className");
        digester.addSetProperties(prefix + "Context/Loader");
        //将类加载器设置到Context对象中
        digester.addSetNext(prefix + "Context/Loader",
                            "setLoader",
                            "org.apache.catalina.Loader");
        //创建session管理器StandardManager
        digester.addObjectCreate(prefix + "Context/Manager",
                                 "org.apache.catalina.session.StandardManager",
                                 "className");
        digester.addSetProperties(prefix + "Context/Manager");
        //将session管理器设置到Context对象
        digester.addSetNext(prefix + "Context/Manager",
                            "setManager",
                            "org.apache.catalina.Manager");
        //设置存储session逻辑的Store对象
        digester.addObjectCreate(prefix + "Context/Manager/Store",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Context/Manager/Store");
        //将Store对象设置到StandardManager中
        digester.addSetNext(prefix + "Context/Manager/Store",
                            "setStore",
                            "org.apache.catalina.Store");
        //创建sessionId的生成器
        digester.addObjectCreate(prefix + "Context/Manager/SessionIdGenerator",
                                 "org.apache.catalina.util.StandardSessionIdGenerator",
                                 "className");
        digester.addSetProperties(prefix + "Context/Manager/SessionIdGenerator");
        
         //设置sessionId的生成器到StandardManager中去
        digester.addSetNext(prefix + "Context/Manager/SessionIdGenerator",
                            "setSessionIdGenerator",
                            "org.apache.catalina.SessionIdGenerator");
       //构建应用参数对象，内部维护key-》value
        digester.addObjectCreate(prefix + "Context/Parameter",
                                 "org.apache.tomcat.util.descriptor.web.ApplicationParameter");
        digester.addSetProperties(prefix + "Context/Parameter");
        //将参数设置到Context中
        digester.addSetNext(prefix + "Context/Parameter",
                            "addApplicationParameter",
                            "org.apache.tomcat.util.descriptor.web.ApplicationParameter");

        digester.addRuleSet(new RealmRuleSet(prefix + "Context/"));
        //创建StandardRoot对象，它实现了WebResourceRoot接口，用获取web资源
        digester.addObjectCreate(prefix + "Context/Resources",
                                 "org.apache.catalina.webresources.StandardRoot",
                                 "className");
        digester.addSetProperties(prefix + "Context/Resources");
        digester.addSetNext(prefix + "Context/Resources",
                            "setResources",
                            "org.apache.catalina.WebResourceRoot");

        //创建实现了WebResourceSet接口的对象，这个PreResources标签的资源会被设置到StandardRoot的preResources字段上
        digester.addObjectCreate(prefix + "Context/Resources/PreResources",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Context/Resources/PreResources");
        //设置PreResources资源集合
        digester.addSetNext(prefix + "Context/Resources/PreResources",
                            "addPreResources",
                            "org.apache.catalina.WebResourceSet");
        //创建JarResources资源
        digester.addObjectCreate(prefix + "Context/Resources/JarResources",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Context/Resources/JarResources");
        digester.addSetNext(prefix + "Context/Resources/JarResources",
                            "addJarResources",
                            "org.apache.catalina.WebResourceSet");
        //创建PostResources资源
        digester.addObjectCreate(prefix + "Context/Resources/PostResources",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Context/Resources/PostResources");
        digester.addSetNext(prefix + "Context/Resources/PostResources",
                            "addPostResources",
                            "org.apache.catalina.WebResourceSet");

        //创建资源链接
        digester.addObjectCreate(prefix + "Context/ResourceLink",
                "org.apache.tomcat.util.descriptor.web.ContextResourceLink");
        digester.addSetProperties(prefix + "Context/ResourceLink");
        digester.addRule(prefix + "Context/ResourceLink",
                new SetNextNamingRule("addResourceLink",
                        "org.apache.tomcat.util.descriptor.web.ContextResourceLink"));
        //创建管道阀
        digester.addObjectCreate(prefix + "Context/Valve",
                                 null, // MUST be specified in the element
                                 "className");
        digester.addSetProperties(prefix + "Context/Valve");
        digester.addSetNext(prefix + "Context/Valve",
                            "addValve",
                            "org.apache.catalina.Valve");
        //调用context对象的addWatchedResource方法，以WatchedResource标签内的内容作为参数
        //监控资源
        digester.addCallMethod(prefix + "Context/WatchedResource",
                               "addWatchedResource", 0);
        //
        digester.addCallMethod(prefix + "Context/WrapperLifecycle",
                               "addWrapperLifecycle", 0);
        //添加被包装的生命周期监听器
        digester.addCallMethod(prefix + "Context/WrapperListener",
                               "addWrapperListener", 0);
        //创建jar扫描器
        digester.addObjectCreate(prefix + "Context/JarScanner",
                                 "org.apache.tomcat.util.scan.StandardJarScanner",
                                 "className");
        digester.addSetProperties(prefix + "Context/JarScanner");
        digester.addSetNext(prefix + "Context/JarScanner",
                            "setJarScanner",
                            "org.apache.tomcat.JarScanner");
        //创建jar扫描器过滤器
        digester.addObjectCreate(prefix + "Context/JarScanner/JarScanFilter",
                                 "org.apache.tomcat.util.scan.StandardJarScanFilter",
                                 "className");
        digester.addSetProperties(prefix + "Context/JarScanner/JarScanFilter");
        digester.addSetNext(prefix + "Context/JarScanner/JarScanFilter",
                            "setJarScanFilter",
                            "org.apache.tomcat.JarScanFilter");
        //cookie处理器
        digester.addObjectCreate(prefix + "Context/CookieProcessor",
                                 "org.apache.tomcat.util.http.Rfc6265CookieProcessor",
                                 "className");
        digester.addSetProperties(prefix + "Context/CookieProcessor");
        digester.addSetNext(prefix + "Context/CookieProcessor",
                            "setCookieProcessor",
                            "org.apache.tomcat.util.http.CookieProcessor");
    }
```
> NamingRuleSet


```
//pref = "Server/Service/Engine/Host/Context/"
 public void addRuleInstances(Digester digester) {
        //创建ContextEjb，这个应该是用于Ejb的资源，Ejb以前听说过，但是从未使用过
        digester.addObjectCreate(prefix + "Ejb",
                                 "org.apache.tomcat.util.descriptor.web.ContextEjb");
        digester.addRule(prefix + "Ejb", new SetAllPropertiesRule());
        digester.addRule(prefix + "Ejb",
                new SetNextNamingRule("addEjb",
                            "org.apache.tomcat.util.descriptor.web.ContextEjb"));
        //创建上下文环境
        digester.addObjectCreate(prefix + "Environment",
                                 "org.apache.tomcat.util.descriptor.web.ContextEnvironment");
        digester.addSetProperties(prefix + "Environment");
        digester.addRule(prefix + "Environment",
                            new SetNextNamingRule("addEnvironment",
                            "org.apache.tomcat.util.descriptor.web.ContextEnvironment"));

        digester.addObjectCreate(prefix + "LocalEjb",
                                 "org.apache.tomcat.util.descriptor.web.ContextLocalEjb");
        digester.addRule(prefix + "LocalEjb", new SetAllPropertiesRule());
        digester.addRule(prefix + "LocalEjb",
                new SetNextNamingRule("addLocalEjb",
                            "org.apache.tomcat.util.descriptor.web.ContextLocalEjb"));
        //创建上下文资源对象
        digester.addObjectCreate(prefix + "Resource",
                                 "org.apache.tomcat.util.descriptor.web.ContextResource");
        digester.addRule(prefix + "Resource", new SetAllPropertiesRule());
        digester.addRule(prefix + "Resource",
                new SetNextNamingRule("addResource",
                            "org.apache.tomcat.util.descriptor.web.ContextResource"));
        //上下资源环境引用
        digester.addObjectCreate(prefix + "ResourceEnvRef",
            "org.apache.tomcat.util.descriptor.web.ContextResourceEnvRef");
        digester.addRule(prefix + "ResourceEnvRef", new SetAllPropertiesRule());
        digester.addRule(prefix + "ResourceEnvRef",
                new SetNextNamingRule("addResourceEnvRef",
                            "org.apache.tomcat.util.descriptor.web.ContextResourceEnvRef"));
        //上下文服务
        digester.addObjectCreate(prefix + "ServiceRef",
            "org.apache.tomcat.util.descriptor.web.ContextService");
        digester.addRule(prefix + "ServiceRef", new SetAllPropertiesRule());
        digester.addRule(prefix + "ServiceRef",
                new SetNextNamingRule("addService",
                            "org.apache.tomcat.util.descriptor.web.ContextService"));
        //事务
        digester.addObjectCreate(prefix + "Transaction",
            "org.apache.tomcat.util.descriptor.web.ContextTransaction");
        digester.addRule(prefix + "Transaction", new SetAllPropertiesRule());
        digester.addRule(prefix + "Transaction",
                new SetNextNamingRule("setTransaction",
                            "org.apache.tomcat.util.descriptor.web.ContextTransaction"));

    }

```
上面的NamingRuleSet设置的类都继承自ResourceBase，这些类的作用是做什么的，我们现在不用急于知道，看到后面我们自然就能够明了

以下为Catalina的类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/8ee582557e5de61a4c22c163ad9088c7.png)
以下是tomcat的架构图（来自汪建的《tomcat内核设计剖析》），其实在这一步tomcat就已经将大部分的内容已经准备完毕了，剩下的就是启动他们，就像一个机器人一样，一切需要准备零件都组将完毕后，剩下的就是按下启动按钮，进行初始化，加载各种程序，开始启动它。

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/5c22a786e221c7a6c81279972e82d690.png)
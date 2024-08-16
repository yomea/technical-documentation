## 1、初始化

context的初始化没啥可分析的逻辑但是它所触发的初始化事件让ContextConfig做了一些事情


```
protected void org.apache.catalina.startup.ContextConfig.init() {
        // Called from StandardContext.init()
        //创建Digester
        //(*1*)
        Digester contextDigester = createContextDigester();
        contextDigester.getParser();

        if (log.isDebugEnabled()) {
            log.debug(sm.getString("contextConfig.init"));
        }
        context.setConfigured(false);//设置配置状态，默认设置为失败，以免被误任务成功
        ok = true;
        //(*2*)
        contextConfig(contextDigester);
    }
    
    //(*1*)
    protected Digester createContextDigester() {
        Digester digester = new Digester();
        digester.setValidating(false);
        digester.setRulesValidation(true);
        HashMap<Class<?>, List<String>> fakeAttributes = new HashMap<>();
        ArrayList<String> attrs = new ArrayList<>();
        attrs.add("className");
        fakeAttributes.put(Object.class, attrs);
        digester.setFakeAttributes(fakeAttributes);
        RuleSet contextRuleSet = new ContextRuleSet("", false);
        //添加context规则组，这些规则组我们在分析catalina启动的时候已经分析过了，此处不再赘述
        digester.addRuleSet(contextRuleSet);
        RuleSet namingRuleSet = new NamingRuleSet("Context/");
        //添加naming规则组，这些规则组我们在分析catalina启动的时候已经分析过了，此处不再赘述
        digester.addRuleSet(namingRuleSet);
        return digester;
    }
    
    //(*2*)
     protected void contextConfig(Digester digester) {

        String defaultContextXml = null;

        // Open the default context.xml file, if it exists 如果存在默认的配置，使用它
        if (context instanceof StandardContext) {
            defaultContextXml = ((StandardContext)context).getDefaultContextXml();
        }
        // set the default if we don't have any overrides 
        //如果没有配置默认的，那么是用全局默认配置conf/context.xml
        if (defaultContextXml == null) {
            defaultContextXml = Constants.DefaultContextXml;
        }
        //如果还没有进行解析，那么就会重新解析，默认的全局配置-》configBase下的context.xml.default-》configBase下的配置
        if (!context.getOverride()) {
            File defaultContextFile = new File(defaultContextXml);
            if (!defaultContextFile.isAbsolute()) {
                defaultContextFile =
                        new File(context.getCatalinaBase(), defaultContextXml);
            }
            if (defaultContextFile.exists()) {
                try {
                    URL defaultContextUrl = defaultContextFile.toURI().toURL();
                    //解析方式就是以digester注册的规则进行sax解析
                    processContextConfig(digester, defaultContextUrl);
                } catch (MalformedURLException e) {
                    log.error(sm.getString(
                            "contextConfig.badUrl", defaultContextFile), e);
                }
            }
            //读取host级别的context描述文件context.xml.default，host级别的配置会覆盖
            //全局的配置
            File hostContextFile = new File(getHostConfigBase(), Constants.HostContextXml);
            if (hostContextFile.exists()) {
                try {
                    URL hostContextUrl = hostContextFile.toURI().toURL();
                    processContextConfig(digester, hostContextUrl);
                } catch (MalformedURLException e) {
                    log.error(sm.getString(
                            "contextConfig.badUrl", hostContextFile), e);
                }
            }
        }
        //读取自身配置的context文件，自身的配置文件优先级最高，会覆盖前面两个的设置
        if (context.getConfigFile() != null) {
            processContextConfig(digester, context.getConfigFile());
        }

    }

```

从上面的代码可以看到，ContextConfig对context的初始化事件主要是进行context描述文件的进一步处理，可能有人就会疑惑，那我们前面进行context发布的时候不是已经进行了context描述文件的读取了吗？是这样的，前面读取的配置描述文件只给Disgester添加了StandardContext的创建规则，所以不会完全进行读取。而且优先级为自身context描述文件优先级最高，再者就是host级别的，最后才是全局的context.xml

## 2、ContextConfig的before_start事件

```
protected synchronized void beforeStart() {

        try {
            fixDocBase();
        } catch (IOException e) {
            log.error(sm.getString(
                    "contextConfig.fixDocBase", context.getName()), e);
        }

        antiLocking();
}
```
修复docBase，这个用来做什么的呢？就是变成指定的解压后的路径
```
 protected void fixDocBase() throws IOException {
        //host对象
        Host host = (Host) context.getParent();
        //获取appBase路径
        File appBase = host.getAppBaseFile();
        //获取docBase路径
        String docBase = context.getDocBase();
        if (docBase == null) {
            // Trying to guess the docBase according to the path
            //如果docBase为空，那么使用path进行代替
            String path = context.getPath();
            if (path == null) {
                return;
            }
            //然后又来一次ContextName设置baseName，这部分逻辑前面分析过，这里就不再赘述了
            ContextName cn = new ContextName(path, context.getWebappVersion());
            //对于appBase目录下的，一般这个baseName就是目录名
            docBase = cn.getBaseName();
        }
        
        File file = new File(docBase);
        //设置成绝对路径
        if (!file.isAbsolute()) {
            docBase = (new File(appBase, docBase)).getPath();
        } else {
            docBase = file.getCanonicalPath();
        }
        file = new File(docBase);
        
        String origDocBase = docBase;
        
        ContextName cn = new ContextName(context.getPath(), context.getWebappVersion());
        //对于appBase目录下的，一般这个baseName就是目录名，没有版本号
        String pathName = cn.getBaseName();

        boolean unpackWARs = true;
        if (host instanceof StandardHost) {
            unpackWARs = ((StandardHost) host).isUnpackWARs();
            //使用子容器覆盖父容器的属性
            if (unpackWARs && context instanceof StandardContext) {
                unpackWARs =  ((StandardContext) context).getUnpackWAR();
            }
        }
        //判断这个context是否是在appBase路径内的
        boolean docBaseInAppBase = docBase.startsWith(appBase.getPath() + File.separatorChar);
        
        if (docBase.toLowerCase(Locale.ENGLISH).endsWith(".war") && !file.isDirectory()) {
            //通过File构建URL路径
            URL war = UriUtil.buildJarUrl(new File(docBase));
            if (unpackWARs) {
                //解压到指定的pathName路径
                docBase = ExpandWar.expand(host, war, pathName);
                file = new File(docBase);
                docBase = file.getCanonicalPath();
                if (context instanceof StandardContext) {
                    //保存原始的路径，对于未解压的war包来说，原始的路径就是war的位置
                    ((StandardContext) context).setOriginalDocBase(origDocBase);
                }
            } else {
                //这个校验我们在前面分析过，就是用于校验路径是否为正确路径，不能存在./和../这种路径
                ExpandWar.validate(host, war, pathName);
            }
        } else {
            File docDir = new File(docBase);
            File warFile = new File(docBase + ".war");
            URL war = null;
            //如果这个目录存在对应的war包，并且已经处在appBase路径，那么构建war包url，以便后面重新解压
            if (warFile.exists() && docBaseInAppBase) {
                war = UriUtil.buildJarUrl(warFile);
            }
            if (docDir.exists()) {
                if (war != null && unpackWARs) {
                    // Check if WAR needs to be re-expanded (e.g. if it has
                    // changed). Note: HostConfig.deployWar() takes care of
                    // ensuring that the correct XML file is used.
                    // This will be a NO-OP if the WAR is unchanged.
                    //重新解压
                    ExpandWar.expand(host, war, pathName);
                }
            } else {
                //如果目录不存在，存在war包，并且允许解压
                if (war != null) {
                    if (unpackWARs) {
                        docBase = ExpandWar.expand(host, war, pathName);
                        file = new File(docBase);
                        docBase = file.getCanonicalPath();
                    } else {
                        docBase = warFile.getCanonicalPath();
                        //校验
                        ExpandWar.validate(host, war, pathName);
                    }
                }
                if (context instanceof StandardContext) {
                    //保存原始路径，以便恢复现场
                    ((StandardContext) context).setOriginalDocBase(origDocBase);
                }
            }
        }

        // Re-calculate now docBase is a canonical path
        //重新判断经过处理后的docBase是否还在appBase路径中
        docBaseInAppBase = docBase.startsWith(appBase.getPath() + File.separatorChar);
        //如果在appBase中，截取掉
        if (docBaseInAppBase) {
            docBase = docBase.substring(appBase.getPath().length());
            docBase = docBase.replace(File.separatorChar, '/');
            if (docBase.startsWith("/")) {
                docBase = docBase.substring(1);
            }
        } else {
            docBase = docBase.replace(File.separatorChar, '/');
        }
        //设置解压后的docBase或者原始不用解压的目录，以便后面解析web.xml能够找到
        context.setDocBase(docBase);
    }
```
具体的解压逻辑

```
public static String expand(Host host, URL war, String pathname)
        throws IOException {

        //打开对war的链接，准备读取
        JarURLConnection juc = (JarURLConnection) war.openConnection();
        //不允许缓存，以免造成文件锁定
        juc.setUseCaches(false);
        URL jarFileUrl = juc.getJarFileURL();
        URLConnection jfuc = jarFileUrl.openConnection();

        boolean success = false;
        //将路径设置到appBase下面
        File docBase = new File(host.getAppBaseFile(), pathname);
        //构建/META-INF/war-tracker路径
        File warTracker = new File(host.getAppBaseFile(), pathname + Constants.WarTracker);
        long warLastModified = -1;

        try (InputStream is = jfuc.getInputStream()) {
            // Get the last modified time for the WAR
            //获取war包的上次修改时间
            warLastModified = jfuc.getLastModified();
        }

        // Check to see of the WAR has been expanded previously
        //如果已经存在了
        if (docBase.exists()) {
            // A WAR was expanded. Tomcat will have set the last modified
            // time of warTracker file to the last modified time of the WAR so
            // changes to the WAR while Tomcat is stopped can be detected
            //如果warTracker不存在或者它们没有发生修改，那么直接返回这个已经解压好的目录的路径即可
            if (!warTracker.exists() || warTracker.lastModified() == warLastModified) {
                // No (detectable) changes to the WAR
                success = true;
                return (docBase.getAbsolutePath());
            }
            //否则需要删除原来解压的文件，以便后面重新解压
            // WAR must have been modified. Remove expanded directory.
            log.info(sm.getString("expandWar.deleteOld", docBase));
            if (!delete(docBase)) {
                throw new IOException(sm.getString("expandWar.deleteFailed", docBase));
            }
        }

        // Create the new document base directory
        //创建解压目录
        if(!docBase.mkdir() && !docBase.isDirectory()) {
            throw new IOException(sm.getString("expandWar.createFailed", docBase));
        }

        // Expand the WAR into the new document base directory
        String canonicalDocBasePrefix = docBase.getCanonicalPath();
        if (!canonicalDocBasePrefix.endsWith(File.separator)) {
            canonicalDocBasePrefix += File.separator;
        }

        // Creating war tracker parent (normally META-INF)
        File warTrackerParent = warTracker.getParentFile();
        //创建war tracker目录
        if (!warTrackerParent.isDirectory() && !warTrackerParent.mkdirs()) {
            throw new IOException(sm.getString("expandWar.createFailed", warTrackerParent.getAbsolutePath()));
        }

        try (JarFile jarFile = juc.getJarFile()) {

            Enumeration<JarEntry> jarEntries = jarFile.entries();
            while (jarEntries.hasMoreElements()) {
                JarEntry jarEntry = jarEntries.nextElement();
                String name = jarEntry.getName();
                //解压后对应的文件名
                File expandedFile = new File(docBase, name);
                if (!expandedFile.getCanonicalPath().startsWith(
                        canonicalDocBasePrefix)) {
                    // Trying to expand outside the docBase
                    // Throw an exception to stop the deployment
                    throw new IllegalArgumentException(
                            sm.getString("expandWar.illegalPath",war, name,
                                    expandedFile.getCanonicalPath(),
                                    canonicalDocBasePrefix));
                }
                int last = name.lastIndexOf('/');
                if (last >= 0) {
                    File parent = new File(docBase,
                                           name.substring(0, last));
                    if (!parent.mkdirs() && !parent.isDirectory()) {
                        throw new IOException(
                                sm.getString("expandWar.createFailed", parent));
                    }
                }
                if (name.endsWith("/")) {
                    continue;
                }
                
                try (InputStream input = jarFile.getInputStream(jarEntry)) {
                    if (null == input) {
                        throw new ZipException(sm.getString("expandWar.missingJarEntry",
                                jarEntry.getName()));
                    }

                    // Bugzilla 33636
                    //解压
                    //(*1*)
                    expand(input, expandedFile);
                    long lastModified = jarEntry.getTime();
                    if ((lastModified != -1) && (lastModified != 0)) {
                        expandedFile.setLastModified(lastModified);
                    }
                }
            }

            // Create the warTracker file and align the last modified time
            // with the last modified time of the WAR
            //创建war tracker，并设置修改时间为war包的时间，如果war包修改，那么就会和war tracker的不一样
            warTracker.createNewFile();
            warTracker.setLastModified(warLastModified);

            success = true;
        } catch (IOException e) {
            throw e;
        } finally {
            if (!success) {
                // If something went wrong, delete expanded dir to keep things
                // clean
                deleteDir(docBase);
            }
        }

        // Return the absolute path to our new document base directory
        //返回解压后的目录
        return docBase.getAbsolutePath();
    }
    
    //(*1*)
    private static void expand(InputStream input, File file) throws IOException {
        //熟悉的流操作，不过多细说
        try (BufferedOutputStream output =
                new BufferedOutputStream(new FileOutputStream(file))) {
            byte buffer[] = new byte[2048];
            while (true) {
                int n = input.read(buffer);
                if (n <= 0)
                    break;
                output.write(buffer, 0, n);
            }
        }
    }
    
```

## 3、父类的start方法与其他容一样的逻辑，所以这里不再赘述，我们研究一下StandardContext的startInternal方法

```
protected synchronized void startInternal() throws LifecycleException {

        if(log.isDebugEnabled())
            log.debug("Starting " + getBaseName());

        // Send j2ee.state.starting notification
        if (this.getObjectName() != null) {
            Notification notification = new Notification("j2ee.state.starting",
                    this.getObjectName(), sequenceNumber.getAndIncrement());
            //触发通知
            broadcaster.sendNotification(notification);
        }
        //设置配置状态为false，并触发属性变更事件
        setConfigured(false);
        boolean ok = true;

        // Currently this is effectively a NO-OP but needs to be called to
        // ensure the NamingResources follows the correct lifecycle
        //启动命名资源，比如我们在context.xml配置的JNDI之类的资源
        if (namingResources != null) {
            namingResources.start();
        }

        // Post work directory
        // 创建工作目录，一般为$tomcat_home/work/engine名（一般为catalina）/host名（如果是本机，那么就是localhost）
        // /context的baseName（通常为path.substring(1)+"##"+版本）
        //（*1*）
        postWorkDirectory();

        // Add missing components as necessary
        //如果WebResourceRoot（一个资源的复合类，用于维护各种资源）为空，那么就new出一个来
        if (getResources() == null) {   // (1) Required by Loader
            if (log.isDebugEnabled())
                log.debug("Configuring default Resources");

            try {
                //StandardRoot是WebResourceRoot子类
                setResources(new StandardRoot(this));
            } catch (IllegalArgumentException e) {
                log.error(sm.getString("standardContext.resourcesInit"), e);
                ok = false;
            }
        }
        if (ok) {
            //启动注册资源到mbeanserver中，并判断是否需要添加
            //WEB-INF/classas/META-INF/resources下的资源，并为其挂在到根路径下
            //这样你存在WEB-INF/classas/META-INF/resources下的资源可以
            //挂载到根路径下。这个webResourceRoot用于后面类加载器设置类加载路径
            resourcesStart();
        }
        //如果没有设置类加载器，那么创建一个专属于context的类加载器
        if (getLoader() == null) {
            //父加载器是share加载器，但是一般地share加载器也会是common加载，
            //通常catalina和share加载都是为空
            WebappLoader webappLoader = new WebappLoader(getParentClassLoader());
            //设置webapp加载器的加载模式，如果为true，那么首先使用父加载器进行加载
            //如果为false，那么自己先加载
            webappLoader.setDelegate(getDelegate());
            //设置加载器，类加载器的分析往下看
            setLoader(webappLoader);
        }

       。。。。。。省略部分代码
    }
    
    //设置工作目录
    //（*1*）
    private void postWorkDirectory() {

        // Acquire (or calculate) the work directory path
        String workDir = getWorkDir();
        if (workDir == null || workDir.length() == 0) {

            // Retrieve our parent (normally a host) name
            String hostName = null;
            String engineName = null;
            String hostWorkDir = null;
            //获取父容器
            Container parentHost = getParent();
            if (parentHost != null) {
                //获取主机名
                hostName = parentHost.getName();
                if (parentHost instanceof StandardHost) {
                    //获取host工作目录，一般为 空
                    hostWorkDir = ((StandardHost)parentHost).getWorkDir();
                }
                //获取Engine容器
                Container parentEngine = parentHost.getParent();
                if (parentEngine != null) {
                    //获取Engine名
                   engineName = parentEngine.getName();
                }
            }
            if ((hostName == null) || (hostName.length() < 1))
                hostName = "_";
            if ((engineName == null) || (engineName.length() < 1))
                engineName = "_";
            //通过前面设置的path与version构建baseName，这里面使用了一个contextName，但是和前面使用不是同一个构造器
            //这个构造的路径是由path+##+version，如果path是/，ROOT，null那么这个path就被置为空字符串，拼接的路径
            //会变成ROOT##版本
            //这里假设当前context的请求路径是/tomcat/context，其版本为1.5，那么最终拼成的baseName就是tomcat#context##1.5
            String temp = getBaseName();
            if (temp.startsWith("/"))
                temp = temp.substring(1);
            temp = temp.replace('/', '_');
            temp = temp.replace('\\', '_');
            if (temp.length() < 1)
                temp = ContextName.ROOT_NAME;
            if (hostWorkDir != null ) {
                //如果存在host的工作目录，那么以hostwork为前缀
                workDir = hostWorkDir + File.separator + temp;
            } else {
                workDir = "work" + File.separator + engineName +
                    File.separator + hostName + File.separator + temp;//work/catalina/localhost/tomcat#context##1.5
            }
            //设置工作目录
            setWorkDir(workDir);
        }

        // Create this directory if necessary
        //创建目录
        File dir = new File(workDir);
        if (!dir.isAbsolute()) {
            String catalinaHomePath = null;
            try {
                catalinaHomePath = getCatalinaBase().getCanonicalPath();//D:/tomcat/work/catalina/localhost/tomcat#context##1.5
                dir = new File(catalinaHomePath, workDir);
            } catch (IOException e) {
                log.warn(sm.getString("standardContext.workCreateException",
                        workDir, catalinaHomePath, getName()), e);
            }
        }
        if (!dir.mkdirs() && !dir.isDirectory()) {
            log.warn(sm.getString("standardContext.workCreateFail", dir,
                    getName()));
        }

        // Set the appropriate servlet context attribute
        //创建ApplicationContext，实际上获取到的是一个门面
        if (context == null) {
            getServletContext();
        }
        //将当前context的工作目录以key为javax.servlet.context.tempdir设置到ApplicationContext中
        context.setAttribute(ServletContext.TEMPDIR, dir);
        //将这个context的目录设置为只读属性
        context.setAttributeReadOnly(ServletContext.TEMPDIR);
    }
    
    //（*1*）
     public ServletContext getServletContext() {

        if (context == null) {
            //创建ApplicationContext，将当前context传入
            context = new ApplicationContext(this);
            if (altDDName != null)
                context.setAttribute(Globals.ALT_DD_ATTR,altDDName);
        }
        //获取门面对象，用于屏蔽一些不提供给外部的操作
        return (context.getFacade());

    }
```

从上面的代码中，我们看到tomcat为每个context设置了单独的类加载，那么这个类加载器到底是怎么设置进去的，加载顺序是如何控制的呢？


```
//父加载器是catalina加载器，但是一般地catalina加载器也回事common加载，
            //通常catalina和share加载都是为空
            WebappLoader webappLoader = new WebappLoader(getParentClassLoader());
            //设置webapp加载器的加载模式，如果为true，那么首先使用父加载器进行加载
            //如果为false，那么自己先加载
            webappLoader.setDelegate(getDelegate());
            //设置加载器，类加载器的分析往下看
            setLoader(webappLoader);
```

我们看到context的setLoader方法

```
 public void org.apache.catalina.core.StandardContext.setLoader(Loader loader) {

        Lock writeLock = loaderLock.writeLock();
        writeLock.lock();
        Loader oldLoader = null;
        try {
            // Change components if necessary
            oldLoader = this.loader;
            //如果老加载器与新的加载器相等，那么直接返回
            if (oldLoader == loader)
                return;
            this.loader = loader;
            //停止旧的的加载器，主要是进行一些缓存，引用的线程进行释放。
            // Stop the old component if necessary
            if (getState().isAvailable() && (oldLoader != null) &&
                (oldLoader instanceof Lifecycle)) {
                try {
                    ((Lifecycle) oldLoader).stop();
                } catch (LifecycleException e) {
                    log.error("StandardContext.setLoader: stop: ", e);
                }
            }

            // Start the new component if necessary
            if (loader != null)
                //关联当前context
                loader.setContext(this);
            if (getState().isAvailable() && (loader != null) &&
                (loader instanceof Lifecycle)) {
                try {
                    //启动类加载器
                    ((Lifecycle) loader).start();
                } catch (LifecycleException e) {
                    log.error("StandardContext.setLoader: start: ", e);
                }
            }
        } finally {
            writeLock.unlock();
        }

        // Report this property change to interested listeners
        //触发属性变更事件
        support.firePropertyChange("loader", oldLoader, loader);
    }
```

我们可以看到，这个WebappLoader类没有继承JDK的Classloader类，它所拥有的方法中没有看到用于加载类的方法，带着疑问，我们继续查看WebappClassloader的start方法，无独有偶，WebappLoader继承了LifecycleMBeanBase类，这个类不再说了，就是注册MBean然后调用子类实现的startInternal方法


```
 protected void org.apache.catalina.loader.WebappLoader.startInternal() throws LifecycleException {

        if (log.isDebugEnabled())
            log.debug(sm.getString("webappLoader.starting"));
        //如果context没有配置WebResourceRoot，那么设置状态正在启动
        if (context.getResources() == null) {
            log.info("No resources for " + context);
            //ContextCofig对starting不感兴趣
            setState(LifecycleState.STARTING);
            return;
        }

        // Construct a class loader based on our current repositories list
        try {
            //创建真正的类加载器
            //(*1*)
            classLoader = createClassLoader();
            //设置资源，用于加载设置的类资源
            classLoader.setResources(context.getResources());
            //设置类加载模式
            classLoader.setDelegate(this.delegate);

            // Configure our repositories
            //拼接context类加载器及其父加载器的类加载路径设置到ApplicationContext中，
            //顺便说一声这个类加载器继承自URLClassLoader。
            //(*2*)
            setClassPath();
            //当设置了JDK的安全管理器时，给类加载器设置资源的操作权限
            //设置context的工作空间为可读可写，resouces设置的资源为只读等等。
            setPermissions();
            //(*3*)
            //主要设置web应用的类加载器地址
            ((Lifecycle) classLoader).start();
            //获取context的名字
            String contextName = context.getName();
            if (!contextName.startsWith("/")) {
                contextName = "/" + contextName;
            }
            //注册mbean
            ObjectName cloname = new ObjectName(context.getDomain() + ":type=" +
                    classLoader.getClass().getSimpleName() + ",host=" +
                    context.getParent().getName() + ",context=" + contextName);
            Registry.getRegistry(null, null)
                .registerComponent(classLoader, cloname, null);

        } catch (Throwable t) {
            t = ExceptionUtils.unwrapInvocationTargetException(t);
            ExceptionUtils.handleThrowable(t);
            log.error( "LifecycleException ", t );
            throw new LifecycleException("start: ", t);
        }
        //设置STARTING状态
        setState(LifecycleState.STARTING);
    }
    
    
    //(*1*)
    private WebappClassLoaderBase createClassLoader()
        throws Exception {
        //loaderClass = ParallelWebappClassLoader，继承自WebappClassLoaderBase
        Class<?> clazz = Class.forName(loaderClass);
        WebappClassLoaderBase classLoader = null;
        //如果父加载器为空，重新获取父加载器，防御式编程
        if (parentClassLoader == null) {
            parentClassLoader = context.getParentClassLoader();
        }
        Class<?>[] argTypes = { ClassLoader.class };
        Object[] args = { parentClassLoader };
        //反射ParallelWebappClassLoader(ClassLoader parentClassLoader)构造器创建ParallelWebappClassLoader
        Constructor<?> constr = clazz.getConstructor(argTypes);
        classLoader = (WebappClassLoaderBase) constr.newInstance(args);
        //返回
        return classLoader;
    }
    
    //(*2*)
    private void setClassPath() {

        // Validate our current state information
        //如果没有设置context，那么直接返回，没有context也就没有lib，classes
        if (context == null)
            return;
        //获取ApplicationContext，如果ApplicationContext为空，返回
        ServletContext servletContext = context.getServletContext();
        if (servletContext == null)
            return;

        StringBuilder classpath = new StringBuilder();

        // Assemble the class path information from our class loader chain
        //获取上面创建的类加载器
        ClassLoader loader = getClassLoader();
        //如果加载模式为父加载器加载
        if (delegate && loader != null) {
            // Skip the webapp loader for now as delegation is enabled
            loader = loader.getParent();
        }
        //那么从父加载器往上append各个类加载器的classpath，在Unix系统中使用：分割，在windows系统中用；分割
        while (loader != null) {
            if (!buildClassPath(classpath, loader)) {
                break;
            }
            loader = loader.getParent();
        }
        //添加当前类加载器的加载路径
        if (delegate) {
            // Delegation was enabled, go back and add the webapp paths
            loader = getClassLoader();
            if (loader != null) {
                buildClassPath(classpath, loader);
            }
        }

        this.classpath = classpath.toString();

        // Store the assembled class path as a servlet context attribute
        //将当前context的类加载器可以加载到类的路径保存到ApplicationContext中，key为org.apache.catalina.jsp_classpath
        servletContext.setAttribute(Globals.CLASS_PATH_ATTR, this.classpath);
    }
    
    //(*3*)
    public void start() throws LifecycleException {
        //将加载器的状态设置为STARTING_PREP
        state = LifecycleState.STARTING_PREP;
        //获取/WEB-INF/classes路径web资源对象
        WebResource classes = resources.getResource("/WEB-INF/classes");
        //将/WEB-INF/classes路径设置为类加载器路径
        if (classes.isDirectory() && classes.canRead()) {
            localRepositories.add(classes.getURL());
        }
        //获取/WEB-INF/lib下的所有jar包资源
        WebResource[] jars = resources.listResources("/WEB-INF/lib");
        for (WebResource jar : jars) {
            if (jar.getName().endsWith(".jar") && jar.isFile() && jar.canRead()) {
                localRepositories.add(jar.getURL());
                jarModificationTimes.put(
                        jar.getName(), Long.valueOf(jar.getLastModified()));
            }
        }
        //将加载器的状态设置为STARTED
        state = LifecycleState.STARTED;
    }
```

接下来我们继续分析context的类加载器是如何加载class的


```
public Class<?> org.apache.catalina.loader.WebappClassLoaderBase.loadClass(String name, boolean resolve) throws ClassNotFoundException {

        synchronized (getClassLoadingLock(name)) {
            if (log.isDebugEnabled())
                log.debug("loadClass(" + name + ", " + resolve + ")");
            Class<?> clazz = null;

            // Log access to stopped class loader
            //如果当前类加载的状态已经是停止状态，那么抛出错误，不允许再使用这个类加载加载类
            checkStateForClassLoading(name);

            // (0) Check our previously loaded local class cache
            //从当前类加载的缓存中寻找已经加载过的类对象
            clazz = findLoadedClass0(name);
            if (clazz != null) {
                if (log.isDebugEnabled())
                    log.debug("  Returning class from cache");
                //是否解析这个类，底层调用本地方法，用于验证，准备，解析一个class类
                if (resolve)
                    resolveClass(clazz);
                return (clazz);
            }

            // (0.1) Check our previously loaded class cache
            //通过JDK自带的本地方法查找jvm是否已经加载过这个类
            clazz = findLoadedClass(name);
            if (clazz != null) {
                if (log.isDebugEnabled())
                    log.debug("  Returning class from cache");
                if (resolve)
                    resolveClass(clazz);
                return (clazz);
            }
            
            // (0.2) Try loading the class with the system class loader, to prevent
            //       the webapp from overriding Java SE classes. This implements
            //       SRV.10.7.2
            //处理类名，比如com.java.Test -》 com/java/Test.class
            String resourceName = binaryNameToPath(name, false);
            //获取JDK系统加载器，这个加载器是在new当前这个类的对象时设置的
            ClassLoader javaseLoader = getJavaseClassLoader();
            boolean tryLoadingFromJavaseLoader;
            try {
                //通过类加载器的getResource查看是否存在这个资源，而不是使用loadClass，因为loadClass的方法
                //相对于getResource来说，它消耗的性能是昂贵（想象一下，系统类加载器是双亲委派机制）。
                tryLoadingFromJavaseLoader = (javaseLoader.getResource(resourceName) != null);
            } catch (Throwable t) {
                // Swallow all exceptions apart from those that must be re-thrown
                ExceptionUtils.handleThrowable(t);
                //虽然抛出错误，依然要设置为true是因为不排除其父类可以加载它。
                tryLoadingFromJavaseLoader = true;
            }

            if (tryLoadingFromJavaseLoader) {
                try {
                    //通过java类加载器加载类
                    clazz = javaseLoader.loadClass(name);
                    if (clazz != null) {
                        if (resolve)
                            resolveClass(clazz);
                        return (clazz);
                    }
                } catch (ClassNotFoundException e) {
                    // Ignore
                }
            }

            // (0.5) Permission to access this class when using a SecurityManager
            //如果缓存中没有加载过并且java的类加载器都加载不到，那么将使用自己
            //的类加载器，但是如果设置了java的安全管理器，那么检查是否有此资源的访问权限
            if (securityManager != null) {
                int i = name.lastIndexOf('.');
                if (i >= 0) {
                    try {
                        securityManager.checkPackageAccess(name.substring(0,i));
                    } catch (SecurityException se) {
                        String error = "Security Violation, attempt to use " +
                            "Restricted Class: " + name;
                        log.info(error, se);
                        throw new ClassNotFoundException(error, se);
                    }
                }
            }
            //判断是否使用代理加载模式，也就是首先使用父加载器加载
            //filter方法是对类名进行判断是否使用父加载器，比如javax开头的用父类加载
            //用el等等，那么自己加载。
            boolean delegateLoad = delegate || filter(name, true);

            // (1) Delegate to our parent if requested
            //通过父加载器加载
            if (delegateLoad) {
                if (log.isDebugEnabled())
                    log.debug("  Delegating to parent classloader1 " + parent);
                try {
                    //指定类加载器为父加载器，第二参数表示是否需要提前进行初始化（调用类加载器）
                    clazz = Class.forName(name, false, parent);
                    if (clazz != null) {
                        if (log.isDebugEnabled())
                            log.debug("  Loading class from parent");
                        if (resolve)
                            resolveClass(clazz);
                        return (clazz);
                    }
                } catch (ClassNotFoundException e) {
                    // Ignore
                }
            }

            // (2) Search local repositories
            if (log.isDebugEnabled())
                log.debug("  Searching local repositories");
            try {
                //从WebResourceRoot中寻找资源，首先从WEB-INF/classes下找，然后从此类设置的URL仓库中
                //从前面的分析（类加载器的start）中可以看到，它会从WEB-INF/lib下。
                clazz = findClass(name);
                if (clazz != null) {
                    if (log.isDebugEnabled())
                        log.debug("  Loading class from local repository");
                    if (resolve)
                        resolveClass(clazz);
                    return (clazz);
                }
            } catch (ClassNotFoundException e) {
                // Ignore
            }

            // (3) Delegate to parent unconditionally
            //如果还是没有找到，那么从父加载器中寻找
            if (!delegateLoad) {
                if (log.isDebugEnabled())
                    log.debug("  Delegating to parent classloader at end: " + parent);
                try {
                    clazz = Class.forName(name, false, parent);
                    if (clazz != null) {
                        if (log.isDebugEnabled())
                            log.debug("  Loading class from parent");
                        if (resolve)
                            resolveClass(clazz);
                        return (clazz);
                    }
                } catch (ClassNotFoundException e) {
                    // Ignore
                }
            }
        }

        throw new ClassNotFoundException(name);
    }
```

从上面的分析，我们可以看到tomcat的应用加载器有两种模式，一种是非代理模式，另一种是代理模式，我们来看下它们之间的区别

- 代理模式 webapp类加器器缓存-》JVM中是否已经加载-》java类加载器（双亲委派）-》父加载器（双亲委派）-》自己加载
- 非代理模式 webapp类加器器缓存-》JVM中是否已经加载-》java类加载器（双亲委派）-》自己加载-》父加载器（双亲委派）

区别就是在后面两个阶段，前面依然遵循了java的双亲委派，主要是为了安全。

到这里web类加载器的分析就结束了，我们继续StandardContext的start阶段的分析。
类加载器设置完毕后，接下来就是设置cookie处理器，初始化字符集映射（地区-》字符集），从MANIFEST.MF文件中检查依赖，设置JNDI相关的命名上下文，将刚才创建的类加载器绑定到当前线程中，启动类加载器（就是上面将的类加载的启动），启动realm的start方法

接下来又是重点，tomcat启动realm之后就触发了configure_start事件


```
protected synchronized void org.apache.catalina.startup.ContextConfig.configureStart() {
        // Called from StandardContext.start()
        
        webConfig();
        
        //加载被@WebFilter，@WebServlet,，@WebListener注解的类
        if (!context.getIgnoreAnnotations()) {
            applicationAnnotationsConfig();
        }
        //添加security role，通常使用security-constraint标签配置
        /**
         *   <security-constraint>
         *      <web-resource-collection>
         *          <url-pattern>/manager/*</url-pattern>
         *       </web-resource-collection>
         *       <auth-constraint>
         *           <role-name>admin</role-name>    
         *       </auth-constraint>  
         *   </security-constraint>
         */
        if (ok) {
            validateSecurityRoles();
        }

        // Configure an authenticator if we need one
        //设置各类验证器，比如登录验证方式，管道阀验证等等
        //    <login-config>
    	//	    <auth-method>BASIC</auth-method>
    	//    </login-config>
        if (ok) {
            authenticatorConfig();
        }
        
        // Make our application available if no problems were encountered
        //如果没有出现配置问题，那么设置为配置成功
        if (ok) {
            context.setConfigured(true);
        } else {
            log.error(sm.getString("contextConfig.unavailable"));
            //否则设置为配置失败
            context.setConfigured(false);
        }

    }
```
上面的代码调用了webConfig()方法，这个方法就是解析我们web项目中的web.xml用的

```
protected void org.apache.catalina.startup.ContextConfig.webConfig() {
        //构建WebXmlParser，在这个解析器的构造器中，构造了我们熟悉的Digester，这个Digester可想而知就
        //设置了很多跟web.xml中标签相关的规则
        WebXmlParser webXmlParser = new WebXmlParser(context.getXmlNamespaceAware(),
                context.getXmlValidation(), context.getXmlBlockExternal());

        Set<WebXml> defaults = new HashSet<>();
        //获取默认web.xml-》全局的conf/web.xml，host级别的web.xml.default，并使用webXmlParser进行解析成WebXml对象
        defaults.add(getDefaultWebXmlFragment(webXmlParser));
        //创建一个空的WebXml对象，作为解析context自身web.xml的空篮子
        WebXml webXml = createWebXml();

        // Parse context level web.xml
        //加载web.xml资源
        InputSource contextWebXml = getContextWebXmlSource();
        //解析内容到webXml对象中，如果失败，那么ok标记为false
        if (!webXmlParser.parseWebXml(contextWebXml, webXml, false)) {
            ok = false;
        }

        。。。。。。省略
```
从上面可以看到，tomcat首先解析了全局的web.xml配置，然后解析host级别的web.xml配置，最后解析自身的web.xml配置，下面我们来看下web.xml是如何被解析的


```
public boolean org.apache.tomcat.util.descriptor.web.WebXmlParser.parseWebXml(InputSource source, WebXml dest,
            boolean fragment) {

        boolean ok = true;

        if (source == null) {
            return ok;
        }

        XmlErrorHandler handler = new XmlErrorHandler();

        Digester digester;
        WebRuleSet ruleSet;
        //如果是web.xml片段，那么使用专用片段的Digester和规则
        //否则使用正常的Digester和规则，片段的根标签是web-fragment
        //正常的web.xml根标签是web-app
        if (fragment) {
            digester = webFragmentDigester;
            ruleSet = webFragmentRuleSet;
        } else {
            digester = webDigester;
            ruleSet = webRuleSet;
        }
        //将webXml对象设置为根对象
        digester.push(dest);
        //设置xml解析时的错误处理器
        digester.setErrorHandler(handler);

        try {
            //解析内容
            digester.parse(source);

            if (handler.getWarnings().size() > 0 ||
                    handler.getErrors().size() > 0) {
                ok = false;
                handler.logFindings(log, source.getSystemId());
            }
        } 
        
        。。。。。。异常打印与设置ok标记为false

        return ok;
    }
```
从上面的代码中我们可以看到，tomcat进行web.xml的解析也是通过Digester和设置其中的规则进行解析的，所以具体的内容还是要看规则的实现，这里不会去研究所有标签的规则解析。

> context-param

web上下文参数

```
//
digester.addCallMethod(fullPrefix + "/context-param",
                               "addContextParam", 2);
digester.addCallParam(fullPrefix + "/context-param/param-name", 0);
digester.addCallParam(fullPrefix + "/context-param/param-value", 1);
```
调用webXml对象的addContextParam方法添加参数。CallMethodRule与CallParamRule规则，我们在前面已经分析过，这里不再赘述，在begin中CallMethodRule是准备好参数数组
CallParamRule往参数数组中添加值，CallMethodRule的end将会使用这些参数调用对应的方法。

方法如下：

```
private final Map<String,String> contextParams = new HashMap<>();
public void org.apache.tomcat.util.descriptor.web.WebXml.addContextParam(String param, String value) {
    contextParams.put(param, value);
}

```

> Filter


```
        //创建FilterDef对象
        digester.addObjectCreate(fullPrefix + "/filter",
                                 "org.apache.tomcat.util.descriptor.web.FilterDef");
        //add到webXml对象中
        digester.addSetNext(fullPrefix + "/filter",
                            "addFilter",
                            "org.apache.tomcat.util.descriptor.web.FilterDef");
        //给FilterDef对象设置描述信息
        digester.addCallMethod(fullPrefix + "/filter/description",
                               "setDescription", 0);
        //给FilterDef对象设置display-name
        digester.addCallMethod(fullPrefix + "/filter/display-name",
                               "setDisplayName", 0);
        //调用FilterDef对象的setFilterClass方法上设置filter的class类
        digester.addCallMethod(fullPrefix + "/filter/filter-class",
                               "setFilterClass", 0);
        //设置过滤器名
        digester.addCallMethod(fullPrefix + "/filter/filter-name",
                               "setFilterName", 0);
        //设置icon地址
        digester.addCallMethod(fullPrefix + "/filter/icon/large-icon",
                               "setLargeIcon", 0);
        //设置icon地址
        digester.addCallMethod(fullPrefix + "/filter/icon/small-icon",
                               "setSmallIcon", 0);
        //是否支持异步
        digester.addCallMethod(fullPrefix + "/filter/async-supported",
                "setAsyncSupported", 0);
        //设置初始化参数
        digester.addCallMethod(fullPrefix + "/filter/init-param",
                               "addInitParameter", 2);
        digester.addCallParam(fullPrefix + "/filter/init-param/param-name",
                              0);
        digester.addCallParam(fullPrefix + "/filter/init-param/param-value",
                              1);
        //创建FilterMap对象
        digester.addObjectCreate(fullPrefix + "/filter-mapping",
                                 "org.apache.tomcat.util.descriptor.web.FilterMap");
        //给webXml对象设置FilterMap（过滤器映射）
        digester.addSetNext(fullPrefix + "/filter-mapping",
                                 "addFilterMapping",
                                 "org.apache.tomcat.util.descriptor.web.FilterMap");
        //FilterMap对象设置过滤器名
        digester.addCallMethod(fullPrefix + "/filter-mapping/filter-name",
                               "setFilterName", 0);
        //设置servlet名
        digester.addCallMethod(fullPrefix + "/filter-mapping/servlet-name",
                               "addServletName", 0);
        //设置url通配符
        digester.addCallMethod(fullPrefix + "/filter-mapping/url-pattern",
                               "addURLPattern", 0);
```
从上面的代码中，我们可以看到tomcat解析Filter标签，是创建一个FilterDef然后注册到webXml对象中，我们看下addFilter这个方法

```
private final Map<String,FilterDef> filters = new LinkedHashMap<>();

public void addFilter(FilterDef filter) {
    if (filters.containsKey(filter.getFilterName())) {
        // Filter names must be unique within a web(-fragment).xml
        throw new IllegalArgumentException(
                sm.getString("webXml.duplicateFilter",
                        filter.getFilterName()));
    }
    //过滤器名-》过滤器定义对象
    filters.put(filter.getFilterName(), filter);
}
```
过滤器的mapping配置
```
private final Set<FilterMap> filterMaps = new LinkedHashSet<>();
private final Set<String> filterMappingNames = new HashSet<>();

public void addFilterMapping(FilterMap filterMap) {
    filterMaps.add(filterMap);
    filterMappingNames.add(filterMap.getFilterName());
}
```

> Listener


```
digester.addCallMethod(fullPrefix + "/listener/listener-class",
                                "addListener", 0);
```
对应的webXml对象的方法

```
private final Set<String> listeners = new LinkedHashSet<>();
    public void addListener(String className) {
        listeners.add(className);
    }
```
从上面的方法来看，tomcat对监听器的处理只是通过listeners这个泛型为String的集合进行储存，没有进行对象的包装，之所以这么做，主要是监听器就一个简单标签，没有其他的信息，所有不需要包装成对象。

> Servlet

```
//创建ServletDefCreateRule规则
digester.addRule(fullPrefix + "/servlet",
                         new ServletDefCreateRule());
//给webXml对象添加Servlet
digester.addSetNext(fullPrefix + "/servlet",
                    "addServlet",
                    "org.apache.tomcat.util.descriptor.web.ServletDef");
//给Servlet对象添加初始化参数
digester.addCallMethod(fullPrefix + "/servlet/init-param",
                       "addInitParameter", 2);
digester.addCallParam(fullPrefix + "/servlet/init-param/param-name",
                      0);
digester.addCallParam(fullPrefix + "/servlet/init-param/param-value",
                      1);
//设置jsp文件路径
digester.addCallMethod(fullPrefix + "/servlet/jsp-file",
                       "setJspFile", 0);
//设置启动顺序
digester.addCallMethod(fullPrefix + "/servlet/load-on-startup",
                       "setLoadOnStartup", 0);
//设置servlet的类对象
digester.addCallMethod(fullPrefix + "/servlet/servlet-class",
                      "setServletClass", 0);
//设置servlet类的名字
digester.addCallMethod(fullPrefix + "/servlet/servlet-name",
                      "setServletName", 0);
//设置MultipartDef对象，用于配置诸如文件上传的最大size
digester.addObjectCreate(fullPrefix + "/servlet/multipart-config",
                         "org.apache.tomcat.util.descriptor.web.MultipartDef");
//设置到WebXml对象中
digester.addSetNext(fullPrefix + "/servlet/multipart-config",
                    "setMultipartDef",
                    "org.apache.tomcat.util.descriptor.web.MultipartDef");
//设置上传配置
digester.addCallMethod(fullPrefix + "/servlet/multipart-config/location",
                       "setLocation", 0);
digester.addCallMethod(fullPrefix + "/servlet/multipart-config/max-file-size",
                       "setMaxFileSize", 0);
digester.addCallMethod(fullPrefix + "/servlet/multipart-config/max-request-size",
                       "setMaxRequestSize", 0);
digester.addCallMethod(fullPrefix + "/servlet/multipart-config/file-size-threshold",
                       "setFileSizeThreshold", 0);
//是否支持异步
digester.addCallMethod(fullPrefix + "/servlet/async-supported",
                       "setAsyncSupported", 0);
//设置当前servlet是否可用
digester.addCallMethod(fullPrefix + "/servlet/enabled",
                       "setEnabled", 0);

//准备addServletMapping方法调用
digester.addRule(fullPrefix + "/servlet-mapping",
                       new CallMethodMultiRule("addServletMapping", 2, 0));
//准备servletName参数
digester.addCallParam(fullPrefix + "/servlet-mapping/servlet-name", 1);
//准备url参数，后面会调用CallMethodMultiRule的end方法
digester.addRule(fullPrefix + "/servlet-mapping/url-pattern", new CallParamMultiRule(0));
```

上面在添加servlet规则的时候，添加了一个ServletDefCreateRule规则，我们来分析一下

> begin


```
public void org.apache.tomcat.util.descriptor.web.ServletDefCreateRule.begin(String namespace, String name, Attributes attributes)
        throws Exception {
        //创建一个类似FilteDef的对象，可想而知，这就是给包装Servlet信息的定义类
        ServletDef servletDef = new ServletDef();
        digester.push(servletDef);
        if (digester.getLogger().isDebugEnabled())
            digester.getLogger().debug("new " + servletDef.getClass().getName());
    }
```
end方法没有做什么，只是弹出对象

下面对上面调用的WebXml对象的方法进行一些简单的罗列

注册servlet

```
//servletName->ServletDef对象
private final Map<String,ServletDef> servlets = new HashMap<>();
    public void addServlet(ServletDef servletDef) {
        servlets.put(servletDef.getServletName(), servletDef);
        if (overridable) {
            servletDef.setOverridable(overridable);
        }
    }
```

添加Servlet映射

```
//urlPattern -> urlPattern
private final Map<String,String> servletMappings = new HashMap<>();
private final Set<String> servletMappingNames = new HashSet<>();

public void addServletMapping(String urlPattern, String servletName) {
    addServletMappingDecoded(UDecoder.URLDecode(urlPattern, getCharset()), servletName);
}
public void addServletMappingDecoded(String urlPattern, String servletName) {
    //urlPattern -> urlPattern
    String oldServletName = servletMappings.put(urlPattern, servletName);
    if (oldServletName != null) {
        // Duplicate mapping. As per clarification from the Servlet EG,
        // deployment should fail.
        throw new IllegalArgumentException(sm.getString(
                "webXml.duplicateServletMapping", oldServletName,
                servletName, urlPattern));
    }
    servletMappingNames.add(servletName);
}
```
好了，web.xml的解析先到这，其他的规则自己去研究，这里不过多的赘述

接下来继续ContextConfig的config_start事件


```
 protected void org.apache.catalina.startup.ContextConfig.webConfig() {
    
        。。。。。省略在上面已经分析过的代码

        ServletContext sContext = context.getServletContext();

        //解析web-fragment（servlet3.0新增的模块化，用于提供在jar包中已经设置好的监听器啊，过滤器啊，servlet啊等等）
        //得扫描/WEB-INF/lib/，/WEB-INF/classes下的META-INF/web-fragment.xml
        //然后创建WebXml对象，创建的方式和之前分析的一样，这里就不过多的分析
        Map<String,WebXml> fragments = processJarsForWebFragments(webXml, webXmlParser);

        // Step 2. Order the fragments.
        //给片段进行排序，片段中的filter，listener，servlet等类的创建是具有顺序关系的，所以需要进行排序
        //比如在某某之后创建，这就是一种顺序。
        Set<WebXml> orderedFragments = null;
        orderedFragments =
                WebXml.orderWebFragments(webXml, fragments, sContext);
                
         // Step 3. Look for ServletContainerInitializer implementations
         //加载jar包，类路径下的META-INF/services/javax.servlet.ServletContainerInitializer配置文件，多个ServletContainerInitializer用换行符分割
         //通过反射进行加载
        if (ok) {
            //(*1*)
            processServletContainerInitializers();
        }
        //通常我们不会指定ServletContainerInitializer，所以typeInitializerMap一般为空
        //如果webXml.isMetadataComplete()我们在context.xml配置文件中指定为true，那么将不会进行注解的扫描
        if  (!webXml.isMetadataComplete() || typeInitializerMap.size() > 0) {
            // Step 4. Process /WEB-INF/classes for annotations and
            // @HandlesTypes matches
            Map<String,JavaClassCacheEntry> javaClassCache = new HashMap<>();

            if (ok) {
                //列出/WEB-INF/classes下的所有资源
                WebResource[] webResources =
                        context.getResources().listResources("/WEB-INF/classes");

                for (WebResource webResource : webResources) {
                    // Skip the META-INF directory from any JARs that have been
                    // expanded in to WEB-INF/classes (sometimes IDEs do this).
                    //跳过META-INF
                    if ("META-INF".equals(webResource.getName())) {
                        continue;
                    }
                    //处理注解@WebServlet,@WebFilter,@WebListener,并根据HandleType是注解还是类型对资源进行筛选
                    //设置到initializerClassMap中 ServletContainerInitializer -》 Set<Class<?>>
                    processAnnotationsWebResource(webResource, webXml,
                            webXml.isMetadataComplete(), javaClassCache);
                }
            }

            // Step 5. Process JARs for annotations and
            // @HandlesTypes matches - only need to process those fragments we
            // are going to use (remember orderedFragments includes any
            // container fragments)
            if (ok) {
                //处理web-fragment对应jar包的@WebServlet,@WebFilter,@WebListener注解,并根据HandleType是注解还是类型对资源进行筛选
                //这里就一个问题了，如果我META-INF下没有web-fragment就不会进行注解的扫描了，想想这也是合理的，要不然所有引用的jar都
                //得扫一遍，这个启动就太慢了
                processAnnotations(
                        orderedFragments, webXml.isMetadataComplete(), javaClassCache);
            }

            // Cache, if used, is no longer required so clear it
            javaClassCache.clear();
        }
        //如果webXml.isMetadataComplete()为true，就不进行操作
        if (!webXml.isMetadataComplete()) {
            // Step 6. Merge web-fragment.xml files into the main web.xml
            // file.
            if (ok) {
                //合并片段中的数据到主webXml中,比如context-params的值进行合并，属性进行合并
                //模块片段相同的属性不会覆盖主配置的值
                ok = webXml.merge(orderedFragments);
            }

            // Step 7. Apply global defaults
            // Have to merge defaults before JSP conversion since defaults
            // provide JSP servlet definition.
            //合并全局和host级别的web.xml，全局的属性不会覆盖主配置的
            webXml.merge(defaults);

            // Step 8. Convert explicitly mentioned jsps to servlets
            //从tomcat的注解直接翻译过来的意思就是：转换显示提到的jsp为servlet
            //用我们的话将其实它就是把我们配置的jsp类型的ServletDef的class设置成org.apache.jasper.servlet.JspServlet
            //然后添加一个初始化参数jspFile -》 jspFile的路径
            if (ok) {
                convertJsps(webXml);
            }

            // Step 9. Apply merged web.xml to Context
            if (ok) {
                //将webXml对象中的内容设置到context中，其中有一个比较重要的操作也在里面
                //那就是将Servlet包装成了context的子容器StandardWrapper
                configureContext(webXml);
            }
        } else {
            //和上面是一样的逻辑
            webXml.merge(defaults);
            convertJsps(webXml);
            configureContext(webXml);
        }
    
        // Always need to look for static resources
        // Step 10. Look for static resources packaged in JARs
        if (ok) {
            // Spec does not define an order.
            // Use ordered JARs followed by remaining JARs
            Set<WebXml> resourceJars = new LinkedHashSet<>();
            //获取所有的模块片段
            for (WebXml fragment : orderedFragments) {
                resourceJars.add(fragment);
            }
            for (WebXml fragment : fragments.values()) {
                if (!resourceJars.contains(fragment)) {
                    resourceJars.add(fragment);
                }
            }
            //处理jar资源，设置META-INF/resources下的静态资源到context的WebResourceRoot中，以便web
            //直接访问
            processResourceJARs(resourceJars);
            // See also StandardContext.resourcesStart() for
            // WEB-INF/classes/META-INF/resources configuration
        }

        // Step 11. Apply the ServletContainerInitializer config to the
        // context
        if (ok) {
            //将ServletContainerInitializer设置到context中
            for (Map.Entry<ServletContainerInitializer,
                    Set<Class<?>>> entry :
                        initializerClassMap.entrySet()) {
                if (entry.getValue().isEmpty()) {
                    context.addServletContainerInitializer(
                            entry.getKey(), null);
                } else {
                    context.addServletContainerInitializer(
                            entry.getKey(), entry.getValue());
                }
            }
}

//(*1*)
protected void processServletContainerInitializers() {

        List<ServletContainerInitializer> detectedScis;
        try {
            //从ApplicationContext指定的jar包路径和context的类路径加载ServletContainerInitializer
            WebappServiceLoader<ServletContainerInitializer> loader = new WebappServiceLoader<>(context);
            detectedScis = loader.load(ServletContainerInitializer.class);
        } catch (IOException e) {
            log.error(sm.getString(
                    "contextConfig.servletContainerInitializerFail",
                    context.getName()),
                e);
            ok = false;
            return;
        }

        for (ServletContainerInitializer sci : detectedScis) {
            //ServletContainerInitializer.class -> new HashSet<Class<?>>()
            initializerClassMap.put(sci, new HashSet<Class<?>>());

            HandlesTypes ht;
            try {
                //获取HandlesTypes注解
                ht = sci.getClass().getAnnotation(HandlesTypes.class);
            } catch (Exception e) {
                continue;
            }
            if (ht == null) {
                continue;
            }
            //获取这个ServletContainerInitializer感兴趣的类型
            Class<?>[] types = ht.value();
            if (types == null) {
                continue;
            }

            for (Class<?> type : types) {
                if (type.isAnnotation()) {
                    handlesTypesAnnotations = true;
                } else {
                    handlesTypesNonAnnotations = true;
                }
                //typeInitializerMap 类型-》对这个类型感兴趣的ServletContainerInitializer集合
                Set<ServletContainerInitializer> scis =
                        typeInitializerMap.get(type);
                if (scis == null) {
                    scis = new HashSet<>();
                    typeInitializerMap.put(type, scis);
                }
                scis.add(sci);
            }
        }
    }

```

到这里，web.xml的配置就已经结束了，所有配置的信息都已经设置到context中，那么接下来将要做什么呢？我们跳出config_start，回到StandardContext的startInternal方法


```

    protected synchronized void startInternal() throws LifecycleException {
    
            。。。。。。省略已分析过的
                //触发config_start事件，这个事件在前面我们已经分析过了
                fireLifecycleEvent(Lifecycle.CONFIGURE_START_EVENT, null);

                // Start our child containers, if not already started
                //启动子容器，也就是StandardWrapper，它的startInternal方法就进行了一下通知，没有
                //做其他的
                for (Container child : findChildren()) {
                    if (!child.getState().isAvailable()) {
                        child.start();
                    }
                }
                
                // Start the Valves in our pipeline (including the basic),
                // if any
                //调用排管的start，循环调用管道阀的start
                if (pipeline instanceof Lifecycle) {
                    ((Lifecycle) pipeline).start();
                }

                // Acquire clustered manager
                Manager contextManager = null;
                //获取管理器，用于管理session，如果没有创建，那么现在创建
                Manager manager = getManager();
                if (manager == null) {
                    if ( (getCluster() != null) && distributable) {
                        try {
                            contextManager = getCluster().createManager(getName());
                        } catch (Exception ex) {
                            log.error("standardContext.clusterFail", ex);
                            ok = false;
                        }
                    } else {
                        contextManager = new StandardManager();
                    }
                }

                // Configure default manager if none was specified
                //设置session管理器
                if (contextManager != null) {
                    setManager(contextManager);
                }

                if (manager!=null && (getCluster() != null) && distributable) {
                    //let the cluster know that there is a context that is distributable
                    //and that it has its own manager
                    getCluster().registerManager(manager);
                }
            }
            //如果配置失败，打印失败日志
            if (!getConfigured()) {
                log.error(sm.getString("standardContext.configurationFail"));
                ok = false;
            }

            // We put the resources into the servlet context
            //将context的资源集设置到ApplicationContext中去
            //key 为 org.apache.catalina.resources
            if (ok)
                getServletContext().setAttribute
                    (Globals.RESOURCES_ATTR, getResources());
            
            if (ok ) {
                //设置实例管理器，用于创建实例，销毁实例
                if (getInstanceManager() == null) {
                    javax.naming.Context context = null;
                    if (isUseNaming() && getNamingContextListener() != null) {
                        context = getNamingContextListener().getEnvContext();
                    }
                    Map<String, Map<String, String>> injectionMap = buildInjectionMap(
                            getIgnoreAnnotations() ? new NamingResourcesImpl(): getNamingResources());
                    setInstanceManager(new DefaultInstanceManager(context,
                            injectionMap, this, this.getClass().getClassLoader()));
                }
                getServletContext().setAttribute(
                        InstanceManager.class.getName(), getInstanceManager());
                InstanceManagerBindings.bind(getLoader().getClassLoader(), getInstanceManager());
            }

            // Create context attributes that will be required
            if (ok) {
                //往ApplicationContext中设置jar扫描器
                getServletContext().setAttribute(
                        JarScanner.class.getName(), getJarScanner());
            }

            // Set up the context init params
            //合并参数，将context中的初始化参数合并到ApplicationContext中
            //如果发生冲突，那么通过ApplicationContext中设置的参数来决定是否可以被覆盖
            mergeParameters();

            // Call ServletContainerInitializers
            for (Map.Entry<ServletContainerInitializer, Set<Class<?>>> entry :
                initializers.entrySet()) {
                try {
                    //调用ServletContainerInitializers，我们可以自定义加载Filter，Servlet，Listener
                    entry.getKey().onStartup(entry.getValue(),
                            getServletContext());
                } catch (ServletException e) {
                    log.error(sm.getString("standardContext.sciFail"), e);
                    ok = false;
                    break;
                }
            }

            // Configure and call application event listeners
            if (ok) {
                //使用上面设置的InstanceManager构建监听器对象，比如ServletContextAttributeListener,HttpSessionIdListener等事件监听器
                //ServletContextListener,HttpSessionListener为生命周期监听器
                //并触发容器启动事件，然后调用ServletContextListener的contextInitialized方法。
                if (!listenerStart()) {
                    log.error(sm.getString("standardContext.listenerFail"));
                    ok = false;
                }
            }

            // Check constraints for uncovered HTTP methods
            // Needs to be after SCIs and listeners as they may programmatically
            // change constraints
            //检查约束
            if (ok) {
                checkConstraintsForUncoveredMethods(findConstraints());
            }

            try {
                // Start manager
                Manager manager = getManager();
                //调用管理器的start方法，这里是StandManager的start方法，它的方法主要是加载
                //javax.servlet.context.tempdir指定的临时目录下的SESSION.ser文件，进行反序列化
                //如果上次没有序列化session到这个文件，就不会进行反序列化，直接返回
                if (manager instanceof Lifecycle) {
                    ((Lifecycle) manager).start();
                }
            } catch(Exception e) {
                log.error(sm.getString("standardContext.managerFail"), e);
                ok = false;
            }

            // Configure and call application filters
            if (ok) {
                //创建filter对象，并调用其init方法，然后注册到context的
                //HashMap<String, ApplicationFilterConfig> filterConfigs = new HashMap<>();
                //中，ApplicationFilterConfig继承FilterConfig，持有Filter对象
                if (!filterStart()) {
                    log.error(sm.getString("standardContext.filterFail"));
                    ok = false;
                }
            }

            // Load and initialize all "load on startup" servlets
            if (ok) {
                //启动loadOnStartup大于0的，内部使用TreeMap进行排序，key值为loadOnStartup，value为Wrapper集合
                //通过InstanceManager反射创建Servlet，然后调用其初始化方法
                if (!loadOnStartup(findChildren())){
                    log.error(sm.getString("standardContext.servletFail"));
                    ok = false;
                }
            }

            // Start ContainerBackgroundProcessor thread
            super.threadStart();
        } finally {
            // Unbinding thread
            //取消绑定，并把老的加载器恢复（恢复现场）
            unbindThread(oldCCL);
        }

        // Set available status depending upon startup success
        if (ok) {
            if (log.isDebugEnabled())
                log.debug("Starting completed");
        } else {
            log.error(sm.getString("standardContext.startFailed", getName()));
        }

        startTime=System.currentTimeMillis();

        // Send j2ee.state.running notification
        if (ok && (this.getObjectName() != null)) {
            Notification notification =
                new Notification("j2ee.state.running", this.getObjectName(),
                                 sequenceNumber.getAndIncrement());
            broadcaster.sendNotification(notification);
        }

        // The WebResources implementation caches references to JAR files. On
        // some platforms these references may lock the JAR files. Since web
        // application start is likely to have read from lots of JARs, trigger
        // a clean-up now.
        //清理一下缓存
        getResources().gc();

        // Reinitializing if something went wrong
        if (!ok) {
            setState(LifecycleState.FAILED);
        } else {
            //设置启动状态starting
            setState(LifecycleState.STARTING);
        }
    }
```

StandardContext的启动非常的复杂，看这部分源码需要很大的耐心，现在来总结一下，首先tomcat对context进行了fixDocBase，换句话就是对于war包，将docBase设置成解压后的目录中，然后就是设置完全的解析context.xml配置文件，设置专属的类加载器，解析web.xml配置文件，解析模块化jar包中的web配置，解析注解配置，循环调用ServletContainerIniter（可以自定义注册Filter，Servlet，Listener逻辑），反射创建Listeners，并调用其中的ServletContextListener的contextInitialized方法，创建Filter，包装成ApplicationFilterConfig，调用其init方法，创建servlet，并调用其init方法。


用过springboot的人都知道，springboot只需要通过一个main方法就能够启动，然后就可以直接在浏览器中敲入映射的地址就可以访问资源，那么springboot是如何将web服务器嵌入进去的人，这里我们只分析tomcat（因为我对tomcat更熟悉）

那么问题来了，这个Tomcat是在哪里启动的嘞！
springboot启动的web容器是ServletWebServerApplicationContext，这个类实现了onRefresh方法。

```
protected void org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.onRefresh() {
        //调用父类的onRefresh方法
		super.onRefresh();
		try {
		    //创建web服务器
		    //(*1*)
			createWebServer();
		}
		catch (Throwable ex) {
			throw new ApplicationContextException("Unable to start web server", ex);
		}
}

//(*1*)
private void ServletWebServerApplicationContext.createWebServer() {
        //在未创建web服务时，这里的webServer与ServletContext都是null
		WebServer webServer = this.webServer;
		ServletContext servletContext = getServletContext();
		if (webServer == null && servletContext == null) {
		    //获取创建webServer的工厂
			ServletWebServerFactory factory = getWebServerFactory();
			//我们先来看下getSelfInitializer()方法
			//(*1*)
			this.webServer = factory.getWebServer(getSelfInitializer());
		}
		else if (servletContext != null) {
			try {
			    //做了这么几件事，spring容器保存到servletContext中，注册于web相关的scope，比如requestScope，sessionScope，application级别的
			    //ServletContextScope，注册web相关的一些依赖，如ServletRequest，ServletResponse，jsf等
			    //注册servletContext，servletConfig，contextParameters，contextAttribute为单例bean到容器中
			    //重点：从容器中获取ServletContextInitializer，如果是ServletRegistrationBean，它可以向tomcat的context中注册Servlet
			    //FilterRegistrationBean注册Filter，ServletListenerRegistrationBean注册监听器
				getSelfInitializer().onStartup(servletContext);
			}
			catch (ServletException ex) {
				throw new ApplicationContextException("Cannot initialize servlet context",
						ex);
			}
		}
		//将servletContext作为属性源设置到Environment中去
		initPropertySources();
	}
	
//(*1*)
private org.springframework.boot.web.servlet.ServletContextInitializer getSelfInitializer() {
    //(*2*)
    //java8拉姆达语法
	return this::selfInitialize;
}

 //(*2*)
private void selfInitialize(ServletContext servletContext) throws ServletException {
    //将spring容器存储到ServletContext中
	prepareWebApplicationContext(servletContext);
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	//获取存在的scope
	ExistingWebApplicationScopes existingScopes = new ExistingWebApplicationScopes(
			beanFactory);
    //注册web相关的scope，比如RequestScope，SessionScope，ServletContextScope，设置一些web相关的依赖，比如ServletRequest，ServletResponse等
	WebApplicationContextUtils.registerWebApplicationScopes(beanFactory,
			getServletContext());
	//重新设置scope，覆盖旧的，并打印日志，提醒
	existingScopes.restore();
	//注册单例bean，比如ServletContext，ServletConfig，contextParameter，contextAttribute
	WebApplicationContextUtils.registerEnvironmentBeans(beanFactory,
			getServletContext());
	//servlet容器初始化器，主要用于注册Servlet，Filter，ServletListener
	for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
		beans.onStartup(servletContext);
	}
}
```

获取ServletWebServer工厂

```
protected ServletWebServerFactory org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.getWebServerFactory() {
		// Use bean names so that we don't consider the hierarchy
		//从容器中获取ServletWebServerFactory
		String[] beanNames = getBeanFactory()
				.getBeanNamesForType(ServletWebServerFactory.class);
		//如果没有直接抛错
		if (beanNames.length == 0) {
			throw new ApplicationContextException(
					"Unable to start ServletWebServerApplicationContext due to missing "
							+ "ServletWebServerFactory bean.");
		}
		//多了也不行
		if (beanNames.length > 1) {
			throw new ApplicationContextException(
					"Unable to start ServletWebServerApplicationContext due to multiple "
							+ "ServletWebServerFactory beans : "
							+ StringUtils.arrayToCommaDelimitedString(beanNames));
		}
		return getBeanFactory().getBean(beanNames[0], ServletWebServerFactory.class);
	}
```
从上面的代码上看，当我们有没有明确指定使用什么web容器的时候，springboot肯定在某个地方设置了ServletWebServerFactory，那么springboot会在哪里设置呢？在前一节的分析中我们知道，springboot会有些自动配置，那我们到自动配置里面找找看，果不其然发现了一个叫做ServletWebServerFactoryAutoConfiguration的自动配置

```
//元注解为@Component
@Configuration
//自动配置顺序为最高
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
//当存在ServletRequest时
@ConditionalOnClass(ServletRequest.class)
//当spring容器类型为Servlet时
@ConditionalOnWebApplication(type = Type.SERVLET)
//启动配置属性，实际上就是利用@Import注解引入其他的配置类
@EnableConfigurationProperties(ServerProperties.class)
//这里我们比较关心ServletWebServerFactoryConfiguration.EmbeddedTomcat.class
@Import({ ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
		ServletWebServerFactoryConfiguration.EmbeddedTomcat.class,
		ServletWebServerFactoryConfiguration.EmbeddedJetty.class,
		ServletWebServerFactoryConfiguration.EmbeddedUndertow.class })
public class ServletWebServerFactoryAutoConfiguration {
。。。。。。
}
```

嵌入的tomcat配置类

```
class ServletWebServerFactoryConfiguration {

	@Configuration
	//当存在Servlet，Tomcat，UpgradeProtocol时
	@ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class })
	//当当前容器中没有其他的ServletWebServerFactory类型的bean时，启用TomcatServletWebServerFactory
	@ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
	public static class EmbeddedTomcat {

		@Bean
		public TomcatServletWebServerFactory tomcatServletWebServerFactory() {
			return new TomcatServletWebServerFactory();
		}

	}
。。。。。。
}
```

获取webServer

```
public WebServer org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory.getWebServer(ServletContextInitializer... initializers) {
    //这个类主要用于维护Server实例，端口，密码，用户身份，角色，基路径等信息
	Tomcat tomcat = new Tomcat();
	//基路径，如果用户没有设置，那么生成临时路径
	File baseDir = (this.baseDirectory != null ? this.baseDirectory
			: createTempDir("tomcat"));
	tomcat.setBaseDir(baseDir.getAbsolutePath());
	//连接器，内部顺带创建协议处理器（默认Http11NioProtocol），这里的操作逻辑和tomcat源码分析的连接器处理是一样的
	//不过多的说明了，详细可查看tomcat源码分析文章
	Connector connector = new Connector(this.protocol);
	//添加连接器
	tomcat.getService().addConnector(connector);
	//定制连接器，比如设置端口，给协议处理器设置未来端点的绑定地址，编码等
	customizeConnector(connector);
	//关联上连接器
	tomcat.setConnector(connector);
	//是否自动部署（为true时，当tomcat管理的web应用资源发生修改时，tomcat会进行热部署）
	//这里设置为false，通常我们的内容发生变化的时候，由springboot进行热部署
	tomcat.getHost().setAutoDeploy(false);
	//配置引擎，添加管道阀，设置tomcat容器后台线程延时时间
	configureEngine(tomcat.getEngine());
	//添加偏好连接器，比如我添加了一个其他协议的连接器
	for (Connector additionalConnector : this.additionalTomcatConnectors) {
		tomcat.getService().addConnector(additionalConnector);
	}
	//准备tomcat的context容器
	prepareContext(tomcat.getHost(), initializers);
	return getTomcatWebServer(tomcat);
}
```

准备tomcat的context容器

```
protected void org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory.prepareContext(Host host, ServletContextInitializer[] initializers) {
    //获取有效的web目录
    //springboot尝试从DocumentRoot的代码所在目录中查找war
    //也会尝试是否存在src/main/webapp目录
	File documentRoot = getValidDocumentRoot();
	//创建嵌入的tomcat的context
	TomcatEmbeddedContext context = new TomcatEmbeddedContext();
	if (documentRoot != null) {
	    //设置根资源，tomcat可以通过根资源浏览其下所有资源
		context.setResources(new LoaderHidingResourceRoot(context));
	}
	context.setName(getContextPath());
	context.setDisplayName(getDisplayName());
	context.setPath(getContextPath());
	File docBase = (documentRoot != null ? documentRoot
			: createTempDir("tomcat-docbase"));
	//设置web应用的基路径，tomcat基于这个路径去获取资源
	context.setDocBase(docBase.getAbsolutePath());
	//添加生命周期监听器，这个监听器对CONFIGURE_START_EVENT（配置开始）感兴趣，主要用于
	//扫描web注解，比如@Resource，@DeclareRoles
	context.addLifecycleListener(new FixContextListener());
	//设置加载器
	context.setParentClassLoader(
			this.resourceLoader != null ? this.resourceLoader.getClassLoader()
					: ClassUtils.getDefaultClassLoader());
	//设置默认的Locale与字符集映射，比如(Locale.ENGLISH-》utf-8
	resetDefaultLocaleMapping(context);
	//添加定义的Locale与字符集映射
	addLocaleMappings(context);
	context.setUseRelativeRedirects(false);
	//配置tld文件跳过规则
	configureTldSkipPatterns(context);
	//创建WebappLoader，我们在tomcat源码分析的文章中有深入分析过tomcat为每个web应用创建类加载器的过程
	WebappLoader loader = new WebappLoader(context.getParentClassLoader());
	loader.setLoaderClass(TomcatEmbeddedWebappClassLoader.class.getName());
	//设置类加载模式，这里为true，表示使用父类先加载，双亲委派
	loader.setDelegate(true);
	//设置类加载器
	context.setLoader(loader);
	//是否注册默认的Servlet，也就是注册org.apache.catalina.servlets.DefaultServlet，映射路径为/
	if (isRegisterDefaultServlet()) {
		addDefaultServlet(context);
	}
	//注册jsp servlet，用于处理jsp
	if (shouldRegisterJspServlet()) {
		addJspServlet(context);
		addJasperInitializer(context);
	}
	//添加声明周期监听器，对CONFIGURE_START_EVENT（配置开始）感兴趣，主要用于注册资源信息
	context.addLifecycleListener(new StaticResourceConfigurer(context));
	//合并ServletContextInitializer，其实就是额外添加ServletContextInitializer，比如用于向ServletContext设置初始化参数等等
	ServletContextInitializer[] initializersToUse = mergeInitializers(initializers);
	//给host添加子容器
	host.addChild(context);
	//配置web应用容器，主要是设置session，添加生命周期监听器，设置TomcatStarter
	//（一个复合ServletContainerInitializer，维护着多个ServletContainerInitializer）
	//添加管道阀，添加错误提示页面，添加扩展名与媒体类型的映射，定制context
	configureContext(context, initializersToUse);
	//空操作
	postProcessContext(context);
}
```
一切准备就绪，那么就可以启动服务了
getTomcatWebServer

```
protected TomcatWebServer org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory.getTomcatWebServer(Tomcat tomcat) {
        //(*1*)
		return new TomcatWebServer(tomcat, getPort() >= 0);
}

//(*1*)
public org.springframework.boot.web.embedded.tomcat.TomcatWebServer.TomcatWebServer(Tomcat tomcat, boolean autoStart) {
	Assert.notNull(tomcat, "Tomcat Server must not be null");
	this.tomcat = tomcat;
	this.autoStart = autoStart;
	//(*2*)
	initialize();
}

//(*2*)
private void initialize() throws WebServerException {
	TomcatWebServer.logger
			.info("Tomcat initialized with port(s): " + getPortsDescription(false));
	synchronized (this.monitor) {
		try {
		    //设置Engine名
			addInstanceIdToEngineName();
            //从Host中获取Context容器
			Context context = findContext();
			//又添加了一个生命周期监听器，这个监听器对启动事件感兴趣
			context.addLifecycleListener((event) -> {
				if (context.equals(event.getSource())
						&& Lifecycle.START_EVENT.equals(event.getType())) {
					// Remove service connectors so that protocol binding doesn't
					// happen when the service is started.
					//移除service中的连接器，并存储到org.springframework.boot.web.embedded.tomcat.TomcatWebServer.serviceConnectors
					//中，防止启动tomcat服务时进行地址绑定
					removeServiceConnectors();
				}
			});

			// Start the server to trigger initialization listeners
			//启动服务，内部调用Tomcat的Server的start方法，启动方式可查阅tomcat源码分析文章，此处不过多的赘述
			this.tomcat.start();

			// We can re-throw failure exception directly in the main thread
			//如果tomcat启动的过程中，ServletContextInier抛出了错误，或者启动状态不是Started，也就是启动不成功
			//那么抛错错误
			rethrowDeferredStartupExceptions();

			try {
			    //绑定类加载器
				ContextBindings.bindClassLoader(context, context.getNamingToken(),
						getClass().getClassLoader());
			}
			catch (NamingException ex) {
				// Naming is not enabled. Continue
			}

			// Unlike Jetty, all Tomcat threads are daemon threads. We create a
			// blocking non-daemon to stop immediate shutdown
			//启动阻塞线程，调用server的await方法，防止退出
			startDaemonAwaitThread();
		}
		catch (Exception ex) {
			throw new WebServerException("Unable to start embedded Tomcat", ex);
		}
	}
}
```

从上面的代码中，我们注意到了一点，那就是在spring容器在onRefresh阶段启动的tomcat并没有启动Connector，因为移除了Service中的Connector，并把Connector储存到了TomcatWebServer类中的serviceConnectors属性中，那么为什么要移除它呢？那它在什么时候才会去启动Connector呢？

首先移除Connector是因为spring的容器还未完全刷新完，比如还没有注册监听器，还没有初始化eager的bean

在finishRefresh的时候，springboot就会将connector设置回service中

```
protected void org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.finishRefresh() {
	super.finishRefresh();
	//(*1*)
	WebServer webServer = startWebServer();
	if (webServer != null) {
		publishEvent(new ServletWebServerInitializedEvent(webServer, this));
	}
}

//(*1*)
private WebServer startWebServer() {
	WebServer webServer = this.webServer;
	if (webServer != null) {
	    //(*2*)
		webServer.start();
	}
	return webServer;
}

//(*2*)
public void start() throws WebServerException {
	synchronized (this.monitor) {
		if (this.started) {
			return;
		}
		try {
		    //将之前移除的Connector重新设置回去
			addPreviouslyRemovedConnectors();
			Connector connector = this.tomcat.getConnector();
			if (connector != null && this.autoStart) {
			    //启动连接
				startConnector();
			}
			//检查Connector是否启动成功
			checkThatConnectorsHaveStarted();
			this.started = true;
		    。。。。。。
}
```

到这里tomcat的启动就算完成了，但是还有一个疑问，那就是spring的DispatcherServlet是什么时候设置进去的？？？

如果进行debug的话，你会发现，springboot创建了一个TomcatStarter，它实现了ServletContainerInitializer,它是个复合类，它维护着一个ServletContainerInitializer集合，这个集合中的内容有部分是从spring容器中获取的，通过debug可以看到其中包含一个用于注册DispatcherServlet的ServletRegistrationBean，TomcatStarter又被springboot注册到了tomcat，我们在[tomcat源码分析](https://blog.csdn.net/qq_27785239/article/category/9206820)的文章中提到过，实现了ServletContainerInitializer的类会参与StandardContext（web应用）的启动，这个时候你可以向tomcat动态注册servlet，filter，listener。

```
protected void org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory.configureContext(Context context,
		ServletContextInitializer[] initializers) {
	//创建TomcatStarter，一个复合类
	TomcatStarter starter = new TomcatStarter(initializers);
	if (context instanceof TomcatEmbeddedContext) {
		// Should be true
		((TomcatEmbeddedContext) context).setStarter(starter);
	}
	//向tomcat的web应用context中注册ServletContainerInitializer
	context.addServletContainerInitializer(starter, NO_CLASSES);
	。。。。。。
}
```
TomcatStarter的onStartup方法

```
public void onStartup(Set<Class<?>> classes, ServletContext servletContext)
			throws ServletException {
	try {
	    //循环调用每个ServletContextInitializer的onStartup方法
		for (ServletContextInitializer initializer : this.initializers) {
			initializer.onStartup(servletContext);
		}
	}
	catch (Exception ex) {
		this.startUpException = ex;
		// Prevent Tomcat from logging and re-throwing when we know we can
		// deal with it in the main thread, but log for information here.
		if (logger.isErrorEnabled()) {
			logger.error("Error starting Tomcat context. Exception: "
					+ ex.getClass().getName() + ". Message: " + ex.getMessage());
		}
	}
	}
```
这样springboot的DispatcherServlet就会被注册到tomcat的web应用context中，那问题又来了，这个DispatcherServlet是什么时候后被包装成ServletRegistrationBean注册到spring的容器中的呢？

我们没有在任何地方配置这个DispatcherServlet，很显然它肯定是被springboot自动配置进去的，所以我们再次从springboot的自动配置spring.factories中查找，果不其然找到了一个叫做DispatcherServletAutoConfiguration的配置类，其内部类DispatcherServletRegistrationConfiguration配置了DispatcherServlet的ServletRegistrationBean

```
@Bean(name = DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
		@ConditionalOnBean(value = DispatcherServlet.class, name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
public ServletRegistrationBean<DispatcherServlet> dispatcherServletRegistration(
		DispatcherServlet dispatcherServlet) {
	ServletRegistrationBean<DispatcherServlet> registration = new ServletRegistrationBean<>(
			dispatcherServlet,
			this.serverProperties.getServlet().getServletMapping());
	registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
	registration.setLoadOnStartup(
			this.webMvcProperties.getServlet().getLoadOnStartup());
	if (this.multipartConfig != null) {
		registration.setMultipartConfig(this.multipartConfig);
	}
	return registration;
}
```
DispatcherServlet的初始化，将会启动spring的web容器，这里面的启动源码，可参考[springMVC源码分析](https://blog.csdn.net/qq_27785239/article/category/9202697)文章。
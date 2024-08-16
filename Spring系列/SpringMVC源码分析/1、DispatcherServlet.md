SpringMVC是基于spring形成的MVC框架，所以本系列的也是基于spring源码分析系列之上的，涉及到spring的内容，本篇不会再次赘述，如有遗忘，请自行查阅，话不多说，let's go!

我们启动springMVC的时候会在web.xml中配置一个Servlet，操作如下：

```
<servlet>
		<servlet-name>springMVC</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>classpath:spring/springMVC-servlet.xml</param-value>
		</init-param>
		<load-on-startup>1</load-on-startup>
	</servlet>
	<servlet-mapping>
		<servlet-name>springMVC</servlet-name>
		<url-pattern>/</url-pattern>
	</servlet-mapping>
```
可以看到上面配置了一个Servlet，并且其配置load-on-startup是1，所以在tomcat启动我们web应用的时候就会构建DispatcherServlet实例并进行初始化，我们来看下它的继承结构

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/5216e160c65741549f9a7d96a1d83381.png)

首先我们先从HttpServletBean入手，这个类实现了servlet的init方法

```
public final void org.springframework.web.servlet.HttpServletBean.init() throws ServletException {
		if (logger.isDebugEnabled()) {
			logger.debug("Initializing servlet '" + getServletName() + "'");
		}

		// Set bean properties from init parameters.
		try {
		    //创建一个属性键值对复合对象，构造器中主要就是对Servlet的初始化参数进行校验
			PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
			//构建BeanWrapper，这个BeanWrapper具有设置属性值，属性值类型转换的能力，当然转化器，编辑需要自己额外设置进去
			BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
			//构建ServletContex资源加载器，通过tomcat提供的资源访问
			ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
			//注册自定义ResourceEditor，将字符串转换成Resource对象
			bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
			//初始化，这里是空实现，像这种在spring中一般都是注册转换服务和注册自定编辑器用的
			initBeanWrapper(bw);
			//设置属性值，比如contextConfigLocation，在上面的配置中它的值为classpath:spring/springMVC-servlet.xml
			bw.setPropertyValues(pvs, true);
		}
		catch (BeansException ex) {
			logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
			throw ex;
		}

		// Let subclasses do whatever initialization they like.
		//初始化ServletBean
		initServletBean();

		if (logger.isDebugEnabled()) {
			logger.debug("Servlet '" + getServletName() + "' configured successfully");
		}
	}
```
> initServletBean()

```
protected final void org.springframework.web.servlet.FrameworkServlet.initServletBean() throws ServletException {
        //不用看了，就是想打印日志，告诉你FrameworkServlet在初始化
		getServletContext().log("Initializing Spring FrameworkServlet '" + getServletName() + "'");
		if (this.logger.isInfoEnabled()) {
			this.logger.info("FrameworkServlet '" + getServletName() + "': initialization started");
		}
		long startTime = System.currentTimeMillis();

		try {
		    //构建WebApplicationContext
		    //(*1*)
			this.webApplicationContext = initWebApplicationContext();
			//初始化FrameworkServlet
			initFrameworkServlet();
		}
		catch (ServletException ex) {
			this.logger.error("Context initialization failed", ex);
			throw ex;
		}
		catch (RuntimeException ex) {
			this.logger.error("Context initialization failed", ex);
			throw ex;
		}

		if (this.logger.isInfoEnabled()) {
			long elapsedTime = System.currentTimeMillis() - startTime;
			this.logger.info("FrameworkServlet '" + getServletName() + "': initialization completed in " +
					elapsedTime + " ms");
		}
	}
	
	//(*1*)
	protected WebApplicationContext initWebApplicationContext() {
	    //从Servletcontext中获取根容器，很明显这个根容器就是我们在web.xml中配置的那个监听器启动的容器
		WebApplicationContext rootContext =
				WebApplicationContextUtils.getWebApplicationContext(getServletContext());
		WebApplicationContext wac = null;
        //如果已经启动了SpringMVC的webApplicationContext
		if (this.webApplicationContext != null) {
			// A context instance was injected at construction time -> use it
			wac = this.webApplicationContext;
			if (wac instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
				if (!cwac.isActive()) {
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					if (cwac.getParent() == null) {
						// The context instance was injected without an explicit parent -> set
						// the root application context (if any; may be null) as the parent
						//如果没有设置根容器，那么将从ServletContext就获取的根容器设置进去
						cwac.setParent(rootContext);
					}
					//配置和刷新容器，这里面的逻辑和spring源码分析中的容器配置和刷新是一样的
					//(*2*)
					configureAndRefreshWebApplicationContext(cwac);
				}
			}
		}
		if (wac == null) {
			// No context instance was injected at construction time -> see if one
			// has been registered in the servlet context. If one exists, it is assumed
			// that the parent context (if any) has already been set and that the
			// user has performed any initialization such as setting the context id
			//尝试从ServletContext获取WebApplicationContext，如果想在ServletContext中设置值，我们
			//可以实现ServletContainerInitializer参与tomcat的StandardContext的启动过程
			//实现类配置在META-INF/services/javax.servlet.ServletContainerInitializer文件中，换行符分割
			//使用@HandlesTypes注解标注自己感兴趣的类型
			//方法签名为：void onStartup(Set<Class<?>> c, ServletContext ctx) throws ServletException
			wac = findWebApplicationContext();
		}
		if (wac == null) {
			// No context instance is defined for this servlet -> create a local one
			//创建默认的WebApplicationContext，它的实例是org.springframework.web.context.support.XmlWebApplicationContext
			//创建并设置父容器为上面从ServletContext中获取的根容器，并且对容器进行配置和刷新
			wac = createWebApplicationContext(rootContext);
		}
        //refreshEventReceived标记是否已经刷新，如果还没有刷新，那么需要触发刷新
		if (!this.refreshEventReceived) {
			// Either the context is not a ConfigurableApplicationContext with refresh
			// support or the context injected at construction time had already been
			// refreshed -> trigger initial onRefresh manually here.
			onRefresh(wac);
		}

        //是否将SpringMVC容器设置到servletContext中
		if (this.publishContext) {
			// Publish the context as a servlet context attribute.
			//FrameworkServlet.class.getName() + ".CONTEXT." + getServletName()
			String attrName = getServletContextAttributeName();
			//设置
			getServletContext().setAttribute(attrName, wac);
			if (this.logger.isDebugEnabled()) {
				this.logger.debug("Published WebApplicationContext of servlet '" + getServletName() +
						"' as ServletContext attribute with name [" + attrName + "]");
			}
		}

		return wac;
	}
	
	//(*2*)
	protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
		if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
		    
		    //设置contextId，可以通过web.xml中指定
			if (this.contextId != null) {
				wac.setId(this.contextId);
			}
			else {
				// Generate default id...
				wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
						ObjectUtils.getDisplayString(getServletContext().getContextPath()) + "/" + getServletName());
			}
		}
        //设置ServletContext
		wac.setServletContext(getServletContext());
		//设置ServletConfig
		wac.setServletConfig(getServletConfig());
		//获取命名空间，通常我们使用默认的命名空间getServletName() + "-servlet"
		wac.setNamespace(getNamespace());
		//添加监听器，这是一种装饰器模式，当前发生容器刷新的源类型是当前的容器，才会触发容器刷新事件
		//这里触发是直接调用org.springframework.web.servlet.DispatcherServlet.onRefresh(ApplicationContext)方法
		wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));

		// The wac environment's #initPropertySources will be called in any case when the context
		// is refreshed; do it eagerly here to ensure servlet property sources are in place for
		// use in any post-processing or initialization that occurs below prior to #refresh
		//获取环境对象
		ConfigurableEnvironment env = wac.getEnvironment();
		if (env instanceof ConfigurableWebEnvironment) {
		    //设置ServletContext属性资源和ServletConfig属性资源
			((ConfigurableWebEnvironment) env).initPropertySources(getServletContext(), getServletConfig());
		}
        //后置处理容器，默认啥都没有实现
		postProcessWebApplicationContext(wac);
		//调用ApplicationContextInitializer
		applyInitializers(wac);
		//刷新容器，这个就不过多赘述了，在spring源码分析中有详细的分析
		wac.refresh();
	}
```
FrameworkServlet的initFrameworkServlet方法啥都没有干
```
protected void initFrameworkServlet() throws ServletException {
	}
```

下面这段代码是DispatcherServlet的静态块

```
static {
		// Load default strategy implementations from properties file.
		// This is currently strictly internal and not meant to be customized
		// by application developers.
		try {
		    //加载DispatcherServlet路径下的DispatcherServlet.properties配置文件
			ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, DispatcherServlet.class);
			defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
		}
		catch (IOException ex) {
			throw new IllegalStateException("Could not load 'DispatcherServlet.properties': " + ex.getMessage());
		}
	}
```
一下便是DispatcherServlet.properties配置文件的内容

```
# Default implementation classes for DispatcherServlet's strategy interfaces.
# Used as fallback when no matching beans are found in the DispatcherServlet context.
# Not meant to be customized by application developers.

# Local解析器
org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver
# 主题解析器
org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver
# 处理器映射，默认才两个，一个是通过BeanName进行映射的HandlerMapping，另一个是通过注解，比如@RequestMapping这样的注解
org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping
# HandlerAdapter
org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
	org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter
# 全局异常处理器
org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver
# 将请求路径转换成视图名的转换器
org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator
# 视图解析器，这个一般就是用于jsp的视图解析器
org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver
# FlashMapManager管理器，用于管理重定向保存参数的管理器
org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager

```

> org.springframework.web.servlet.DispatcherServlet.onRefresh(ApplicationContext)

```
protected void onRefresh(ApplicationContext context) {
		initStrategies(context);
}

protected void initStrategies(ApplicationContext context) {
		initMultipartResolver(context);
		initLocaleResolver(context);
		initThemeResolver(context);
		initHandlerMappings(context);
		initHandlerAdapters(context);
		initHandlerExceptionResolvers(context);
		initRequestToViewNameTranslator(context);
		initViewResolvers(context);
		initFlashMapManager(context);
}
```
> initMultipartResolver

```
private void org.springframework.web.servlet.DispatcherServlet.initMultipartResolver(ApplicationContext context) {
		try {
		    //MULTIPART_RESOLVER_BEAN_NAME -> multipartResolver，上传文件相关
			this.multipartResolver = context.getBean(MULTIPART_RESOLVER_BEAN_NAME, MultipartResolver.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Using MultipartResolver [" + this.multipartResolver + "]");
			}
		}
		catch (NoSuchBeanDefinitionException ex) {
			// Default is no multipart resolver.
			this.multipartResolver = null;
			if (logger.isDebugEnabled()) {
				logger.debug("Unable to locate MultipartResolver with name '" + MULTIPART_RESOLVER_BEAN_NAME +
						"': no multipart request handling provided");
			}
		}
	}
```
> initLocaleResolver

```
private void org.springframework.web.servlet.DispatcherServlet.initLocaleResolver(ApplicationContext context) {
		try {
		    //LOCALE_RESOLVER_BEAN_NAME -> localeResolver
			this.localeResolver = context.getBean(LOCALE_RESOLVER_BEAN_NAME, LocaleResolver.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Using LocaleResolver [" + this.localeResolver + "]");
			}
		}
		catch (NoSuchBeanDefinitionException ex) {
			// We need to use the default.
			this.localeResolver = getDefaultStrategy(context, LocaleResolver.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Unable to locate LocaleResolver with name '" + LOCALE_RESOLVER_BEAN_NAME +
						"': using default [" + this.localeResolver + "]");
			}
		}
	}
```

> initThemeResolver

```
private void initThemeResolver(ApplicationContext context) {
		try {
		    //THEME_RESOLVER_BEAN_NAME -> themeResolver
			this.themeResolver = context.getBean(THEME_RESOLVER_BEAN_NAME, ThemeResolver.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Using ThemeResolver [" + this.themeResolver + "]");
			}
		}
		catch (NoSuchBeanDefinitionException ex) {
			// We need to use the default.
			this.themeResolver = getDefaultStrategy(context, ThemeResolver.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Unable to locate ThemeResolver with name '" + THEME_RESOLVER_BEAN_NAME +
						"': using default [" + this.themeResolver + "]");
			}
		}
	}
```
> initHandlerMappings

```
private void initHandlerMappings(ApplicationContext context) {
		this.handlerMappings = null;
        //是否自动装配HandlerMapping
		if (this.detectAllHandlerMappings) {
			// Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
			//从容器中获取HandlerMapping
			//这个方法我们在spring源码分析中分析过
			Map<String, HandlerMapping> matchingBeans =
					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.handlerMappings = new ArrayList<HandlerMapping>(matchingBeans.values());
				// We keep HandlerMappings in sorted order.
				//排序
				AnnotationAwareOrderComparator.sort(this.handlerMappings);
			}
		}
		else {
			try {
			    //HANDLER_MAPPING_BEAN_NAME -> handlerMapping, 从容器中获取beanName为handlerMapping，类型为HandlerMapping的bean
				HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
				this.handlerMappings = Collections.singletonList(hm);
			}
			catch (NoSuchBeanDefinitionException ex) {
				// Ignore, we'll add a default HandlerMapping later.
			}
		}

		// Ensure we have at least one HandlerMapping, by registering
		// a default HandlerMapping if no other mappings are found.
		//如果没有，那么就从配置文件中获取默认的handlerMapping
		if (this.handlerMappings == null) {
		
		    //从DispatcherServlet.properties中获取
			this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
			if (logger.isDebugEnabled()) {
				logger.debug("No HandlerMappings found in servlet '" + getServletName() + "': using default");
			}
		}
	}
```

> initHandlerAdapters

```
private void initHandlerAdapters(ApplicationContext context) {
		this.handlerAdapters = null;
        //自动装配
		if (this.detectAllHandlerAdapters) {
			// Find all HandlerAdapters in the ApplicationContext, including ancestor contexts.
			//从容器中获取HandlerAdapter
			Map<String, HandlerAdapter> matchingBeans =
					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerAdapter.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.handlerAdapters = new ArrayList<HandlerAdapter>(matchingBeans.values());
				// We keep HandlerAdapters in sorted order.
				AnnotationAwareOrderComparator.sort(this.handlerAdapters);
			}
		}
		else {
			try {
			    //HANDLER_ADAPTER_BEAN_NAME -》 handlerAdapter
				HandlerAdapter ha = context.getBean(HANDLER_ADAPTER_BEAN_NAME, HandlerAdapter.class);
				this.handlerAdapters = Collections.singletonList(ha);
			}
			catch (NoSuchBeanDefinitionException ex) {
				// Ignore, we'll add a default HandlerAdapter later.
			}
		}

		// Ensure we have at least some HandlerAdapters, by registering
		// default HandlerAdapters if no other adapters are found.
		//从配置文件中获取
		if (this.handlerAdapters == null) {
			this.handlerAdapters = getDefaultStrategies(context, HandlerAdapter.class);
			if (logger.isDebugEnabled()) {
				logger.debug("No HandlerAdapters found in servlet '" + getServletName() + "': using default");
			}
		}
	}
```

> initHandlerExceptionResolvers

```
private void initHandlerExceptionResolvers(ApplicationContext context) {
		this.handlerExceptionResolvers = null;
        //自动装配异常处理器
		if (this.detectAllHandlerExceptionResolvers) {
			// Find all HandlerExceptionResolvers in the ApplicationContext, including ancestor contexts.
			Map<String, HandlerExceptionResolver> matchingBeans = BeanFactoryUtils
					.beansOfTypeIncludingAncestors(context, HandlerExceptionResolver.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.handlerExceptionResolvers = new ArrayList<HandlerExceptionResolver>(matchingBeans.values());
				// We keep HandlerExceptionResolvers in sorted order.
				AnnotationAwareOrderComparator.sort(this.handlerExceptionResolvers);
			}
		}
		else {
			try {
			    //HANDLER_EXCEPTION_RESOLVER_BEAN_NAME -> handlerExceptionResolver
				HandlerExceptionResolver her =
						context.getBean(HANDLER_EXCEPTION_RESOLVER_BEAN_NAME, HandlerExceptionResolver.class);
				this.handlerExceptionResolvers = Collections.singletonList(her);
			}
			catch (NoSuchBeanDefinitionException ex) {
				// Ignore, no HandlerExceptionResolver is fine too.
			}
		}

		// Ensure we have at least some HandlerExceptionResolvers, by registering
		// default HandlerExceptionResolvers if no other resolvers are found.
		//获取默认的异常处理器
		if (this.handlerExceptionResolvers == null) {
			this.handlerExceptionResolvers = getDefaultStrategies(context, HandlerExceptionResolver.class);
			if (logger.isDebugEnabled()) {
				logger.debug("No HandlerExceptionResolvers found in servlet '" + getServletName() + "': using default");
			}
		}
	}
```

> initRequestToViewNameTranslator

```
private void initRequestToViewNameTranslator(ApplicationContext context) {
		try {
		    //REQUEST_TO_VIEW_NAME_TRANSLATOR_BEAN_NAME -> viewNameTranslator
			this.viewNameTranslator =
					context.getBean(REQUEST_TO_VIEW_NAME_TRANSLATOR_BEAN_NAME, RequestToViewNameTranslator.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Using RequestToViewNameTranslator [" + this.viewNameTranslator + "]");
			}
		}
		catch (NoSuchBeanDefinitionException ex) {
			// We need to use the default.
			this.viewNameTranslator = getDefaultStrategy(context, RequestToViewNameTranslator.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Unable to locate RequestToViewNameTranslator with name '" +
						REQUEST_TO_VIEW_NAME_TRANSLATOR_BEAN_NAME + "': using default [" + this.viewNameTranslator +
						"]");
			}
		}
	}
```
> initViewResolvers

```
private void initViewResolvers(ApplicationContext context) {
		this.viewResolvers = null;
        //从容器中获取视图解析器
		if (this.detectAllViewResolvers) {
			// Find all ViewResolvers in the ApplicationContext, including ancestor contexts.
			Map<String, ViewResolver> matchingBeans =
					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, ViewResolver.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.viewResolvers = new ArrayList<ViewResolver>(matchingBeans.values());
				// We keep ViewResolvers in sorted order.
				AnnotationAwareOrderComparator.sort(this.viewResolvers);
			}
		}
		else {
			try {
			    //VIEW_RESOLVER_BEAN_NAME -> viewResolver 
				ViewResolver vr = context.getBean(VIEW_RESOLVER_BEAN_NAME, ViewResolver.class);
				this.viewResolvers = Collections.singletonList(vr);
			}
			catch (NoSuchBeanDefinitionException ex) {
				// Ignore, we'll add a default ViewResolver later.
			}
		}

		// Ensure we have at least one ViewResolver, by registering
		// a default ViewResolver if no other resolvers are found.
		//从配置文件中获取
		if (this.viewResolvers == null) {
			this.viewResolvers = getDefaultStrategies(context, ViewResolver.class);
			if (logger.isDebugEnabled()) {
				logger.debug("No ViewResolvers found in servlet '" + getServletName() + "': using default");
			}
		}
	}

```

> initFlashMapManager

```
private void initFlashMapManager(ApplicationContext context) {
		try {
		    //FLASH_MAP_MANAGER_BEAN_NAME -> flashMapManager 闪存管理器
			this.flashMapManager = context.getBean(FLASH_MAP_MANAGER_BEAN_NAME, FlashMapManager.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Using FlashMapManager [" + this.flashMapManager + "]");
			}
		}
		catch (NoSuchBeanDefinitionException ex) {
			// We need to use the default.
			//获取配置文件中闪存管理器，用于存储属性，重定向时依然可以获取到上次请求的属性值
			this.flashMapManager = getDefaultStrategy(context, FlashMapManager.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Unable to locate FlashMapManager with name '" +
						FLASH_MAP_MANAGER_BEAN_NAME + "': using default [" + this.flashMapManager + "]");
			}
		}
	}

```

一切准备就绪后，那么只要等待请求进来就行了

顺便说明一下，我们除了使用xml进行配置mvc之外，我们还可以通过注解的方式，配置如下

```
@Configuration
@EnableWebMvc
public class MyWebMvcConfigurer extends WebMvcConfigurerAdapter {

	@Override
	public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
		//extension -》 mediaType
		configurer.mediaType("html", MediaType.ALL);
		
	}
	
}
```

可以看到这个类使用了@Configuration注解，我们在spring源码分析中分析过这个注解，所以不再赘述，那么上面还有个@EnableWebMvc，这个注解是干啥的呢？通常这种跟@Configuration注解一起使用的注解都是一种配置注解，看它的名字也能猜出它就是用于配置mvc的，我们看看这个注解的实现

```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc {
}
```

发现没有，这个注解有个元注解是@Import，我们在spring源码分析系列中分析过，这个注解就是引入其他配置项的注解，我们来看下它引入的配置类的配置

```
@Configuration
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {

	private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();

    //注入WebMvcConfigurer集合，我们在上面就定义了一个MyWebMvcConfigurer
	@Autowired(required = false)
	public void setConfigurers(List<WebMvcConfigurer> configurers) {
		if (configurers == null || configurers.isEmpty()) {
			return;
		}
		this.configurers.addWebMvcConfigurers(configurers);
	}


	@Override
	protected void addInterceptors(InterceptorRegistry registry) {
		this.configurers.addInterceptors(registry);
	}

	@Override
	protected void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
		this.configurers.configureContentNegotiation(configurer);
	}

	。。。。。。

}
```
> WebMvcConfigurationSupport

```
public class WebMvcConfigurationSupport implements ApplicationContextAware, ServletContextAware {
    。。。。。。省略部分代码
    
	@Bean
	public RequestMappingHandlerMapping requestMappingHandlerMapping() {
	
	    //创建RequestMappingHandlerMapping
		RequestMappingHandlerMapping handlerMapping = createRequestMappingHandlerMapping();
		handlerMapping.setOrder(0);
		handlerMapping.setInterceptors(getInterceptors());
		handlerMapping.setContentNegotiationManager(mvcContentNegotiationManager());
		handlerMapping.setCorsConfigurations(getCorsConfigurations());

        //设置路径匹配规则，比如是否后缀匹配，是否尾部/匹配
		PathMatchConfigurer configurer = getPathMatchConfigurer();
		if (configurer.isUseSuffixPatternMatch() != null) {
			handlerMapping.setUseSuffixPatternMatch(configurer.isUseSuffixPatternMatch());
		}
		if (configurer.isUseRegisteredSuffixPatternMatch() != null) {
			handlerMapping.setUseRegisteredSuffixPatternMatch(configurer.isUseRegisteredSuffixPatternMatch());
		}
		if (configurer.isUseTrailingSlashMatch() != null) {
			handlerMapping.setUseTrailingSlashMatch(configurer.isUseTrailingSlashMatch());
		}
		if (configurer.getPathMatcher() != null) {
			handlerMapping.setPathMatcher(configurer.getPathMatcher());
		}
		if (configurer.getUrlPathHelper() != null) {
			handlerMapping.setUrlPathHelper(configurer.getUrlPathHelper());
		}

		return handlerMapping;
	}

	@Bean
	public ContentNegotiationManager mvcContentNegotiationManager() {
		if (this.contentNegotiationManager == null) {
		    //创建内容协商
			ContentNegotiationConfigurer configurer = new ContentNegotiationConfigurer(this.servletContext);
			//配置默认的媒体类型
			configurer.mediaTypes(getDefaultMediaTypes());
			configureContentNegotiation(configurer);
			try {
				this.contentNegotiationManager = configurer.getContentNegotiationManager();
			}
			catch (Exception ex) {
				throw new BeanInitializationException("Could not create ContentNegotiationManager", ex);
			}
		}
		//返回内容协商管理器
		return this.contentNegotiationManager;
	}

	@Bean
	public HandlerMapping viewControllerHandlerMapping() {
		ViewControllerRegistry registry = new ViewControllerRegistry();
		registry.setApplicationContext(this.applicationContext);
		addViewControllers(registry);

		AbstractHandlerMapping handlerMapping = registry.getHandlerMapping();
		handlerMapping = (handlerMapping != null ? handlerMapping : new EmptyHandlerMapping());
		handlerMapping.setPathMatcher(mvcPathMatcher());
		handlerMapping.setUrlPathHelper(mvcUrlPathHelper());
		handlerMapping.setInterceptors(getInterceptors());
		handlerMapping.setCorsConfigurations(getCorsConfigurations());
		return handlerMapping;
	}


	@Bean
	public BeanNameUrlHandlerMapping beanNameHandlerMapping() {
		BeanNameUrlHandlerMapping mapping = new BeanNameUrlHandlerMapping();
		mapping.setOrder(2);
		mapping.setInterceptors(getInterceptors());
		mapping.setCorsConfigurations(getCorsConfigurations());
		return mapping;
	}


	@Bean
	public HandlerMapping resourceHandlerMapping() {
		ResourceHandlerRegistry registry = new ResourceHandlerRegistry(this.applicationContext, this.servletContext);
		addResourceHandlers(registry);

		AbstractHandlerMapping handlerMapping = registry.getHandlerMapping();
		if (handlerMapping != null) {
			handlerMapping.setPathMatcher(mvcPathMatcher());
			handlerMapping.setUrlPathHelper(mvcUrlPathHelper());
			handlerMapping.setInterceptors(new HandlerInterceptor[] {
					new ResourceUrlProviderExposingInterceptor(mvcResourceUrlProvider())});
			handlerMapping.setCorsConfigurations(getCorsConfigurations());
		}
		else {
			handlerMapping = new EmptyHandlerMapping();
		}
		return handlerMapping;
	}

	@Bean
	public ResourceUrlProvider mvcResourceUrlProvider() {
		ResourceUrlProvider urlProvider = new ResourceUrlProvider();
		UrlPathHelper pathHelper = getPathMatchConfigurer().getUrlPathHelper();
		if (pathHelper != null) {
			urlProvider.setUrlPathHelper(pathHelper);
		}
		PathMatcher pathMatcher = getPathMatchConfigurer().getPathMatcher();
		if (pathMatcher != null) {
			urlProvider.setPathMatcher(pathMatcher);
		}
		return urlProvider;
	}


	@Bean
	public HandlerMapping defaultServletHandlerMapping() {
		DefaultServletHandlerConfigurer configurer = new DefaultServletHandlerConfigurer(servletContext);
		configureDefaultServletHandling(configurer);
		AbstractHandlerMapping handlerMapping = configurer.getHandlerMapping();
		handlerMapping = handlerMapping != null ? handlerMapping : new EmptyHandlerMapping();
		return handlerMapping;
	}

	@Bean
	public RequestMappingHandlerAdapter requestMappingHandlerAdapter() {
		List<HandlerMethodArgumentResolver> argumentResolvers = new ArrayList<HandlerMethodArgumentResolver>();
		addArgumentResolvers(argumentResolvers);

		List<HandlerMethodReturnValueHandler> returnValueHandlers = new ArrayList<HandlerMethodReturnValueHandler>();
		addReturnValueHandlers(returnValueHandlers);

		RequestMappingHandlerAdapter adapter = new RequestMappingHandlerAdapter();
		adapter.setContentNegotiationManager(mvcContentNegotiationManager());
		adapter.setMessageConverters(getMessageConverters());
		adapter.setWebBindingInitializer(getConfigurableWebBindingInitializer());
		adapter.setCustomArgumentResolvers(argumentResolvers);
		adapter.setCustomReturnValueHandlers(returnValueHandlers);

		if (jackson2Present) {
			List<RequestBodyAdvice> requestBodyAdvices = new ArrayList<RequestBodyAdvice>();
			requestBodyAdvices.add(new JsonViewRequestBodyAdvice());
			adapter.setRequestBodyAdvice(requestBodyAdvices);

			List<ResponseBodyAdvice<?>> responseBodyAdvices = new ArrayList<ResponseBodyAdvice<?>>();
			responseBodyAdvices.add(new JsonViewResponseBodyAdvice());
			adapter.setResponseBodyAdvice(responseBodyAdvices);
		}

		AsyncSupportConfigurer configurer = new AsyncSupportConfigurer();
		configureAsyncSupport(configurer);

		if (configurer.getTaskExecutor() != null) {
			adapter.setTaskExecutor(configurer.getTaskExecutor());
		}
		if (configurer.getTimeout() != null) {
			adapter.setAsyncRequestTimeout(configurer.getTimeout());
		}
		adapter.setCallableInterceptors(configurer.getCallableInterceptors());
		adapter.setDeferredResultInterceptors(configurer.getDeferredResultInterceptors());

		return adapter;
	}



	@Bean
	public FormattingConversionService mvcConversionService() {
	    //配置默认的转换服务
		FormattingConversionService conversionService = new DefaultFormattingConversionService();
		addFormatters(conversionService);
		return conversionService;
	}


	@Bean
	public Validator mvcValidator() {
	    //校验器，用于校验参数，这个后面学习handlerAdapter会提到
		Validator validator = getValidator();
		if (validator == null) {
			if (ClassUtils.isPresent("javax.validation.Validator", getClass().getClassLoader())) {
				Class<?> clazz;
				try {
					String className = "org.springframework.validation.beanvalidation.OptionalValidatorFactoryBean";
					clazz = ClassUtils.forName(className, WebMvcConfigurationSupport.class.getClassLoader());
				}
				catch (ClassNotFoundException ex) {
					throw new BeanInitializationException("Could not find default validator class", ex);
				}
				catch (LinkageError ex) {
					throw new BeanInitializationException("Could not load default validator class", ex);
				}
				validator = (Validator) BeanUtils.instantiate(clazz);
			}
			else {
				validator = new NoOpValidator();
			}
		}
		return validator;
	}


	@Bean
	public PathMatcher mvcPathMatcher() {
	    //路径匹配器，用于匹配请求路径的，在分析HandlerMapping时会提到
		if (getPathMatchConfigurer().getPathMatcher() != null) {
			return getPathMatchConfigurer().getPathMatcher();
		}
		else {
			return new AntPathMatcher();
		}
	}


	@Bean
	public UrlPathHelper mvcUrlPathHelper() {
	    //url助手类，用于获取url
		if (getPathMatchConfigurer().getUrlPathHelper() != null) {
			return getPathMatchConfigurer().getUrlPathHelper();
		}
		else {
			return new UrlPathHelper();
		}
	}

	@Bean
	public CompositeUriComponentsContributor mvcUriComponentsContributor() {
	    //一个请求信息封装类，封装url，ip，host等
		return new CompositeUriComponentsContributor(
				requestMappingHandlerAdapter().getArgumentResolvers(), mvcConversionService());
	}

    //一些handlerAdapter，后面会挑选一些讲解
	@Bean
	public HttpRequestHandlerAdapter httpRequestHandlerAdapter() {
		return new HttpRequestHandlerAdapter();
	}


	@Bean
	public SimpleControllerHandlerAdapter simpleControllerHandlerAdapter() {
		return new SimpleControllerHandlerAdapter();
	}

	//异常处理器
	@Bean
	public HandlerExceptionResolver handlerExceptionResolver() {
		List<HandlerExceptionResolver> exceptionResolvers = new ArrayList<HandlerExceptionResolver>();
		configureHandlerExceptionResolvers(exceptionResolvers);

		if (exceptionResolvers.isEmpty()) {
			addDefaultHandlerExceptionResolvers(exceptionResolvers);
		}

		HandlerExceptionResolverComposite composite = new HandlerExceptionResolverComposite();
		composite.setOrder(0);
		composite.setExceptionResolvers(exceptionResolvers);
		return composite;
	}

    //视图解析器
	@Bean
	public ViewResolver mvcViewResolver() {
		ViewResolverRegistry registry = new ViewResolverRegistry();
		registry.setContentNegotiationManager(mvcContentNegotiationManager());
		registry.setApplicationContext(this.applicationContext);
		configureViewResolvers(registry);

		if (registry.getViewResolvers().isEmpty()) {
			String[] names = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
					this.applicationContext, ViewResolver.class, true, false);
			if (names.length == 1) {
				registry.getViewResolvers().add(new InternalResourceViewResolver());
			}
		}

		ViewResolverComposite composite = new ViewResolverComposite();
		composite.setOrder(registry.getOrder());
		composite.setViewResolvers(registry.getViewResolvers());
		composite.setApplicationContext(this.applicationContext);
		composite.setServletContext(this.servletContext);
		return composite;
	}

    。。。。。。
```
上面构建很多mvc需要的bean，这些东西在我们后面分析会提到，也有些不会提到，但是可以自行去查阅。
我们建立一个SpringMVC应用的时候通常会在xml配置中配置一下标签

```
<mvc:annotation-driven></mvc:annotation-driven>
```

或者使用java代码配置

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
java代码配置的方式，我们在第一小节大致的分析了一下@EnableWebMvc，其实它的主要目的就是引入其他的@Configuration标注的java配置DelegatingWebMvcConfiguration，这个java配置，配置了RequestMappingHandlerMapping，RequestMappingHandlerAdapter，内容协商，web数据初始化绑定等，其实现在我们将要分析的annotation-driven的标签它的作用和这个DelegatingWebMvcConfiguration是一样的，只是换了中配法罢了。

mvc命名空间的处理器是MvcNamespaceHandler

其中处理annotation-driven标签的是AnnotationDrivenBeanDefinitionParser解析器，不多说，直接看到它的parse方法

```
public BeanDefinition parse(Element element, ParserContext parserContext) {
		Object source = parserContext.extractSource(element);
		XmlReaderContext readerContext = parserContext.getReaderContext();

		CompositeComponentDefinition compDefinition = new CompositeComponentDefinition(element.getTagName(), source);
		parserContext.pushContainingComponent(compDefinition);
        //解析annotation-driven上的属性content-negotiation-manager获取内容协商管理器，如果没有就构建默认的
		RuntimeBeanReference contentNegotiationManager = getContentNegotiationManager(element, source, parserContext);
        。。。。。。

		return null;
	}
```
解析内容协商

```
private RuntimeBeanReference getContentNegotiationManager(Element element, Object source,
			ParserContext parserContext) {

		RuntimeBeanReference beanRef;
		//查看annotation-driven标签上是否存在content-negotiation-manager属性
		if (element.hasAttribute("content-negotiation-manager")) {
			String name = element.getAttribute("content-negotiation-manager");
			//如果有，那么封装成RuntimeBeanReference，这种类型的表示它这个属性需要引用容器中对应beanName的bean
			//对应的解析器是BeanDefinitionValueResolver，我们在spring源码分析篇中分析过，这里不再多说
			beanRef = new RuntimeBeanReference(name);
		}
		else {
		    //如果用户没有指定，那么构建ContentNegotiationManagerFactoryBean的bean定义，这个ContentNegotiationManagerFactoryBean是实现了
		    //FactoryBean接口的类，所以最后获取的bean是FactoryBean#getOject的内容，也就是ContentNegotiationManager
			RootBeanDefinition factoryBeanDef = new RootBeanDefinition(ContentNegotiationManagerFactoryBean.class);
			factoryBeanDef.setSource(source);
			//bean的角色，ROLE_INFRASTRUCTURE基础设施，用于辅助其他功能的角色，对于这种，一般不提供给用户的
			//扩展的后置处理器处理
			factoryBeanDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
			//设置默认的媒体类型                                 //(*1*)
			factoryBeanDef.getPropertyValues().add("mediaTypes", getDefaultMediaTypes());

			String name = CONTENT_NEGOTIATION_MANAGER_BEAN_NAME;
			parserContext.getReaderContext().getRegistry().registerBeanDefinition(name , factoryBeanDef);
			parserContext.registerComponent(new BeanComponentDefinition(factoryBeanDef, name));
			beanRef = new RuntimeBeanReference(name);
		}
		return beanRef;
	}
	
	 //(*1*)
	 private Properties getDefaultMediaTypes() {
		Properties props = new Properties();
		//添加一些后缀，比如xml，那么对应的媒体类型为application/xml
		//这些判断条件都是在AnnotationDrivenBeanDefinitionParser解析器的静态块中通过检测是否存在对应格式的处理类来决定是否添加这个后缀
		//比如json，它会判断应用中是否存在com.fasterxml.jackson.databind.ObjectMapper and com.fasterxml.jackson.core.JsonGenerator
		// or com.google.gson.Gson
		if (romePresent) {
			props.put("atom", MediaType.APPLICATION_ATOM_XML_VALUE);
			props.put("rss", "application/rss+xml");
		}
		if (jaxb2Present || jackson2XmlPresent) {
			props.put("xml", MediaType.APPLICATION_XML_VALUE);
		}
		if (jackson2Present || gsonPresent) {
			props.put("json", MediaType.APPLICATION_JSON_VALUE);
		}
		return props;
	}

```

配置RequestMappingHandlerMapping

```
public BeanDefinition parse(Element element, ParserContext parserContext) {
    
    。。。。。。
    //构建RequestMappingHandlerMapping对象
    RootBeanDefinition handlerMappingDef = new RootBeanDefinition(RequestMappingHandlerMapping.class);
	handlerMappingDef.setSource(source);
	handlerMappingDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
	//设置排序
	handlerMappingDef.getPropertyValues().add("order", 0);
	//将上面获取到的内容协商管理器的引用设置到属性中
	handlerMappingDef.getPropertyValues().add("contentNegotiationManager", contentNegotiationManager);
    //是否开启矩阵
	if (element.hasAttribute("enable-matrix-variables")) {
		Boolean enableMatrixVariables = Boolean.valueOf(element.getAttribute("enable-matrix-variables"));
		//设置为不移除分号内容，也就是开启矩阵
		handlerMappingDef.getPropertyValues().add("removeSemicolonContent", !enableMatrixVariables);
	}
	else if (element.hasAttribute("enableMatrixVariables")) {
	    //这个应该是兼容老的属性
		Boolean enableMatrixVariables = Boolean.valueOf(element.getAttribute("enableMatrixVariables"));
		handlerMappingDef.getPropertyValues().add("removeSemicolonContent", !enableMatrixVariables);
	}
    。。。。。。
}
```
配置路径匹配器

```
public BeanDefinition parse(Element element, ParserContext parserContext) {
    。。。。。。
    //(*1*)
    configurePathMatchingProperties(handlerMappingDef, element, parserContext);
    readerContext.getRegistry().registerBeanDefinition(HANDLER_MAPPING_BEAN_NAME , handlerMappingDef);
    。。。。。。
}

//(*1*)
private void configurePathMatchingProperties(RootBeanDefinition handlerMappingDef, Element element,
			ParserContext parserContext) {
        //annotation-driven标签上的path-matching属性
		Element pathMatchingElement = DomUtils.getChildElementByTagName(element, "path-matching");
		if (pathMatchingElement != null) {
			Object source = parserContext.extractSource(element);
			if (pathMatchingElement.hasAttribute("suffix-pattern")) {
			    //设置是否需要进行后缀匹配，比如请求路径*.html,*.do
				Boolean useSuffixPatternMatch = Boolean.valueOf(pathMatchingElement.getAttribute("suffix-pattern"));
				handlerMappingDef.getPropertyValues().add("useSuffixPatternMatch", useSuffixPatternMatch);
			}
			if (pathMatchingElement.hasAttribute("trailing-slash")) {
			    //设置是否需要进行尾部斜杆匹配，比如请求路径/test/，匹配尾部的/，就会给用户设置@RequestMapping中pattern加上/
				Boolean useTrailingSlashMatch = Boolean.valueOf(pathMatchingElement.getAttribute("trailing-slash"));
				handlerMappingDef.getPropertyValues().add("useTrailingSlashMatch", useTrailingSlashMatch);
			}
			if (pathMatchingElement.hasAttribute("registered-suffixes-only")) {
			    //是否使用注册的后缀进行匹配
				Boolean useRegisteredSuffixPatternMatch = Boolean.valueOf(pathMatchingElement.getAttribute("registered-suffixes-only"));
				handlerMappingDef.getPropertyValues().add("useRegisteredSuffixPatternMatch", useRegisteredSuffixPatternMatch);
			}
			RuntimeBeanReference pathHelperRef = null;
			if (pathMatchingElement.hasAttribute("path-helper")) {
			    //设置url路径助手类
				pathHelperRef = new RuntimeBeanReference(pathMatchingElement.getAttribute("path-helper"));
			}
			//注册UrlPathHelper
			pathHelperRef = MvcNamespaceUtils.registerUrlPathHelper(pathHelperRef, parserContext, source);
			//设置UrlPathHelper
			handlerMappingDef.getPropertyValues().add("urlPathHelper", pathHelperRef);

			RuntimeBeanReference pathMatcherRef = null;
			if (pathMatchingElement.hasAttribute("path-matcher")) {
				pathMatcherRef = new RuntimeBeanReference(pathMatchingElement.getAttribute("path-matcher"));
			}
			//配置路径匹配器，如果没有匹配，默认是AntPathMatcher
			pathMatcherRef = MvcNamespaceUtils.registerPathMatcher(pathMatcherRef, parserContext, source);
			handlerMappingDef.getPropertyValues().add("pathMatcher", pathMatcherRef);
		}
	}

```


注册转化服务，校验器，消息编码解析器

```
public BeanDefinition parse(Element element, ParserContext parserContext) {
    。。。。。。
    //注册跨域配置
    RuntimeBeanReference corsConfigurationsRef = MvcNamespaceUtils.registerCorsConfigurations(null, parserContext, source);
	handlerMappingDef.getPropertyValues().add("corsConfigurations", corsConfigurationsRef);
    //获取转换服务，用于转化属性类型
    //(*1*)
	RuntimeBeanReference conversionService = getConversionService(element, source, parserContext);
	//获取校验器，JSR-303，默认工厂bean org.springframework.validation.beanvalidation.OptionalValidatorFactoryBean
	//一般使用实现了JSR-303的hibernate校验器
	RuntimeBeanReference validator = getValidator(element, source, parserContext);
	//获取消息编码解析器，用于错误编码和国际化信息的输出
	RuntimeBeanReference messageCodesResolver = getMessageCodesResolver(element);
    。。。。。。
}

//(*1*)
private RuntimeBeanReference getConversionService(Element element, Object source, ParserContext parserContext) {
		RuntimeBeanReference conversionServiceRef;
		//从标签属性conversion-service中获取引用
		if (element.hasAttribute("conversion-service")) {
			conversionServiceRef = new RuntimeBeanReference(element.getAttribute("conversion-service"));
		}
		else {
		    //构建默认的转换服务
			RootBeanDefinition conversionDef = new RootBeanDefinition(FormattingConversionServiceFactoryBean.class);
			conversionDef.setSource(source);
			conversionDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
			String conversionName = parserContext.getReaderContext().registerWithGeneratedName(conversionDef);
			parserContext.registerComponent(new BeanComponentDefinition(conversionDef, conversionName));
			conversionServiceRef = new RuntimeBeanReference(conversionName);
		}
		return conversionServiceRef;
	}

```
定义web绑定初始化器
```
public BeanDefinition parse(Element element, ParserContext parserContext) {
    。。。。。。
    //定义可配置的web数据绑定初始化器
    RootBeanDefinition bindingDef = new RootBeanDefinition(ConfigurableWebBindingInitializer.class);
	bindingDef.setSource(source);
	bindingDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
	//添加转换服务，在初始化数据绑定的时候设置给数据绑定
	bindingDef.getPropertyValues().add("conversionService", conversionService);
	//添加校验器
	bindingDef.getPropertyValues().add("validator", validator);
	//添加消息编码解析器
	bindingDef.getPropertyValues().add("messageCodesResolver", messageCodesResolver);
    。。。。。。
}
```
获取消息转换器，解析器，返回值处理器和异步处理拦截器

```
public BeanDefinition parse(Element element, ParserContext parserContext) {
    。。。。。。
    //获取消息转换器
    ManagedList<?> messageConverters = getMessageConverters(element, source, parserContext);
    //获取参数解析器
	ManagedList<?> argumentResolvers = getArgumentResolvers(element, parserContext);
	//获取返回值处理器
	ManagedList<?> returnValueHandlers = getReturnValueHandlers(element, parserContext);
	//获取异步超时时间
	String asyncTimeout = getAsyncTimeout(element);
	//获取异步线程池
	RuntimeBeanReference asyncExecutor = getAsyncExecutor(element);
	//获取回调拦截器
	ManagedList<?> callableInterceptors = getCallableInterceptors(element, source, parserContext);
	//获取延时结果拦截器
	ManagedList<?> deferredResultInterceptors = getDeferredResultInterceptors(element, source, parserContext);
    。。。。。。
}
```

RequestMappingHandlerAdapter

```
public BeanDefinition parse(Element element, ParserContext parserContext) {
    。。。。。。
    RootBeanDefinition handlerAdapterDef = new RootBeanDefinition(RequestMappingHandlerAdapter.class);
	handlerAdapterDef.setSource(source);
	handlerAdapterDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
	//设置内容协商管理器
	handlerAdapterDef.getPropertyValues().add("contentNegotiationManager", contentNegotiationManager);
	//设置web参数绑定初始化器
	handlerAdapterDef.getPropertyValues().add("webBindingInitializer", bindingDef);
	//设置消息转换器
	handlerAdapterDef.getPropertyValues().add("messageConverters", messageConverters);
	//添加一些默认的RequestBodyAdvice --> JsonViewRequestBodyAdvice
	addRequestBodyAdvice(handlerAdapterDef);
	//添加一些默认的ResponseBodyAdvice --> JsonViewRequestBodyAdvice
	addResponseBodyAdvice(handlerAdapterDef);
    
    //重定向时是否忽略默认的defaultModel
	if (element.hasAttribute("ignore-default-model-on-redirect")) {
		Boolean ignoreDefaultModel = Boolean.valueOf(element.getAttribute("ignore-default-model-on-redirect"));
		handlerAdapterDef.getPropertyValues().add("ignoreDefaultModelOnRedirect", ignoreDefaultModel);
	}
	//兼容性代码，spring的注释都已经告诉大家它是deprecated的
	else if (element.hasAttribute("ignoreDefaultModelOnRedirect")) {
		// "ignoreDefaultModelOnRedirect" spelling is deprecated
		Boolean ignoreDefaultModel = Boolean.valueOf(element.getAttribute("ignoreDefaultModelOnRedirect"));
		handlerAdapterDef.getPropertyValues().add("ignoreDefaultModelOnRedirect", ignoreDefaultModel);
	}

	if (argumentResolvers != null) {
	    //设置参数解析器
		handlerAdapterDef.getPropertyValues().add("customArgumentResolvers", argumentResolvers);
	}
	if (returnValueHandlers != null) {
	    //设置返回值处理器
		handlerAdapterDef.getPropertyValues().add("customReturnValueHandlers", returnValueHandlers);
	}
	if (asyncTimeout != null) {
	    //设置异步请求超时时间
		handlerAdapterDef.getPropertyValues().add("asyncRequestTimeout", asyncTimeout);
	}
	if (asyncExecutor != null) {
	    //设置任务的执行器
		handlerAdapterDef.getPropertyValues().add("taskExecutor", asyncExecutor);
	}
    //设置回调拦截器
	handlerAdapterDef.getPropertyValues().add("callableInterceptors", callableInterceptors);
	//设置延时结果拦截器
	handlerAdapterDef.getPropertyValues().add("deferredResultInterceptors", deferredResultInterceptors);
	//注册handlerAdapter的BeanDefinition
	readerContext.getRegistry().registerBeanDefinition(HANDLER_ADAPTER_BEAN_NAME , handlerAdapterDef);
    。。。。。。
}
```
注册uriCompContrib，用于辅助设置方法参数

```
public BeanDefinition parse(Element element, ParserContext parserContext) {
    。。。。。。
    String uriCompContribName = MvcUriComponentsBuilder.MVC_URI_COMPONENTS_CONTRIBUTOR_BEAN_NAME;
	RootBeanDefinition uriCompContribDef = new RootBeanDefinition(CompositeUriComponentsContributorFactoryBean.class);
	uriCompContribDef.setSource(source);
	uriCompContribDef.getPropertyValues().addPropertyValue("handlerAdapter", handlerAdapterDef);
	uriCompContribDef.getPropertyValues().addPropertyValue("conversionService", conversionService);
	readerContext.getRegistry().registerBeanDefinition(uriCompContribName, uriCompContribDef);
    。。。。。。
}
```
注册默认拦截器

```
public BeanDefinition parse(Element element, ParserContext parserContext) {
    。。。。。。
    //曝光ConversionService的拦截器
    RootBeanDefinition csInterceptorDef = new RootBeanDefinition(ConversionServiceExposingInterceptor.class);
    csInterceptorDef.setSource(source);
    //设置构造参数conversionService
    csInterceptorDef.getConstructorArgumentValues().addIndexedArgumentValue(0, conversionService);
    //映射拦截器
    RootBeanDefinition mappedCsInterceptorDef = new RootBeanDefinition(MappedInterceptor.class);
    mappedCsInterceptorDef.setSource(source);
    mappedCsInterceptorDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
    mappedCsInterceptorDef.getConstructorArgumentValues().addIndexedArgumentValue(0, (Object) null);
    //将ConversionServiceExposingInterceptor作为构造参数传入
    mappedCsInterceptorDef.getConstructorArgumentValues().addIndexedArgumentValue(1, csInterceptorDef);
    String mappedInterceptorName = readerContext.registerWithGeneratedName(mappedCsInterceptorDef);
    。。。。。。
}
```
异常处理器

```
public BeanDefinition parse(Element element, ParserContext parserContext) {
    。。。。。。
    //构造Controller层的@ExceptionHandler，异常处理解析器
    RootBeanDefinition exceptionHandlerExceptionResolver = new RootBeanDefinition(ExceptionHandlerExceptionResolver.class);
	exceptionHandlerExceptionResolver.setSource(source);
	exceptionHandlerExceptionResolver.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
	exceptionHandlerExceptionResolver.getPropertyValues().add("contentNegotiationManager", contentNegotiationManager);
	exceptionHandlerExceptionResolver.getPropertyValues().add("messageConverters", messageConverters);
	exceptionHandlerExceptionResolver.getPropertyValues().add("order", 0);
	addResponseBodyAdvice(exceptionHandlerExceptionResolver);

	String methodExceptionResolverName = readerContext.registerWithGeneratedName(exceptionHandlerExceptionResolver);

	RootBeanDefinition responseStatusExceptionResolver = new RootBeanDefinition(ResponseStatusExceptionResolver.class);
	responseStatusExceptionResolver.setSource(source);
	responseStatusExceptionResolver.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
	responseStatusExceptionResolver.getPropertyValues().add("order", 1);
	String responseStatusExceptionResolverName =
			readerContext.registerWithGeneratedName(responseStatusExceptionResolver);

	RootBeanDefinition defaultExceptionResolver = new RootBeanDefinition(DefaultHandlerExceptionResolver.class);
	defaultExceptionResolver.setSource(source);
	defaultExceptionResolver.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
	defaultExceptionResolver.getPropertyValues().add("order", 2);
	String defaultExceptionResolverName =
				readerContext.registerWithGeneratedName(defaultExceptionResolver);
    。。。。。。
}
```

注册基础的HandlerMapping，HandlerAdapter

```
public BeanDefinition parse(Element element, ParserContext parserContext) {
    。。。。。。
    // Ensure BeanNameUrlHandlerMapping (SPR-8289) and default HandlerAdapters are not "turned off"
    //注册BeanNameUrlHandlerMapping，HttpRequestHandlerAdapter，SimpleControllerHandlerAdapter
	MvcNamespaceUtils.registerDefaultComponents(parserContext, source);
    。。。。。。
}
```

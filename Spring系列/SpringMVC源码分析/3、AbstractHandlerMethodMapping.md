
AbstractHandlerMethodMapping是处理HandlerMethod的HandlerMapping，其中有一个实现了是我们常用的
它就是RequestMappingHandlerMapping

我们直接看到RequestMappingHandlerMapping的类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/56e7b2c48aee7a1af63e49303fc6763f.png)
这里大部分接口我们在spring源码分析的时候已经解释过了，这里解释一下那些没有分析过的接口的作用

- EmbeddedValueResolverAware 在bean进行初始化时设置StringValueResolver，这个StringValueResolver是一个占位符解析器
- ApplicationObjectSupport 实现了ApplicationContextAware，额外提供了获取MessageSource（用于国际化）的方法
- HandlerMapping 获取handler和能够用于拦截找到的handler的拦截器，然后组成HandlerExecutionChain

几个实现类的分析

- AbstractHandlerMapping

```
public abstract class AbstractHandlerMapping extends WebApplicationObjectSupport
		implements HandlerMapping, Ordered {

	private int order = Integer.MAX_VALUE;  // default: same as non-Ordered

	private Object defaultHandler;
    //构建urlPathHelper
	private UrlPathHelper urlPathHelper = new UrlPathHelper();
    //AntPathMatcher，这个类我们在分析spring源码的时候进行分析过
	private PathMatcher pathMatcher = new AntPathMatcher();

	private final List<Object> interceptors = new ArrayList<Object>();

	private final List<HandlerInterceptor> adaptedInterceptors = new ArrayList<HandlerInterceptor>();
    //默认跨域处理器
	private CorsProcessor corsProcessor = new DefaultCorsProcessor();
    //基于url的跨域配置源
	private final UrlBasedCorsConfigurationSource corsConfigSource = new UrlBasedCorsConfigurationSource();
	
	。。。。。。
}
```


```
public final HandlerExecutionChain getHandler(PortletRequest request) throws Exception {
        //抽象方法，有子类实现
		Object handler = getHandlerInternal(request);
		//如果没有获取到，尝试使用默认的handler，一般不设置
		if (handler == null) {
			handler = getDefaultHandler();
		}
		if (handler == null) {
			return null;
		}
		// Bean name or resolved handler?
		//如果是字符串类型，那么从容器中获取
		if (handler instanceof String) {
			String handlerName = (String) handler;
			handler = getApplicationContext().getBean(handlerName);
		}
		//将获取到的handler和拦截器构建成HandlerExecutionChain
		return getHandlerExecutionChain(handler, request);
	}
```
- AbstractHandlerMethodMapping 这个抽象类实现了InitializingBean接口，让我们来看看它的afterPropertiesSet做了什么

```
//这个方法被子类RequestMappingHandlerMapping覆盖
public void org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping.afterPropertiesSet() {
        //构建配置
		this.config = new RequestMappingInfo.BuilderConfiguration();
		this.config.setPathHelper(getUrlPathHelper());
		this.config.setPathMatcher(getPathMatcher());
		this.config.setSuffixPatternMatch(this.useSuffixPatternMatch);
		this.config.setTrailingSlashMatch(this.useTrailingSlashMatch);
		this.config.setRegisteredSuffixPatternMatch(this.useRegisteredSuffixPatternMatch);
		this.config.setContentNegotiationManager(getContentNegotiationManager());
        
		super.afterPropertiesSet();
	}
	
	public void AbstractHandlerMethodMapping.afterPropertiesSet() {
		initHandlerMethods();
	}

//初始化HandlerMethod
protected void initHandlerMethods() {
		if (logger.isDebugEnabled()) {
			logger.debug("Looking for request mappings in application context: " + getApplicationContext());
		}
		//从容器中获取所有beanName，detectHandlerMethodsInAncestorContexts表示是否需要获取父容器的beanName
		//默认是false，所以一般我们配置的Controller都是在SpringMVC加载配置文件中
		String[] beanNames = (this.detectHandlerMethodsInAncestorContexts ?
				BeanFactoryUtils.beanNamesForTypeIncludingAncestors(getApplicationContext(), Object.class) :
				getApplicationContext().getBeanNamesForType(Object.class));

		for (String beanName : beanNames) {
		    //是否以scopedTarget.为前缀，这个scopedTarget.表示范围对象代理，比如request，session等这些scope的对象
		    //如果我们要把短生命周期的注册到长生命周期的对象中，就应当使用代理，否则就变成和长生命周期的对象一样长的寿命
		    //这里不考虑这类被代理过的Controller
			if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
				Class<?> beanType = null;
				try {
				    //获取beanName对应bean的类型
					beanType = getApplicationContext().getType(beanName);
				}
				catch (Throwable ex) {
					// An unresolvable bean type, probably from a lazy bean - let's ignore it.
					if (logger.isDebugEnabled()) {
						logger.debug("Could not resolve target class for bean with name '" + beanName + "'", ex);
					}
				}
				//判断当前类是否是需要的handler，isHandler方法被子类RequestMappingHandlerMapping实现
				//其判断逻辑就一行代码
				//((AnnotationUtils.findAnnotation(beanType, Controller.class) != null) ||
				//(AnnotationUtils.findAnnotation(beanType, RequestMapping.class) != null))
				//类是否被@Controller和@RequestMapping两个注解标注
				if (beanType != null && isHandler(beanType)) {
				    //装配HanderMethod
					detectHandlerMethods(beanName);
				}
			}
		}
		//获取到handlerMethod的后置处理方法，为空方法。
		handlerMethodsInitialized(getHandlerMethods());
	}

```

装配HanderMethod

```
protected void detectHandlerMethods(final Object handler) {
        //获取对应handler的类型
		Class<?> handlerType = (handler instanceof String ?
				getApplicationContext().getType((String) handler) : handler.getClass());
		final Class<?> userType = ClassUtils.getUserClass(handlerType);
        //筛选@RequestMapping注解注释的方法
		Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
				new MethodIntrospector.MetadataLookup<T>() {
					@Override
					public T inspect(Method method) {
					    //获取@RequestMapp注解对应的映射信息，由子类实现
					    //RequestMappingHandlerMapping返回的是RequestMappingInfo
						return getMappingForMethod(method, userType);
					}
				});

		if (logger.isDebugEnabled()) {
			logger.debug(methods.size() + " request handler methods found on " + userType + ": " + methods);
		}
		for (Map.Entry<Method, T> entry : methods.entrySet()) {
		    //注册HandlerMethod
			registerHandlerMethod(handler, entry.getKey(), entry.getValue());
		}
	}
```
- RequestMappingHandlerMapping

筛选HandlerMethod

```
public static <T> Map<Method, T> org.springframework.core.MethodIntrospector.selectMethods(Class<?> targetType, final MetadataLookup<T> metadataLookup) {
		final Map<Method, T> methodMap = new LinkedHashMap<Method, T>();
		Set<Class<?>> handlerTypes = new LinkedHashSet<Class<?>>();
		Class<?> specificHandlerType = null;
        //判断当前类是否被代理过，如果没有被代理，那么当前类也应当考虑进去
		if (!Proxy.isProxyClass(targetType)) {
			handlerTypes.add(targetType);
			specificHandlerType = targetType;
		}
		//添加接口
		handlerTypes.addAll(Arrays.asList(targetType.getInterfaces()));
        
		for (Class<?> currentHandlerType : handlerTypes) {
		    //设置目标类，用于获取最具体的方法（比如最终的那个覆盖方法）
			final Class<?> targetClass = (specificHandlerType != null ? specificHandlerType : currentHandlerType);

			ReflectionUtils.doWithMethods(currentHandlerType, new ReflectionUtils.MethodCallback() {
				@Override
				public void doWith(Method method) {
				    //从targetClass获取最具体的方法，比如接口的方法可能被targetClass覆盖
					Method specificMethod = ClassUtils.getMostSpecificMethod(method, targetClass);
					
					
					//(*1*)
					T result = metadataLookup.inspect(specificMethod);
					if (result != null) {
					    //获取被桥的方法（被桥的方法才是用户真正的定义的方法，桥方法的解释在spring源码分析的时候已经解释过了）
						Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);
						//如果被桥方法就是当前具体的方法或者这个被桥的方法就是HandlerMethod方法，那么保存数据
						if (bridgedMethod == specificMethod || metadataLookup.inspect(bridgedMethod) == null) {
							methodMap.put(specificMethod, result);
						}
					}
				}
				//一个类过滤器，过滤掉桥方法和Object类定义的方法
			}, ReflectionUtils.USER_DECLARED_METHODS);
		}

		return methodMap;
	}
	
		//(*1*)
	protected RequestMappingInfo org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping.getMappingForMethod(Method method, Class<?> handlerType) {
	    //创建方法请求映射信息
		RequestMappingInfo info = createRequestMappingInfo(method);
		if (info != null) {
		    //如果类上面也有@RequestMapping注解，那么需要创建类请求映射信息
			RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType);
			if (typeInfo != null) {
			    //进行合并
				info = typeInfo.combine(info);
			}
		}
		//返回最终映射信息
		return info;
	}
```

创建RequestMappingInfo

```
private RequestMappingInfo org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping.createRequestMappingInfo(AnnotatedElement element) {
        //获取RequestMapping注解，注意这里获取的RequestMapping注解类型是一个代理对象，主要是因为spring会对元数据进行处理，
        //元数据被当成注解的父类，他们的属性会被继承，覆盖，AnnotatedElementUtils在spring源码分析中分析过
		RequestMapping requestMapping = AnnotatedElementUtils.findMergedAnnotation(element, RequestMapping注解类型可能.class);
		//获取自定义的请求条件，我们可以继承这个RequestMappingHandlerMapping，然后实现这个方法并注册到spring容器中
		RequestCondition<?> condition = (element instanceof Class<?> ?
				getCustomTypeCondition((Class<?>) element) : getCustomMethodCondition((Method) element));
		//创建RequestMappingInfo
		//(*1*)
		return (requestMapping != null ? createRequestMappingInfo(requestMapping, condition) : null);
	}
	
//(*1*)
protected RequestMappingInfo createRequestMappingInfo(
			RequestMapping requestMapping, RequestCondition<?> customCondition) {

		return RequestMappingInfo
		        //设置路径信息，路径信息如果有${}占位符，会进行替换
				.paths(resolveEmbeddedValuesInPatterns(requestMapping.path()))
				//设置请求方法
				.methods(requestMapping.method())
				//请求参数，比如a=10
				.params(requestMapping.params())
				//设置头部信息
				.headers(requestMapping.headers())
				//指定能够处理此媒体类型的处理器去处理请求
				.consumes(requestMapping.consumes())
				//指定请求返回时能够接受的媒体类型
				.produces(requestMapping.produces())
				//映射名
				.mappingName(requestMapping.name())
				//设置自定义的请求条件
				.customCondition(customCondition)
				//配置对象设置
				.options(this.config)
				//返回RequestMappingInfo对象
				.build();
	}

```
在上面的代码中给RequestMappingInfo设置一个配置，下面就是这个配置的类定义

```
public static class org.springframework.web.servlet.mvc.method.RequestMappingInfo.BuilderConfiguration {
        //用户获取url路径的助手类
		private UrlPathHelper urlPathHelper;
        //路径匹配器
		private PathMatcher pathMatcher;
        //是否应用尾部的/匹配
		private boolean trailingSlashMatch = true;
        //是否匹配后缀
		private boolean suffixPatternMatch = true;
        //注册后缀
		private boolean registeredSuffixPatternMatch = false;
        //内容协商管理器，用于解析媒体类型，后缀等
		private ContentNegotiationManager contentNegotiationManager;
		
    。。。。。。		
}
```

build

```
public RequestMappingInfo build() {
            //从配置操作中获取内容协商管理器
			ContentNegotiationManager manager = this.options.getContentNegotiationManager();
            //创建模式请求条件，虽然可以是通配符的形式，但是我们在实际的SpringMVC的项目中，似乎很少使用
            //有通配符的路径，基本上是很具体的路径
			PatternsRequestCondition patternsCondition = new PatternsRequestCondition(
					this.paths, this.options.getUrlPathHelper(), this.options.getPathMatcher(),
					this.options.useSuffixPatternMatch(), this.options.useTrailingSlashMatch(),
					this.options.getFileExtensions());
            //创建RequestMappingInfo
			return new RequestMappingInfo(this.mappingName, patternsCondition,
					new RequestMethodsRequestCondition(methods),
					new ParamsRequestCondition(this.params),
					new HeadersRequestCondition(this.headers),
					new ConsumesRequestCondition(this.consumes, this.headers),
					new ProducesRequestCondition(this.produces, this.headers, manager),
					this.customCondition);
		}
```
下面是RequestCondition的类层级结构

```
RequestCondition (org.springframework.web.servlet.mvc.condition)
    RequestMappingInfo (org.springframework.web.servlet.mvc.method)
    AbstractRequestCondition (org.springframework.web.servlet.mvc.condition)
        RequestMethodsRequestCondition (org.springframework.web.servlet.mvc.condition)
        ProducesRequestCondition (org.springframework.web.servlet.mvc.condition)
        PatternsRequestCondition (org.springframework.web.servlet.mvc.condition)
        ParamsRequestCondition (org.springframework.web.servlet.mvc.condition)
        RequestConditionHolder (org.springframework.web.servlet.mvc.condition)
        ConsumesRequestCondition (org.springframework.web.servlet.mvc.condition)
        HeadersRequestCondition (org.springframework.web.servlet.mvc.condition)
        CompositeRequestCondition (org.springframework.web.servlet.mvc.condition)

```
我们随便选其中一个RequestCondition来大致分析下，比如ConsumesRequestCondition，我们直接看到它实现的getMatchingCondition方法

```
public ConsumesRequestCondition getMatchingCondition(HttpServletRequest request) {
		if (isEmpty()) {
			return this;
		}
		//消费媒体类型表达式，用于进行匹配
		Set<ConsumeMediaTypeExpression> result = new LinkedHashSet<ConsumeMediaTypeExpression>(expressions);
		for (Iterator<ConsumeMediaTypeExpression> iterator = result.iterator(); iterator.hasNext();) {
			ConsumeMediaTypeExpression expression = iterator.next();
			if (!expression.match(request)) {
			    //把不匹配的去掉
				iterator.remove();
			}
		}
		//返回能够匹配当前request请求的消费请求条件，主要用于后面将要进行最佳匹配RequestMappingInfo的筛选
		return (result.isEmpty()) ? null : new ConsumesRequestCondition(result);
	}

```
ConsumeMediaTypeExpression的匹配

```
public final boolean match(HttpServletRequest request) {
		try {
		    //(*1*)
			boolean match = matchMediaType(request);
			return (!this.isNegated ? match : !match);
		}
		catch (HttpMediaTypeException ex) {
			return false;
		}
	}
	
	protected boolean matchMediaType(HttpServletRequest request) throws HttpMediaTypeNotSupportedException {
			try {
			    //解析request中告知的媒体类型
				MediaType contentType = StringUtils.hasLength(request.getContentType()) ?
						MediaType.parseMediaType(request.getContentType()) :
						MediaType.APPLICATION_OCTET_STREAM;
						//将程序中设置的媒体类型与请求的媒体类型进行对比
						return getMediaType().includes(contentType);
			}
			catch (InvalidMediaTypeException ex) {
				throw new HttpMediaTypeNotSupportedException(
						"Can't parse Content-Type [" + request.getContentType() + "]: " + ex.getMessage());
			}
		}
	}
```
解析request请求中的媒体类型

```
public static MimeType org.springframework.util.MimeTypeUtils.parseMimeType(String mimeType) {
		if (!StringUtils.hasLength(mimeType)) {
			throw new InvalidMimeTypeException(mimeType, "'mimeType' must not be empty");
		}
		//分号分割，比如text/html;charset=utf-8
		String[] parts = StringUtils.tokenizeToStringArray(mimeType, ";");
        //获取完整的类型
		String fullType = parts[0].trim();
		// java.net.HttpURLConnection returns a *; q=.2 Accept header
		//如果是一个*号，那么设置fulltype为*/*
		if (MimeType.WILDCARD_TYPE.equals(fullType)) {
			fullType = "*/*";
		}
		//获取子类型的分割符下标
		int subIndex = fullType.indexOf('/');
		//无效的媒体类型
		if (subIndex == -1) {
			throw new InvalidMimeTypeException(mimeType, "does not contain '/'");
		}
		//媒体类型必须是主类型与子类型进行/分割的
		if (subIndex == fullType.length() - 1) {
			throw new InvalidMimeTypeException(mimeType, "does not contain subtype after '/'");
		}
		//获取主类型
		String type = fullType.substring(0, subIndex);
		//获取子类型
		String subtype = fullType.substring(subIndex + 1, fullType.length());
		//错误的媒体类型，不能*/html
		if (MimeType.WILDCARD_TYPE.equals(type) && !MimeType.WILDCARD_TYPE.equals(subtype)) {
			throw new InvalidMimeTypeException(mimeType, "wildcard type is legal only in '*/*' (all mime types)");
		}
        //解析参数
		Map<String, String> parameters = null;
		if (parts.length > 1) {
			parameters = new LinkedHashMap<String, String>(parts.length - 1);
			for (int i = 1; i < parts.length; i++) {
			    //比如charset=utf-8
				String parameter = parts[i];
				int eqIndex = parameter.indexOf('=');
				if (eqIndex != -1) {
					String attribute = parameter.substring(0, eqIndex);
					String value = parameter.substring(eqIndex + 1, parameter.length());
					//比如 charset utf-8
					parameters.put(attribute, value);
				}
			}
		}

		try {
		    //创建MimeType
			return new MimeType(type, subtype, parameters);
		}
		catch (UnsupportedCharsetException ex) {
			throw new InvalidMimeTypeException(mimeType, "unsupported charset '" + ex.getCharsetName() + "'");
		}
		catch (IllegalArgumentException ex) {
			throw new InvalidMimeTypeException(mimeType, ex.getMessage());
		}
	}
```

匹配媒体类型

```
public boolean org.springframework.util.MimeType.includes(MimeType other) {
		if (other == null) {
			return false;
		}
		//如果是*/*，表示匹配所有
		if (this.isWildcardType()) {
			// */* includes anything
			return true;
		}
		else if (getType().equals(other.getType())) {
		    //主和子类型都匹配，那么直接返回true
			if (getSubtype().equals(other.getSubtype())) {
				return true;
			}
			//是否存在子类型通配符
			if (this.isWildcardSubtype()) {
				// wildcard with suffix, e.g. application/*+xml
				int thisPlusIdx = getSubtype().indexOf('+');
				//如果没有+号，也就是 application/*，那么返回true
				if (thisPlusIdx == -1) {
					return true;
				}
				else {
					// application/*+xml includes application/soap+xml
					int otherPlusIdx = other.getSubtype().indexOf('+');
					if (otherPlusIdx != -1) {
						String thisSubtypeNoSuffix = getSubtype().substring(0, thisPlusIdx);
						String thisSubtypeSuffix = getSubtype().substring(thisPlusIdx + 1);
						String otherSubtypeSuffix = other.getSubtype().substring(otherPlusIdx + 1);
						//如果+号后面的字符串匹配并且+号前面确实是*号，那么表示匹配
						if (thisSubtypeSuffix.equals(otherSubtypeSuffix) && WILDCARD_TYPE.equals(thisSubtypeNoSuffix)) {
							return true;
						}
					}
				}
			}
		}
		return false;
	}
```

好了，回到获取handlerMethod的逻辑，当 当前类，父类，接口的方法都进行一番检查之后，形成了一个Map<Method, RequestMappingInfo>，接下来就是注册了

```
protected void org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.registerHandlerMethod(Object handler, Method method, T mapping) {
        //注册映射
		this.mappingRegistry.register(mapping, handler, method);
	}
```
注册映射

```
public void org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.MappingRegistry.register(T mapping, Object handler, Method method) {
			this.readWriteLock.writeLock().lock();
			try {
			    //创建HandlerMethod，一个维护handler，RequestMappingInfo，method，beanFactory，bridgedMethod，parameters
				HandlerMethod handlerMethod = createHandlerMethod(handler, method);
				//检查是否是唯一的方法映射，RequestMappingInfo -》 handlerMethod
				assertUniqueMethodMapping(handlerMethod, mapping);

				if (logger.isInfoEnabled()) {
					logger.info("Mapped \"" + mapping + "\" onto " + handlerMethod);
				}
				//一个map《RequestMappingInfo -》 handlerMethod》
				this.mappingLookup.put(mapping, handlerMethod);
                
                //获取直接路径，也就是具体路径，没有*号，？号这类通配符的路径
				List<String> directUrls = getDirectUrls(mapping);
				for (String url : directUrls) {
				    //map《直接路径，List<RequestMappingInfo>》
					this.urlLookup.add(url, mapping);
				}

				String name = null;
				//获取命名策略，这个对象是在创建RequestMappingHandlerMapping时，其父类RequestMappingInfoHandlerMapping的构造器中设置的
				//它是RequestMappingInfoHandlerMethodMappingNamingStrategy的实例，当然如果在@RequestMapping注解中
				//设置了name就不会再次设置，否则将通过类名的第一个大写字符#方法名组成映射名字
				if (getNamingStrategy() != null) {
					name = getNamingStrategy().getName(handlerMethod, mapping);
					
					
					//Map<String, List<HandlerMethod>> nameLookup
					addMappingName(name, handlerMethod);
				}
                //初始化跨域配置
				CorsConfiguration corsConfig = initCorsConfiguration(handler, method, mapping);
				if (corsConfig != null) {
				    //map《handlerMethod， 跨域配置》
					this.corsLookup.put(handlerMethod, corsConfig);
				}
                //map《mapping, MappingRegistration》
				this.registry.put(mapping, new MappingRegistration<T>(mapping, handlerMethod, directUrls, name));
			}
			finally {
				this.readWriteLock.writeLock().unlock();
			}
		}
```
跨域配置

```
protected CorsConfiguration org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping.initCorsConfiguration(Object handler, Method method, RequestMappingInfo mappingInfo) {
        //创建HandlerMethod，和前面的创建方式一样
		HandlerMethod handlerMethod = createHandlerMethod(handler, method);
		//获取类上的@CrossOrigin信息
		CrossOrigin typeAnnotation = AnnotatedElementUtils.findMergedAnnotation(handlerMethod.getBeanType(), CrossOrigin.class);
		//获取方法上的@CrossOrigin信息
		CrossOrigin methodAnnotation = AnnotatedElementUtils.findMergedAnnotation(method, CrossOrigin.class);

		if (typeAnnotation == null && methodAnnotation == null) {
			return null;
		}
        //跨域配置，比如设置允许的来源url，方法，请求头，有效时间，凭证
		CorsConfiguration config = new CorsConfiguration();
		//设置类上的跨域配置
		updateCorsConfig(config, typeAnnotation);
		//设置方法上的跨域配置（更新操作）
		updateCorsConfig(config, methodAnnotation);
        //如果没有设置，那么设置默认的，默认的为*，表示允许所有
		if (CollectionUtils.isEmpty(config.getAllowedOrigins())) {
			config.setAllowedOrigins(Arrays.asList(CrossOrigin.DEFAULT_ORIGINS));
		}
		if (CollectionUtils.isEmpty(config.getAllowedMethods())) {
			for (RequestMethod allowedMethod : mappingInfo.getMethodsCondition().getMethods()) {
				config.addAllowedMethod(allowedMethod.name());
			}
		}
		if (CollectionUtils.isEmpty(config.getAllowedHeaders())) {
			config.setAllowedHeaders(Arrays.asList(CrossOrigin.DEFAULT_ALLOWED_HEADERS));
		}
		if (config.getAllowCredentials() == null) {
			config.setAllowCredentials(CrossOrigin.DEFAULT_ALLOW_CREDENTIALS);
		}
		if (config.getMaxAge() == null) {
			config.setMaxAge(CrossOrigin.DEFAULT_MAX_AGE);
		}
		return config;
	}
```

总结一下：RequestMappingHandlerMapping在初始化的时候会获取容器中所有的beanName，然后找到所有被标注了@Controller或者@RequestMapping注解的bean，然后筛选HandlerMethod，也就是被@RequestMapping标注的方法，创建RequestMappingInfo，最后进行注册。

这里有个问题，那就是在bean进行初始化的时候获取所有的bean，哎，这里会发生controller bean的遗漏吗？不会的，我们在springmvc源码分析的前几节的时候可以看到SpringMVC是在所有容器启动之后在onRefresh中进行的，这也解答了，SpringMVC为什么要通过ApplicationContext.getBean(BeanDefinition)进行bean的创建了，而不是直接new。

好了，一切都已经准备就绪，接下来就是寻找handlerMethod了

```
protected HandlerMethod org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.getHandlerInternal(HttpServletRequest request) throws Exception {
        //获取请求路径
		String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
		if (logger.isDebugEnabled()) {
			logger.debug("Looking up handler method for path " + lookupPath);
		}
		this.mappingRegistry.acquireReadLock();
		try {
		    //寻找HandlerMethod
			HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
			if (logger.isDebugEnabled()) {
				if (handlerMethod != null) {
					logger.debug("Returning handler method [" + handlerMethod + "]");
				}
				else {
					logger.debug("Did not find handler method for [" + lookupPath + "]");
				}
			}
			//返回handlerMethod，创建一个新的handlerMethod，防止被修改，如果内部持有的handler是字符串类型，那么会从
			//容器中寻找对应
			return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
		}
		finally {
			this.mappingRegistry.releaseReadLock();
		}
	}
```
lookupHandlerMethod

```
protected HandlerMethod org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
		List<Match> matches = new ArrayList<Match>();
		//首先通过精确匹配的方式获取   从urlLookup中获取
		List<T> directPathMatches = this.mappingRegistry.getMappingsByUrl(lookupPath);
		if (directPathMatches != null) {
		    //向matches添加能够匹配到的Match对象，Match对象维护着RequestMappingInfo和对应的HandlerMethod
			addMatchingMappings(directPathMatches, matches, request);
		}
		//如果没有获取到，那么进行全对比
		if (matches.isEmpty()) {
			// No choice but to go through all mappings...
			//向matches添加能够匹配到的Match对象，Match对象维护着RequestMappingInfo和对应的HandlerMethod
			addMatchingMappings(this.mappingRegistry.getMappings().keySet(), matches, request);
		}
        //如果找到了，那么还需要选出最适合的
		if (!matches.isEmpty()) {
		    //获取映射比较器
			Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));
			//进行排序
			Collections.sort(matches, comparator);
			if (logger.isTraceEnabled()) {
				logger.trace("Found " + matches.size() + " matching mapping(s) for [" +
						lookupPath + "] : " + matches);
			}
			//第一个被认为是最佳匹配
			Match bestMatch = matches.get(0);
			//如果匹配到的个数大于1
			if (matches.size() > 1) {
			    //检查是否为跨域请求，如果是并且是options请求类型，不允许跨域，返回EmptHandler
				if (CorsUtils.isPreFlightRequest(request)) {
					return PREFLIGHT_AMBIGUOUS_MATCH;
				}
				Match secondBestMatch = matches.get(1);
				//如果第一个和第二个在比较中是相等的，那么出现歧义，抛错
				if (comparator.compare(bestMatch, secondBestMatch) == 0) {
					Method m1 = bestMatch.handlerMethod.getMethod();
					Method m2 = secondBestMatch.handlerMethod.getMethod();
					throw new IllegalStateException("Ambiguous handler methods mapped for HTTP path '" +
							request.getRequestURL() + "': {" + m1 + ", " + m2 + "}");
				}
			}
			handleMatch(bestMatch.mapping, lookupPath, request);
			//获取handlerMethod
			return bestMatch.handlerMethod;
		}
		else {
		    //木有找到
			return handleNoMatch(this.mappingRegistry.getMappings().keySet(), lookupPath, request);
		}
	}
```

addMatchingMappings

```
private void org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.addMatchingMappings(Collection<T> mappings, List<Match> matches, HttpServletRequest request) {
        //循环RequestMappingInfo集合
		for (T mapping : mappings) {
		    //获取匹配的RequestMappingInfo
		    //(*1*)
			T match = getMatchingMapping(mapping, request);
			if (match != null) {
			    //包装成Match，添加到matches集合中
				matches.add(new Match(match, this.mappingRegistry.getMappings().get(mapping)));
			}
		}
	}
	
	//(*1*)
	protected RequestMappingInfo org.springframework.web.servlet.mvc.method.RequestMappingInfoHandlerMapping.getMatchingMapping(RequestMappingInfo info, HttpServletRequest request) {
	    //(*2*)
		return info.getMatchingCondition(request);
	}
	
	//(*2*)
	public RequestMappingInfo org.springframework.web.servlet.mvc.method.RequestMappingInfo.getMatchingCondition(HttpServletRequest request) {
	
	    //匹配请求方法，和我们在上面分析的ConsumesRequestCondition差不多，就是把匹配的方法重新包装返回回来
	    //，如果注解中没有配置此项，返回其本身（一个空的condition）
		RequestMethodsRequestCondition methods = this.methodsCondition.getMatchingCondition(request);
		//匹配请求参数，把能匹配到的重新构建ParamsRequestCondition返回，如果注解中没有配置此项，返回其本身（一个空的condition）
		ParamsRequestCondition params = this.paramsCondition.getMatchingCondition(request);
		//匹配请求头，把能匹配的重新构建HeadersRequestCondition返回，如果注解中没有配置此项，返回其本身（一个空的condition）
		HeadersRequestCondition headers = this.headersCondition.getMatchingCondition(request);
		//匹配请求媒体类型，把能匹配到的重新构建ConsumesRequestCondition返回，如果注解中没有配置此项，返回其本身（一个空的condition）
		ConsumesRequestCondition consumes = this.consumesCondition.getMatchingCondition(request);
		//匹配返回媒体类型，把能够匹配到的重新构建ProducesRequestCondition返回，如果注解中没有配置此项，返回其本身（一个空的condition）
		ProducesRequestCondition produces = this.producesCondition.getMatchingCondition(request);
        
        //如果有一个不匹配，看是不是跨域，如果没有跨域直接返回null，表示没有找到
		if (methods == null || params == null || headers == null || consumes == null || produces == null) {
			if (CorsUtils.isPreFlightRequest(request)) {
			    //匹配跨域请求方法，并重新构建RequestMethodsRequestCondition返回
				methods = getAccessControlRequestMethodCondition(request);
				if (methods == null || params == null) {
					return null;
				}
			}
			else {
				return null;
			}
		}
        
        //通配符匹配
		PatternsRequestCondition patterns = this.patternsCondition.getMatchingCondition(request);
		if (patterns == null) {
			return null;
		}
        //自定义请求条件匹配
		RequestConditionHolder custom = this.customConditionHolder.getMatchingCondition(request);
		if (custom == null) {
			return null;
		}
        //如果匹配，那么把与当前请求最精确的匹配重新构建成RequestMappingInfo返回
		return new RequestMappingInfo(this.name, patterns,
				methods, params, headers, consumes, produces, custom.getCondition());
	}
	
```
这里我们再看下PatternsRequestCondition是如何匹配的

```
public PatternsRequestCondition getMatchingCondition(HttpServletRequest request) {
        //如果没有设置路径匹配，那么直接返回自身
		if (this.patterns.isEmpty()) {
			return this;
		}
        //获取请求路径
		String lookupPath = this.pathHelper.getLookupPathForRequest(request);
		//获取匹配的路径
		//(*1*)
		List<String> matches = getMatchingPatterns(lookupPath);
        //如果不为空，那么重新构建一个与当前request最匹配的PatternsRequestCondition
		return matches.isEmpty() ? null :
			new PatternsRequestCondition(matches, this.pathHelper, this.pathMatcher, this.useSuffixPatternMatch,
					this.useTrailingSlashMatch, this.fileExtensions);
	}
	
	//(*1*)
	public List<String> getMatchingPatterns(String lookupPath) {
		List<String> matches = new ArrayList<String>();
		for (String pattern : this.patterns) {
		    //获取匹配的路径，可能会做些处理，比如添加后缀什么的
		    //(*2*)
			String match = getMatchingPattern(pattern, lookupPath);
			if (match != null) {
				matches.add(match);
			}
		}
		//进行排序，使用的排序器是AntPatternComparator
		Collections.sort(matches, this.pathMatcher.getPatternComparator(lookupPath));
		return matches;
	}
	
	//(*2*)
	private String getMatchingPattern(String pattern, String lookupPath) {
	    //如果直接相等，那么直接返回，这个路径已经超级具体了
		if (pattern.equals(lookupPath)) {
			return pattern;
		}
		//是否允许后缀匹配
		if (this.useSuffixPatternMatch) {
		    //如果有备用后缀和请求路径就有后缀
			if (!this.fileExtensions.isEmpty() && lookupPath.indexOf('.') != -1) {
				for (String extension : this.fileExtensions) {
				    //匹配
					if (this.pathMatcher.match(pattern + extension, lookupPath)) {
					    //返回添加了后缀的路径，比如/test/*/abc + .html
						return pattern + extension;
					}
				}
			}
			else {
			    //如果@RequestMapping注解定义的路径没有后缀
				boolean hasSuffix = pattern.indexOf('.') != -1;
				//那么加上.*进行匹配
				if (!hasSuffix && this.pathMatcher.match(pattern + ".*", lookupPath)) {
					return pattern + ".*";
				}
			}
		}
		//直接路径匹配
		if (this.pathMatcher.match(pattern, lookupPath)) {
			return pattern;
		}
		//是否使用后缀/匹配，如果@RequestMapping注解定义的路径不是/结尾，那么加上后再匹配
		if (this.useTrailingSlashMatch) {
			if (!pattern.endsWith("/") && this.pathMatcher.match(pattern + "/", lookupPath)) {
				return pattern +"/";
			}
		}
		return null;
	}

```

回到org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.lookupHandlerMethod(String, HttpServletRequest)方法
这个时候我们可能已经找到了候选的HandlerMethod，也可能没有找到

找到和没找到，SpringMVC还有一些后置处理（比如路径变量等），我们先来看下找到后的

```
protected void org.springframework.web.servlet.mvc.method.RequestMappingInfoHandlerMapping.handleMatch(RequestMappingInfo info, String lookupPath, HttpServletRequest request) {
        
        //request.setAttribute(HandlerMapping.PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE, lookupPath);
        //记录查找到当前handlerMethod的请求路径到request中
		super.handleMatch(info, lookupPath, request);

        //最佳匹配路径
		String bestPattern;
		//路径变量
		Map<String, String> uriVariables;
		//解码后的路径变量
		Map<String, String> decodedUriVariables;
        //获取匹配到的所有pattern路径
		Set<String> patterns = info.getPatternsCondition().getPatterns();
		if (patterns.isEmpty()) {
		    //最佳匹配路径当然是request中的uri
			bestPattern = lookupPath;
			//uri变量为空
			uriVariables = Collections.emptyMap();
			decodedUriVariables = Collections.emptyMap();
		}
		else {
		    //如果patterns不为空，那么第一个被认为是最佳的路径
			bestPattern = patterns.iterator().next();
			//解析路径变量，比如bestPattern为/test/{a}，lookupPath为/test/520,那么路径变量就是a=520
			uriVariables = getPathMatcher().extractUriTemplateVariables(bestPattern, lookupPath);
			//对路径变量的值进行url解码
			decodedUriVariables = getUrlPathHelper().decodePathVariables(request, uriVariables);
		}
        //记录最佳的模式路径 HandlerMapping.class.getName() + ".bestMatchingPattern" -》 bestPattern
		request.setAttribute(BEST_MATCHING_PATTERN_ATTRIBUTE, bestPattern);
		//HandlerMapping.class.getName() + ".uriTemplateVariables" -》 decodedUriVariables
		request.setAttribute(HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE, decodedUriVariables);

        //是否需要解析矩阵变量。默认是不解析的
		if (isMatrixVariableContentAvailable()) {
		    //解析矩阵参数 比如/test/{a}，然后发来的请求是/test/arg1=0;arg2=1;arg3=2,3,4,5,6
		    //然后这里的返回值就是a -> {arg1:[0],arg2:[1],arg2:[2,3,4,5,6]}
		    //如果是这种/test/0;arg2=1;arg3=2,3,4,5,6，也就是第一个参数没有等号
		    //那么uriVariables原先对应的a=0;arg2=1;arg3=2,3,4,5,6变成了a=0
		    //然后matrixVars就是{arg2:[1],arg2:[2,3,4,5,6]}
			Map<String, MultiValueMap<String, String>> matrixVars = extractMatrixVariables(request, uriVariables);
			//HandlerMapping.class.getName() + ".matrixVariables" -》 matrixVars
			request.setAttribute(HandlerMapping.MATRIX_VARIABLES_ATTRIBUTE, matrixVars);
		}
        //设置允许返回的媒体类型
		if (!info.getProducesCondition().getProducibleMediaTypes().isEmpty()) {
			Set<MediaType> mediaTypes = info.getProducesCondition().getProducibleMediaTypes();
			//HandlerMapping.class.getName() + ".producibleMediaTypes" -> mediaTypes
			request.setAttribute(PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE, mediaTypes);
		}
	}
```

如果没有匹配到handlermethod又是怎么处理的呢？

```
protected HandlerMethod org.springframework.web.servlet.mvc.method.RequestMappingInfoHandlerMapping.handleNoMatch(Set<RequestMappingInfo> requestMappingInfos,
			String lookupPath, HttpServletRequest request) throws ServletException {
        //存放能匹配到的方法
		Set<String> allowedMethods = new LinkedHashSet<String>(4);
        //存放能够匹配上的原始RequestMappingInfo
		Set<RequestMappingInfo> patternMatches = new HashSet<RequestMappingInfo>();
		//存放既能匹配方法，又能匹配路径的RequestMappingInfo
		Set<RequestMappingInfo> patternAndMethodMatches = new HashSet<RequestMappingInfo>();

		for (RequestMappingInfo info : requestMappingInfos) {
		    //匹配路径
			if (info.getPatternsCondition().getMatchingCondition(request) != null) {
			    //如果存在匹配当前request的路径子集，那么存到patternMatches集合中
				patternMatches.add(info);
				if (info.getMethodsCondition().getMatchingCondition(request) != null) {
				    //如果连方法都能匹配到子集，那么存放到patternAndMethodMatches集合中
					patternAndMethodMatches.add(info);
				}
				else {
				    //如果方法未匹配到，那么记录当前循环到的RequestMappingInfo支持的方法
					for (RequestMethod method : info.getMethodsCondition().getMethods()) {
						allowedMethods.add(method.name());
					}
				}
			}
		}
        //如果路径都不匹配，那说明项目中确实没有定义处理这个请求的handler，返回null
		if (patternMatches.isEmpty()) {
			return null;
		}
		//如果路径匹配到了，却没有一个能够接受这个请求方法，那么没得说，项目中已经没有能够处理当前request的请求方法了
		//报错吧！
		else if (patternAndMethodMatches.isEmpty() && !allowedMethods.isEmpty()) {
			throw new HttpRequestMethodNotSupportedException(request.getMethod(), allowedMethods);
		}

		Set<MediaType> consumableMediaTypes;
		Set<MediaType> producibleMediaTypes;
		List<String[]> paramConditions;
        //如果有匹配到路径，程序也没有设置能够允许的请求方法，那么从patternMatches寻找不匹配的请求媒体类型
        //不匹配的返回媒体类型，不匹配的参数RequestMappingInfo
		if (patternAndMethodMatches.isEmpty()) {
			consumableMediaTypes = getConsumableMediaTypes(request, patternMatches);
			producibleMediaTypes = getProducibleMediaTypes(request, patternMatches);
			paramConditions = getRequestParams(request, patternMatches);
		}
		//从patternAndMethodMatches寻找不匹配的请求媒体类型
        //不匹配的返回媒体类型，不匹配的参数RequestMappingInfo
		else {
			consumableMediaTypes = getConsumableMediaTypes(request, patternAndMethodMatches);
			producibleMediaTypes = getProducibleMediaTypes(request, patternAndMethodMatches);
			paramConditions = getRequestParams(request, patternAndMethodMatches);
		}
        //然后确实发现了不匹配的请求类型的RequestMappingInfo
		if (!consumableMediaTypes.isEmpty()) {
			MediaType contentType = null;
			if (StringUtils.hasLength(request.getContentType())) {
				try {
					contentType = MediaType.parseMediaType(request.getContentType());
				}
				catch (InvalidMediaTypeException ex) {
					throw new HttpMediaTypeNotSupportedException(ex.getMessage());
				}
			}
			//抛出错误，表示你请求的媒体类型，springMVC中定义的handler不支持
			throw new HttpMediaTypeNotSupportedException(contentType, new ArrayList<MediaType>(consumableMediaTypes));
		}
		else if (!producibleMediaTypes.isEmpty()) {
		    //抛出没有可接受的返回类型
			throw new HttpMediaTypeNotAcceptableException(new ArrayList<MediaType>(producibleMediaTypes));
		}
		else if (!CollectionUtils.isEmpty(paramConditions)) {
		    //不满足指定的参数类型
			throw new UnsatisfiedServletRequestParameterException(paramConditions, request.getParameterMap());
		}
		else {
		    //其他的返回null
			return null;
		}
	}
```
回到AbstractHandlerMapping的getHandler方法

```
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		Object handler = getHandlerInternal(request);
		if (handler == null) {
		    //获取默认的handlerMethod
			handler = getDefaultHandler();
		}
		if (handler == null) {
			return null;
		}
		// Bean name or resolved handler?
		//如果是字符串类型，那么从容器中寻找
		if (handler instanceof String) {
			String handlerName = (String) handler;
			handler = getApplicationContext().getBean(handlerName);
		}
        //用拦截器和handlerMethod构建HandlerExecutionChain
		HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
		//检查request的请求头中是否存在Origin，如果有表示跨域
		if (CorsUtils.isCorsRequest(request)) {
		    //从当前HandlerMapping中获取全局的跨域请求配置
			CorsConfiguration globalConfig = this.corsConfigSource.getCorsConfiguration(request);
			//从注册mapping中获取跨域配置
			CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
			//合并
			CorsConfiguration config = (globalConfig != null ? globalConfig.combine(handlerConfig) : handlerConfig);
			//额外添加跨域相关的拦截器，用于判断是否允许跨域，不予许就被拦截，并设置禁止访问编码
			executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
		}
		return executionChain;
	}
```

获取拦截器

```
protected HandlerExecutionChain org.springframework.web.servlet.handler.AbstractHandlerMapping.getHandlerExecutionChain(Object handler, HttpServletRequest request) {
		HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
				(HandlerExecutionChain) handler : new HandlerExecutionChain(handler));

		String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);
		//this.adaptedInterceptors这里面的值是在HandlerMapping初始化设置ApplicationContext时扩展和自动装配的
		for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
		    //判断是否是映射拦截器，如果是，那么需要进行uri匹配，一般我们配置的拦截器会配置在
		    //SpringMVC的配置文件中，并设置url匹配路径，最后会被包装成这个类型
			if (interceptor instanceof MappedInterceptor) {
				MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
				//(*1*)
				if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
					chain.addInterceptor(mappedInterceptor.getInterceptor());
				}
			}
			else {
				chain.addInterceptor(interceptor);
			}
		}
		return chain;
	}
	
	//(*1*)
	public boolean matches(String lookupPath, PathMatcher pathMatcher) {
		PathMatcher pathMatcherToUse = (this.pathMatcher != null) ? this.pathMatcher : pathMatcher;
		//不拦截的路径
		if (this.excludePatterns != null) {
			for (String pattern : this.excludePatterns) {
			    //使用的AntPatchMatcher
				if (pathMatcherToUse.match(pattern, lookupPath)) {
					return false;
				}
			}
		}
		if (this.includePatterns == null) {
			return true;
		}
		else {
			for (String pattern : this.includePatterns) {
			    //匹配拦截路径
				if (pathMatcherToUse.match(pattern, lookupPath)) {
					return true;
				}
			}
			return false;
		}
	}
```

自此，我们查找的handlerMethod的任务就结束了，下一节分析一下AbstractUrlHandlerMapping
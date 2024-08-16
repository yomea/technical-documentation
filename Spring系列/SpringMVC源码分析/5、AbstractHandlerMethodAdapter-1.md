我们来分析下AbstractHandlerMethodAdapter的实现类RequestMappingHandlerAdapter

直接上类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/0dcc58df159757b56336af42c0585138.png)


```
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter
		implements BeanFactoryAware, InitializingBean {

    //自定义HandlerMethod参数解析器
	private List<HandlerMethodArgumentResolver> customArgumentResolvers;
    
    //参数解析器复合类
	private HandlerMethodArgumentResolverComposite argumentResolvers;

    //参数解析器复合类（用于@InitBinder标注的方法的参数解析）
	private HandlerMethodArgumentResolverComposite initBinderArgumentResolvers;
    
    //自定义返回值处理器
	private List<HandlerMethodReturnValueHandler> customReturnValueHandlers;

    //返回值处理复合类
	private HandlerMethodReturnValueHandlerComposite returnValueHandlers;

    //自定义ModelAndView解析器
	private List<ModelAndViewResolver> modelAndViewResolvers;

    //内容协商管理器，内部维护内容协商策略和后缀处理策略，用于解析媒体类型
	private ContentNegotiationManager contentNegotiationManager = new ContentNegotiationManager();

    //http消息转换器，把http的内容转换成想要的类型
	private List<HttpMessageConverter<?>> messageConverters;
    
    //@RequestBodyAdvice,@ResponseBodyAdvice注解的bean对象
	private List<Object> requestResponseBodyAdvice = new ArrayList<Object>();
    //web绑定初始化器
	private WebBindingInitializer webBindingInitializer;
    //异步任务执行器
	private AsyncTaskExecutor taskExecutor = new SimpleAsyncTaskExecutor("MvcAsync");

	private Long asyncRequestTimeout;
    
	private CallableProcessingInterceptor[] callableInterceptors = new CallableProcessingInterceptor[0];

	private DeferredResultProcessingInterceptor[] deferredResultInterceptors = new DeferredResultProcessingInterceptor[0];

	private boolean ignoreDefaultModelOnRedirect = false;

	private int cacheSecondsForSessionAttributeHandlers = 0;
    //session 同步，对同一个session进行同步
	private boolean synchronizeOnSession = false;
    //用于储存session属性
	private SessionAttributeStore sessionAttributeStore = new DefaultSessionAttributeStore();
    //参数名解析器
	private ParameterNameDiscoverer parameterNameDiscoverer = new DefaultParameterNameDiscoverer();
    //beanFactory
	private ConfigurableBeanFactory beanFactory;

    //@SessionAttributes标注的bean的class对象 -》 SessionAttributesHandler
	private final Map<Class<?>, SessionAttributesHandler> sessionAttributesHandlerCache =
			new ConcurrentHashMap<Class<?>, SessionAttributesHandler>(64);
			
    //@Controller或@RequestMapping标注的bean的class对象 -》 被@InitBinder注释的方法
	private final Map<Class<?>, Set<Method>> initBinderCache = new ConcurrentHashMap<Class<?>, Set<Method>>(64);
	
    //@ControllerAdvice标注的包装bean -》 被@InitBinder注释的方法
	private final Map<ControllerAdviceBean, Set<Method>> initBinderAdviceCache =
			new LinkedHashMap<ControllerAdviceBean, Set<Method>>();
			
    //@Controller或@RequestMapping标注的bean的class对象 -》 被@ModelAttribute注释的方法（不能有@RequestMapping）
	private final Map<Class<?>, Set<Method>> modelAttributeCache = new ConcurrentHashMap<Class<?>, Set<Method>>(64);
	
    //@ControllerAdvice标注的包装bean -》 被@ModelAttribute注释的方法（不能有@RequestMapping）
	private final Map<ControllerAdviceBean, Set<Method>> modelAttributeAdviceCache =
			new LinkedHashMap<ControllerAdviceBean, Set<Method>>();


	public RequestMappingHandlerAdapter() {
	    //创建StringHttpMessageConverter转化器，从名字上看，它就是把http请求的请求体数据转换成字符串
	    //它默认的支持的媒体类型为text/plain和*/*
		StringHttpMessageConverter stringHttpMessageConverter = new StringHttpMessageConverter();
		//是否从Charset.availableCharsets()获取可用的字符串设置到响应头中
		stringHttpMessageConverter.setWriteAcceptCharset(false);  // see SPR-7316

		this.messageConverters = new ArrayList<HttpMessageConverter<?>>(4);
		//把请求体转化成字节数组
		this.messageConverters.add(new ByteArrayHttpMessageConverter());
		this.messageConverters.add(stringHttpMessageConverter);
		//用于处理xml
		this.messageConverters.add(new SourceHttpMessageConverter<Source>());
		//用于处理压缩数据
		this.messageConverters.add(new AllEncompassingFormHttpMessageConverter());
	}
	
	。。。。。。
	
}
```

从类图上我们可以看到它是实现了InitializingBean接口，所以我们来看看都做了什么

```
public void org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.afterPropertiesSet() {
		// Do this first, it may add ResponseBody advice beans
		//初始化@ControllerAdvice
		initControllerAdviceCache();
        //如果argumentResolvers为空，那么获取默认的参数解析器
        //什么时候不为空呢？我们在前面分析过，SpringMVC首先是从容器中自动装配HandlerAdapter
        //如果这个HandlerAdapter是我们自己注入的，那么这里就可以带值
		if (this.argumentResolvers == null) {
		    //获取默认的参数解析器
		    //(*1*)
			List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
			this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		//设置默认的初始化参数绑定解析器
		//什么时候不为空呢？我们在前面分析过，SpringMVC首先是从容器中自动装配HandlerAdapter
        //如果这个HandlerAdapter是我们自己注入的，那么这里就可以带值
		if (this.initBinderArgumentResolvers == null) {
		    //(*2*)
			List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
			this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		//设置默认的返回值处理器
		//什么时候不为空呢？我们在前面分析过，SpringMVC首先是从容器中自动装配HandlerAdapter
        //如果这个HandlerAdapter是我们自己注入的，那么这里就可以带值
		if (this.returnValueHandlers == null) {
		    //(*3*)
			List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
			this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
		}
	}
	
	//(*1*)
	private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
		List<HandlerMethodArgumentResolver> resolvers = new ArrayList<HandlerMethodArgumentResolver>();

		// Annotation-based argument resolution
		//@RequestParam注解的参数解析
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
		//@RequestParam并且参数类型为Map，@RequestParam的name不能设值，SpringMVC会被request中所有的参数值
		//变成Map设置到这个参数上
		resolvers.add(new RequestParamMapMethodArgumentResolver());
		//@PathVariable，标注路径参数，路径参数的解析我们在分析RequestMappingHandlerMapping的时候
		//SpringMVC在寻找到HandlerMethod之后有个后置处理，就是解析uri路径参数的，然后将这些数据设置到了request中
		resolvers.add(new PathVariableMethodArgumentResolver());
		//@PathVariable,参数类型为Map，注解不能指定name
		resolvers.add(new PathVariableMapMethodArgumentResolver());
		//@MatrixVariable，如果参数是Map时，@MatrixVariable的name必须指定
		resolvers.add(new MatrixVariableMethodArgumentResolver());
		//@MatrixVariable，参数一定要是Map，并且@MatrixVariable没有指定参数名
		resolvers.add(new MatrixVariableMapMethodArgumentResolver());
		//@ModelAttribute，如果构造参数设置为true，那么即使没有标注@ModelAttribute，只要不是简单类型的参数
		//都能被接受
		resolvers.add(new ServletModelAttributeMethodProcessor(false));
		//作为参数传入时处理@RequestBody，如果作为方法返回值处理器，那么处理@ResponseBody
		//注意构造器，设置了消息转换器和全局的请求requestResponseBodyAdvice
		resolvers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
		//@RequestPart，MultipartFile或者javax.servlet.http.Part参数类型，但是绝对不能存在@RequestParam注解
		resolvers.add(new RequestPartMethodArgumentResolver(getMessageConverters(), this.requestResponseBodyAdvice));
		//@RequestHeader，并且参数类型不能是Map
		resolvers.add(new RequestHeaderMethodArgumentResolver(getBeanFactory()));
		//@RequestHeader，参数为Map
		resolvers.add(new RequestHeaderMapMethodArgumentResolver());
		//@CookieValue
		resolvers.add(new ServletCookieValueMethodArgumentResolver(getBeanFactory()));
		//@Value，它的值将从环境变量中获取
		resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));

		// Type-based argument resolution
		//处理诸如Request，Response，HttpSession等这类参数
		resolvers.add(new ServletRequestMethodArgumentResolver());
		//ServletResponse，Writer，Outputstream
		resolvers.add(new ServletResponseMethodArgumentResolver());
		//HttpEntity,RequestEntity
		resolvers.add(new HttpEntityMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
		//参数类型为RedirectAttributes
		resolvers.add(new RedirectAttributesMethodArgumentResolver());
		//参数类型为Model
		resolvers.add(new ModelMethodProcessor());
		//参数类型为Map
		resolvers.add(new MapMethodProcessor());
		//参数类型为Errors
		resolvers.add(new ErrorsMethodArgumentResolver());
		//参数类型为SessionStatus
		resolvers.add(new SessionStatusMethodArgumentResolver());
		//参数类型为UriComponentsBuilder或者ServletUriComponentsBuilder
		resolvers.add(new UriComponentsBuilderMethodArgumentResolver());

		// Custom arguments
		//添加自定义的参数解析器
		if (getCustomArgumentResolvers() != null) {
			resolvers.addAll(getCustomArgumentResolvers());
		}

		// Catch-all
		//@RequestParam,MultipartFile,javax.servlet.http.Part,简单类型参数类型
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
		//@ModelAttribute,构造参数为true，表示没有被@ModelAttribute注解的值，只要不是简单类型
		//也会进行处理
		resolvers.add(new ServletModelAttributeMethodProcessor(true));

		return resolvers;
	}
	
	//(*2*)
	private List<HandlerMethodArgumentResolver> getDefaultInitBinderArgumentResolvers() {
		List<HandlerMethodArgumentResolver> resolvers = new ArrayList<HandlerMethodArgumentResolver>();

		// Annotation-based argument resolution
		//@RequestParam,MultipartFile,javax.servlet.http.Part
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
		///@RequestParam，参数类型为Map
		resolvers.add(new RequestParamMapMethodArgumentResolver());
		//@PathVariable
		resolvers.add(new PathVariableMethodArgumentResolver());
		//@PathVariable，参数类型为Map
		resolvers.add(new PathVariableMapMethodArgumentResolver());
		//@MatrixVariable
		resolvers.add(new MatrixVariableMethodArgumentResolver());
		//@MatrixVariable，参数类型为Map
		resolvers.add(new MatrixVariableMapMethodArgumentResolver());
		//@Value
		resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));

		// Type-based argument resolution
		//request,session参数类型等
		resolvers.add(new ServletRequestMethodArgumentResolver());
		//response参数类型等
		resolvers.add(new ServletResponseMethodArgumentResolver());

		// Custom arguments
		//自定义参数类型
		if (getCustomArgumentResolvers() != null) {
			resolvers.addAll(getCustomArgumentResolvers());
		}

		// Catch-all
		//@RequestParam,MultipartFile,javax.servlet.http.Part,简单类型(构造参数指定true)
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));

		return resolvers;
	}
	
	//(*3*)
	private List<HandlerMethodReturnValueHandler> getDefaultReturnValueHandlers() {
		List<HandlerMethodReturnValueHandler> handlers = new ArrayList<HandlerMethodReturnValueHandler>();

		// Single-purpose return value types
		//返回参数类型为ModelAndView
		handlers.add(new ModelAndViewMethodReturnValueHandler());
		//返回参数类型为Model
		handlers.add(new ModelMethodProcessor());
		//返回参数类型为View
		handlers.add(new ViewMethodReturnValueHandler());
		//返回参数类型为ResponseBodyEmitter
		handlers.add(new ResponseBodyEmitterReturnValueHandler(getMessageConverters()));
		//返回参数为StreamingResponseBody，或者 ResponseEntity<StreamingResponseBody>，泛化类型必须是StreamingResponseBody
		handlers.add(new StreamingResponseBodyReturnValueHandler());
		//HttpEntity, HttpEntity
		handlers.add(new HttpEntityMethodProcessor(getMessageConverters(),
				this.contentNegotiationManager, this.requestResponseBodyAdvice));
		//HttpHeaders
		handlers.add(new HttpHeadersReturnValueHandler());
		//Callable
		handlers.add(new CallableMethodReturnValueHandler());
		//DeferredResult
		handlers.add(new DeferredResultMethodReturnValueHandler());
		//WebAsyncTask
		handlers.add(new AsyncTaskMethodReturnValueHandler(this.beanFactory));
		//ListenableFuture
		handlers.add(new ListenableFutureReturnValueHandler());
		if (completionStagePresent) {
		    //CompletionStage
			handlers.add(new CompletionStageReturnValueHandler());
		}

		// Annotation-based return value types
		//@ModelAttribute
		handlers.add(new ModelAttributeMethodProcessor(false));
		
		//@ResponseBody
		handlers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(),
				this.contentNegotiationManager, this.requestResponseBodyAdvice));

		// Multi-purpose return value types
		//CharSequence或者void
		handlers.add(new ViewNameMethodReturnValueHandler());
		//Map
		handlers.add(new MapMethodProcessor());

		// Custom return value types
		//自定义返回值处理器
		if (getCustomReturnValueHandlers() != null) {
			handlers.addAll(getCustomReturnValueHandlers());
		}

		// Catch-all
		if (!CollectionUtils.isEmpty(getModelAndViewResolvers())) {
		    //添加自定义ModelAndViewResolver，这是一个复合的类
			handlers.add(new ModelAndViewResolverMethodReturnValueHandler(getModelAndViewResolvers()));
		}
		else {
		    //@ModelAttribute，true表示没有被@ModelAttribute标注时，只要不是简单类型，也会进行处理
			handlers.add(new ModelAttributeMethodProcessor(true));
		}

		return handlers;
	}

```

> initControllerAdviceCache

```
private void initControllerAdviceCache() {
		if (getApplicationContext() == null) {
			return;
		}
		if (logger.isInfoEnabled()) {
			logger.info("Looking for @ControllerAdvice: " + getApplicationContext());
		}
        //从容器获取所有被@ControllerAdvice标注的bean，并将他们封装成ControllerAdviceBean对象
        //(*1*)
		List<ControllerAdviceBean> beans = ControllerAdviceBean.findAnnotatedBeans(getApplicationContext());
		//排序
		AnnotationAwareOrderComparator.sort(beans);

		List<Object> requestResponseBodyAdviceBeans = new ArrayList<Object>();

		for (ControllerAdviceBean bean : beans) {
		    //MODEL_ATTRIBUTE_METHODS 是一个过滤器，这个方法会把没被@RequestMapping标注 并且被 @ModelAttribute标注的方法筛选出来
			Set<Method> attrMethods = MethodIntrospector.selectMethods(bean.getBeanType(), MODEL_ATTRIBUTE_METHODS);
			if (!attrMethods.isEmpty()) {
			    //缓存 ControllerAdviceBean -> Set<Method>
				this.modelAttributeAdviceCache.put(bean, attrMethods);
				if (logger.isInfoEnabled()) {
					logger.info("Detected @ModelAttribute methods in " + bean);
				}
			}
			//INIT_BINDER_METHODS 一个筛选被@InitBinder标注的方法的过滤器
			Set<Method> binderMethods = MethodIntrospector.selectMethods(bean.getBeanType(), INIT_BINDER_METHODS);
			if (!binderMethods.isEmpty()) {
			    //ControllerAdviceBean -> Set<Method>
				this.initBinderAdviceCache.put(bean, binderMethods);
				if (logger.isInfoEnabled()) {
					logger.info("Detected @InitBinder methods in " + bean);
				}
			}
			//是否实现了RequestBodyAdvice接口，提供读取http body前的拦截与之后的拦截
			if (RequestBodyAdvice.class.isAssignableFrom(bean.getBeanType())) {
			    //保存到集合中
				requestResponseBodyAdviceBeans.add(bean);
				if (logger.isInfoEnabled()) {
					logger.info("Detected RequestBodyAdvice bean in " + bean);
				}
			}
			//是否实现了ResponseBodyAdvice接口，提供写http body前的拦截与之后的拦截
			if (ResponseBodyAdvice.class.isAssignableFrom(bean.getBeanType())) {
				requestResponseBodyAdviceBeans.add(bean);
				if (logger.isInfoEnabled()) {
					logger.info("Detected ResponseBodyAdvice bean in " + bean);
				}
			}
		}

		if (!requestResponseBodyAdviceBeans.isEmpty()) {
		    //将这些从容器中获取到的ControllerAdviceBean放在最前面
			this.requestResponseBodyAdvice.addAll(0, requestResponseBodyAdviceBeans);
		}
	}
	
	//(*1*)
	public static List<ControllerAdviceBean> org.springframework.web.method.ControllerAdviceBean.findAnnotatedBeans(ApplicationContext applicationContext) {
		List<ControllerAdviceBean> beans = new ArrayList<ControllerAdviceBean>();
		for (String name : BeanFactoryUtils.beanNamesForTypeIncludingAncestors(applicationContext, Object.class)) {
		    //寻找被@ControllerAdvice注解标注的bean
			if (applicationContext.findAnnotationOnBean(name, ControllerAdvice.class) != null) {
			    //封装成ControllerAdviceBean
			    //(*2*)
				beans.add(new ControllerAdviceBean(name, applicationContext));
			}
		}
		return beans;
	}
	
	//(*2*)
	private ControllerAdviceBean(Object bean, BeanFactory beanFactory) {
		this.bean = bean;
		this.beanFactory = beanFactory;
		Class<?> beanType;

		if (bean instanceof String) {
			String beanName = (String) bean;
			Assert.hasText(beanName, "Bean name must not be null");
			Assert.notNull(beanFactory, "BeanFactory must not be null");
			if (!beanFactory.containsBean(beanName)) {
				throw new IllegalArgumentException("BeanFactory [" + beanFactory +
						"] does not contain specified controller advice bean '" + beanName + "'");
			}
			//获取这个被@ControllerAdvice注解的bean的类型
			beanType = this.beanFactory.getType(beanName);
			//获取其order，通过@Order或者@Priority
			this.order = initOrderFromBeanType(beanType);
		}
		else {
			Assert.notNull(bean, "Bean must not be null");
			//获取bean的类型
			beanType = bean.getClass();
			//获取order，如果是实现了order接口，那么优先从实现的接口中获取
			this.order = initOrderFromBean(bean);
		}

		ControllerAdvice annotation = AnnotationUtils.findAnnotation(beanType, ControllerAdvice.class);
		if (annotation != null) {
		    //获取能够适用的基包
			this.basePackages = initBasePackages(annotation);
			//获取当前指定类型的子类（继承了当前指定类型的子类或者它自己将被拦截）
			this.assignableTypes = Arrays.asList(annotation.assignableTypes());
			//获取注解（被相应注解修饰的将被拦截）
			this.annotations = Arrays.asList(annotation.annotations());
		}
		else {
			this.basePackages = Collections.emptySet();
			this.assignableTypes = Collections.emptyList();
			this.annotations = Collections.emptyList();
		}
	}

```
好了，一切准备就绪，就等着有handler需要我们处理了，但是并不是每个handler都是能够处理的，所以还得有个方法告诉SpringMVC，我能够处理什么样的handler

```
public final boolean org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.supports(Object handler) {
        //还记得我们分析RequestMappingHandlerMapper的时候，它在初始化方法中将被@RequestMapping的方法封装成了HandlerMethod
        //所以这个方法adapter专为它而生的
		return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
	}

protected boolean org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.supportsInternal(HandlerMethod handlerMethod) {
		return true;
	}
```
如果确定某个handler可以被处理，那么就可以进行处理了

```
public final ModelAndView org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		return handleInternal(request, response, (HandlerMethod) handler);
	}
	
	protected ModelAndView org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
        
		ModelAndView mav;
		//检查request，比如请求的方法是否支持，默认是没有设置任何方法限制的
		//检查session是否是必须的，默认是false
		checkRequest(request);

		// Execute invokeHandlerMethod in synchronized block if required.
		//默认是不进行session同步的
		if (this.synchronizeOnSession) {
			HttpSession session = request.getSession(false);
			if (session != null) {
				Object mutex = WebUtils.getSessionMutex(session);
				synchronized (mutex) {
					mav = invokeHandlerMethod(request, response, handlerMethod);
				}
			}
			else {
				// No HttpSession available -> no mutex necessary
				mav = invokeHandlerMethod(request, response, handlerMethod);
			}
		}
		else {
			// No synchronization on session demanded at all...
			//调用HandlerMethod
			mav = invokeHandlerMethod(request, response, handlerMethod);
		}

        //设置请求头缓存
		if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
			if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
				applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
			}
			else {
				prepareResponse(response);
			}
		}

		return mav;
	}
```

调用HandlerMethod

```
protected ModelAndView org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
        //包装成ServletWebRequest
		ServletWebRequest webRequest = new ServletWebRequest(request, response);
        //获取数据绑定工厂
		WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
		//获取model工厂
		ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
       
       。。。。。。
	}
```

我们先来分析下如果创建数据绑定工厂

```
private WebDataBinderFactory org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.getDataBinderFactory(HandlerMethod handlerMethod) throws Exception {
        //获取handler的bean类型
		Class<?> handlerType = handlerMethod.getBeanType();
		//从缓存获取被@InitBinder的方法集合，注意这里的方法集合是handler的，不包含被@ControllerAdvice标注类的方法
		Set<Method> methods = this.initBinderCache.get(handlerType);
		if (methods == null) {
		    //如果没有，那么就进行筛选
			methods = MethodIntrospector.selectMethods(handlerType, INIT_BINDER_METHODS);
			//缓存
			this.initBinderCache.put(handlerType, methods);
		}
		//InvocableHandlerMethod继承HandlerMethod，额外提供参数解析器等的维护
		List<InvocableHandlerMethod> initBinderMethods = new ArrayList<InvocableHandlerMethod>();
		// Global methods first
		//首先设置全局@InitBinder
		for (Entry<ControllerAdviceBean, Set<Method>> entry : this.initBinderAdviceCache.entrySet()) {
		    //判断是否适用当前handler，比如包检查，类型检查，注解检查
			if (entry.getKey().isApplicableToBeanType(handlerType)) {
			    //获取handler的bean对象
				Object bean = entry.getKey().resolveBean();
				for (Method method : entry.getValue()) {
				    //创建InvocableHandlerMethod对象
				    //（*1*）
					initBinderMethods.add(createInitBinderMethod(bean, method));
				}
			}
		}
		for (Method method : methods) {
		    //处理当前handler bean的@InitBinder方法
			Object bean = handlerMethod.getBean();
			initBinderMethods.add(createInitBinderMethod(bean, method));
		}
		//创建数据绑定工厂，默认是ServletRequestDataBinderFactory
		return createDataBinderFactory(initBinderMethods);
	}
	
	//（*1*）
	private InvocableHandlerMethod org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.createInitBinderMethod(Object bean, Method method) {
	    //构建InvocableHandlerMethod，其父类是HandlerMethod，也就是一个维护bean，method，参数的封装类
		InvocableHandlerMethod binderMethod = new InvocableHandlerMethod(bean, method);
		//设置方法参数处理器
		binderMethod.setHandlerMethodArgumentResolvers(this.initBinderArgumentResolvers);
		//设置数据绑定工厂，用于创建数据绑定，webBindingInitializer是数据绑定初始化器，它给数据绑定设置类型转换器
		//属性编辑注册等初始化操作
		binderMethod.setDataBinderFactory(new DefaultDataBinderFactory(this.webBindingInitializer));
		//设置参数名解析器
		binderMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
		return binderMethod;
	}
	
```
然后接下来我们再看如何获取model工厂

```
private ModelFactory org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.getModelFactory(HandlerMethod handlerMethod, WebDataBinderFactory binderFactory) {
        //获取SessionAttributes处理器，内部维护这sessionAttributeStore和handler的type
		SessionAttributesHandler sessionAttrHandler = getSessionAttributesHandler(handlerMethod);
		Class<?> handlerType = handlerMethod.getBeanType();
		//从缓存中获取被@ModelAttribute标注的方法
		Set<Method> methods = this.modelAttributeCache.get(handlerType);
		if (methods == null) {
		    //筛选出所有被标注了@ModelAttribute注解的方法
			methods = MethodIntrospector.selectMethods(handlerType, MODEL_ATTRIBUTE_METHODS);
			//缓存
			this.modelAttributeCache.put(handlerType, methods);
		}
		List<InvocableHandlerMethod> attrMethods = new ArrayList<InvocableHandlerMethod>();
		// Global methods first
		for (Entry<ControllerAdviceBean, Set<Method>> entry : this.modelAttributeAdviceCache.entrySet()) {
		    //和@InitBinder一样筛选方式
			if (entry.getKey().isApplicableToBeanType(handlerType)) {
				Object bean = entry.getKey().resolveBean();
				for (Method method : entry.getValue()) {    
				    //(*1*)
					attrMethods.add(createModelAttributeMethod(binderFactory, bean, method));
				}
			}
		}
		for (Method method : methods) {
			Object bean = handlerMethod.getBean();
			attrMethods.add(createModelAttributeMethod(binderFactory, bean, method));
		}
		//创建ModelFactory
		//(*2*)
		return new ModelFactory(attrMethods, binderFactory, sessionAttrHandler);
	}
	
	//(*1*)
	private InvocableHandlerMethod createModelAttributeMethod(WebDataBinderFactory factory, Object bean, Method method) {
		InvocableHandlerMethod attrMethod = new InvocableHandlerMethod(bean, method);
		//设置参数解析器
		attrMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
		//设置参数名解析器
		attrMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
		attrMethod.setDataBinderFactory(factory);
		return attrMethod;
	}
	
	//(*2*)
	public ModelFactory(List<InvocableHandlerMethod> handlerMethods,
			WebDataBinderFactory binderFactory, SessionAttributesHandler attributeHandler) {

		if (handlerMethods != null) {
			for (InvocableHandlerMethod handlerMethod : handlerMethods) {
			    //包装成ModelMethod，并解析被@ModelMethod标注的参数
			    //(*3*)
				this.modelMethods.add(new ModelMethod(handlerMethod));
			}
		}
		this.dataBinderFactory = binderFactory;
		this.sessionAttributesHandler = attributeHandler;
	}
	
	//(*3*)
	//构造方法
	private ModelMethod(InvocableHandlerMethod handlerMethod) {
			this.handlerMethod = handlerMethod;
			for (MethodParameter parameter : handlerMethod.getMethodParameters()) {
				if (parameter.hasParameterAnnotation(ModelAttribute.class)) {
				    //(*4*)
					this.dependencies.add(getNameForParameter(parameter));
				}
			}
	}
	 //(*4*)
	public static String org.springframework.web.method.annotation.ModelFactory.getNameForParameter(MethodParameter parameter) {
	    //读取注解@ModelAttribute
		ModelAttribute ann = parameter.getParameterAnnotation(ModelAttribute.class);
		String name = (ann != null ? ann.value() : null);
		//如果注解没有指定名字，那么反射获取参数名
		return StringUtils.hasText(name) ? name : Conventions.getVariableNameForParameter(parameter);
	}
	
```

继续回到调用HandlerMethod的逻辑

```
protected ModelAndView org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

       。。。。。。
       //创建ServletInvocableHandlerMethod，这个ServletInvocableHandlerMethod继承自InvocableHandlerMethod
       //额外做了一些操作，读取handlerMethod方法上的@ResponseStatus注解
       //(*1*)
       ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
        //设置参数解析器
		invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
		//设置返回值参数处理器
		invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
		//设置数据绑定工厂
		invocableMethod.setDataBinderFactory(binderFactory);
		//设置参数名解析器
		invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

		ModelAndViewContainer mavContainer = new ModelAndViewContainer();
		//获取我们在DispatcherServlet的doService方法中筛选的FlashMap
		mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
		//初始化model，开始model方法的调用
		modelFactory.initModel(webRequest, mavContainer, invocableMethod);
		mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

		。。。。。。
       
	}
	
	
	 //(*1*)
	 public ServletInvocableHandlerMethod(HandlerMethod handlerMethod) {
		super(handlerMethod);
		//(*2*)
		initResponseStatus();
	}
	//(*2*)
	private void initResponseStatus() {
	    //获取@ResponseStatus注解信息
		ResponseStatus annotation = getMethodAnnotation(ResponseStatus.class);
		if (annotation != null) {
		    //返回编码
			this.responseStatus = annotation.code();
			//返回信息
			this.responseReason = annotation.reason();
		}
	}
```

初始化model

```
public void org.springframework.web.method.annotation.ModelFactory.initModel(NativeWebRequest request, ModelAndViewContainer container,
			HandlerMethod handlerMethod) throws Exception {
        //handler的类类型上如果有@SessionAttributes，那么依照这个注解设置的属性值从session中获取值
        //最后返回
		Map<String, ?> sessionAttributes = this.sessionAttributesHandler.retrieveAttributes(request);
		//合并属性，已经存在的不会被覆盖
		container.mergeAttributes(sessionAttributes);
		//调用被@ModelAttribute标注的方法
		invokeModelAttributeMethods(request, container);
        //处理当前handlerMethod中被@ModelAttribute标注的参数，如果被@ModelAttribute标注的参数在类上的
        //@SessionAttributes也标注了，那么尝试从session中获取，如果获取不到会报错
		for (String name : findSessionAttributeArguments(handlerMethod)) {
			if (!container.containsAttribute(name)) {
			    //从session中获取值
				Object value = this.sessionAttributesHandler.retrieveAttribute(request, name);
				if (value == null) {
					throw new HttpSessionRequiredException("Expected session attribute '" + name + "'");
				}
				//添加
				container.addAttribute(name, value);
			}
		}
	}
```
调用被@ModelAttribute标注的方法

```
private void org.springframework.web.method.annotation.ModelFactory.invokeModelAttributeMethods(NativeWebRequest request, ModelAndViewContainer container)
			throws Exception {
        
		while (!this.modelMethods.isEmpty()) {
		    //挑选modelMethod，并移除
		    //(*1*)
			InvocableHandlerMethod modelMethod = getNextModelMethod(container).getHandlerMethod();
			//获取其方法上的@ModelAttribute的name属性
			String modelName = modelMethod.getMethodAnnotation(ModelAttribute.class).value();
			//如果已经存在就不会再执行这个modelMethod
			if (container.containsAttribute(modelName)) {
				continue;
			}
            //调用方法
			Object returnValue = modelMethod.invokeForRequest(request, container);
			if (!modelMethod.isVoid()){
			    //如果返回值不是void，那么获取返回参数名，用于设置到ModelAndViewContainer中
				String returnValueName = getNameForReturnValue(returnValue, modelMethod.getReturnType());
				if (!container.containsAttribute(returnValueName)) {
					container.addAttribute(returnValueName, returnValue);
				}
			}
		}
	}
	
	//(*1*)
	private ModelMethod getNextModelMethod(ModelAndViewContainer container) {
		for (ModelMethod modelMethod : this.modelMethods) {
		    //检查modelMethod上的被@ModelAttribute标注的参数在container是否都存在，如果有一个不存在，那么返回false
		    //全部存在，那么直接返回，或者说modelMethod上没有被@ModelAttribute标注的参数，也会直接返回
		    //为什么这么做呢？为什么不按顺序执行呢？主要考虑到有些ModelMethod的参数值可能来自其他的modelMethod执行的结果
			if (modelMethod.checkDependencies(container)) {
				if (logger.isTraceEnabled()) {
					logger.trace("Selected @ModelAttribute method " + modelMethod);
				}
				this.modelMethods.remove(modelMethod);
				return modelMethod;
			}
		}
		//如果这里所有的ModelMethod方法，没有一个被@ModelAttribute标注的参数能够从container获取到值，获取第一个ModelMethod
		ModelMethod modelMethod = this.modelMethods.get(0);
		if (logger.isTraceEnabled()) {
			logger.trace("Selected @ModelAttribute method (not present: " +
					modelMethod.getUnresolvedDependencies(container)+ ") " + modelMethod);
		}
		//移除
		this.modelMethods.remove(modelMethod);
		return modelMethod;
	}
```

modelMethod的调用，org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(NativeWebRequest, ModelAndViewContainer, Object...)
分析了这个方法之后@InitBinder方法也是一个道理

```
//request：request，response的包装类，mavContainer：ModelAndView容器，用于维护属性值，视图等属性，providedArgs：程序提供的方法参数，一般为null
//需要解析参数
public Object org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(NativeWebRequest request, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
        //解析参数
		Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
		if (logger.isTraceEnabled()) {
			StringBuilder sb = new StringBuilder("Invoking [");
			sb.append(getBeanType().getSimpleName()).append(".");
			sb.append(getMethod().getName()).append("] method with arguments ");
			sb.append(Arrays.asList(args));
			logger.trace(sb.toString());
		}
		//反射调用方法，获取返回值
		Object returnValue = doInvoke(args);
		if (logger.isTraceEnabled()) {
			logger.trace("Method [" + getMethod().getName() + "] returned [" + returnValue + "]");
		}
		return returnValue;
	}

```
参数的解析

```
private Object[] org.springframework.web.method.support.InvocableHandlerMethod.getMethodArgumentValues(NativeWebRequest request, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
        //获取参数数组，MethodParameter维护这参数下标，类型，来源的方法等
		MethodParameter[] parameters = getMethodParameters();
		Object[] args = new Object[parameters.length];
		for (int i = 0; i < parameters.length; i++) {
			MethodParameter parameter = parameters[i];
			//设置参数名解析器
			parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
			//解析参数类型
			GenericTypeResolver.resolveParameterType(parameter, getBean().getClass());
			//从提供的参数中获取参数值，一般我们传入的参数为空
			args[i] = resolveProvidedArgument(parameter, providedArgs);
			//如果能从提供的参数中获取到值，那么跳过
			if (args[i] != null) {
				continue;
			}
			//循环方法参数解析器，找到能够支持这个参数的第一个参数解析器，并存入缓存
			//以parameter为key，解析器为value存放
			if (this.argumentResolvers.supportsParameter(parameter)) {
				try {
				    //解析参数
					args[i] = this.argumentResolvers.resolveArgument(
							parameter, mavContainer, request, this.dataBinderFactory);
					continue;
				}
				catch (Exception ex) {
					if (logger.isDebugEnabled()) {
						logger.debug(getArgumentResolutionErrorMessage("Error resolving argument", i), ex);
					}
					throw ex;
				}
			}
			if (args[i] == null) {
			    //如果还是无法解析，只好报错了
				String msg = getArgumentResolutionErrorMessage("No suitable resolver for argument", i);
				throw new IllegalStateException(msg);
			}
		}
		return args;
	}
```
可以看到当没有提供方法参数时，SpringMVC使用了参数解析器，这些参数解析器都是HandlerMethodAdapter在初始化时设置进去的，现在我们先来看看SpringMVC是怎么筛选出合适的参数解析器的

```
private HandlerMethodArgumentResolver org.springframework.web.method.support.HandlerMethodArgumentResolverComposite.getArgumentResolver(MethodParameter parameter) {
        //先从缓存中查找
		HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
		if (result == null) {
		    //循环
			for (HandlerMethodArgumentResolver methodArgumentResolver : this.argumentResolvers) {
				if (logger.isTraceEnabled()) {
					logger.trace("Testing if argument resolver [" + methodArgumentResolver + "] supports [" +
							parameter.getGenericParameterType() + "]");
				}
				//找到能够处理这个参数的第一个解析器并加入缓存
				if (methodArgumentResolver.supportsParameter(parameter)) {
					result = methodArgumentResolver;
					this.argumentResolverCache.put(parameter, result);
					break;
				}
			}
		}
		return result;
	}
```
找到解析器后，接下来就是对参数进行解析了

```
public Object org.springframework.web.method.support.HandlerMethodArgumentResolverComposite.resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        //这个逻辑就是上面那段代码，如果已经存在了就从 缓存中获取，如果没有就重新循环一遍
		HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
		//如果没有找到合适的解析器，那不好意思，直接报错吧
		if (resolver == null) {
			throw new IllegalArgumentException("Unknown parameter type [" + parameter.getParameterType().getName() + "]");
		}
		//调用解析器进行解析参数
		return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
	}
```
参数解析器有好多，我们挑一个我们经常使用的参数解析器，那就是处理@RequestBody的RequestResponseBodyMethodProcessor，内容有点多了，接下来的内容请移步到下一小节。

总结：创建HandlerMethodAdapter在构造器中添加HttpMessageConveter，然后初始化时解析ControllerAdvice，解析全局的@InitBinder，@ModelAttribute方法进行缓存，然后继续构建默认的参数解析器，构建默认的InitBinder参数解析器，构建返回值参数解析器，一切准备就绪之后，可以处理HandlerMethod了，调用方法肯定需要对方法的参数进行解析，所以在解析参宿的时候得做些准备工作，比如处理器自己的@InitBinder和@ModelAttribute方法，既然存在ModelAttribute方法，那么首先就得调用他们，应为它们将可能为后续的操作提供属性，但是调用model方法也是需要进行参数解析的，所以需要有一个绑定参数的类，那就是webDataBinder，所以要事先创建好他们的工厂，已被后续的参数绑定，最后开始解析参数。接下里就是下一节的内容了。
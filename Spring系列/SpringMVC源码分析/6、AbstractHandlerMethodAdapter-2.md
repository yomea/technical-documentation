接着上一小节，上面提到RequestResponseBodyMethodProcessor，这个参数解析器既实现了HandlerMethodArgumentResolver，也实现了HandlerMethodReturnValueHandler，所以它既能处理方法参数，也能处理方法返回值

```
public Object org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor.resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        //RequestResponseBodyMethodProcessor是一个用于处理http请求体的参数处理器，但是http请求体传递过来的参数有不同的表现形式，比如有json类型的，有xml数据流形式的，所以需要额外的http消息转换器进行处理，消息转换器是我们在创建HandlerAdapter的初始化方法中设置到RequestResponseBodyMethodProcessor中的
		Object arg = readWithMessageConverters(webRequest, parameter, parameter.getGenericParameterType());
		//获取参数名
		String name = Conventions.getVariableNameForParameter(parameter);
        //创建参数绑定，为什么要使用参数绑定？参数绑定提供了将我们解析后参数如何设置到方法参数上的能力
        //我们自定义的参数转换器，属性编辑器都是通过它来进行维护，使用。
		WebDataBinder binder = binderFactory.createBinder(webRequest, arg, name);
		if (arg != null) {
		    //校验参数
			validateIfApplicable(binder, parameter);
			if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
			    //绑定出错时，抛出错误
				throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
			}
		}
		//记录参数绑定的结果
		mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());

		return arg;
	}
```
挑选http消息解析器

```
protected <T> Object org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor.readWithMessageConverters(NativeWebRequest webRequest, MethodParameter methodParam,
			Type paramType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {
        //获取request
		HttpServletRequest servletRequest = webRequest.getNativeRequest(HttpServletRequest.class);
		//包装本地request，ServletServerHttpRequest提供更多的扩展方法，比如直接获取请求体数据流等
		ServletServerHttpRequest inputMessage = new ServletServerHttpRequest(servletRequest);
        
		Object arg = readWithMessageConverters(inputMessage, methodParam, paramType);
		//无法解析，抛错
		if (arg == null) {
			if (methodParam.getParameterAnnotation(RequestBody.class).required()) {
				throw new HttpMessageNotReadableException("Required request body is missing: " +
						methodParam.getMethod().toGenericString());
			}
		}
		return arg;
	}
```
> readWithMessageConverters

```
protected <T> Object org.springframework.web.servlet.mvc.method.annotation.AbstractMessageConverterMethodArgumentResolver.readWithMessageConverters(HttpInputMessage inputMessage, MethodParameter param,
			Type targetType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {

		MediaType contentType;
		boolean noContentType = false;
		try {
		    //从请求头中获取请求的媒体类型
			contentType = inputMessage.getHeaders().getContentType();
		}
		catch (InvalidMediaTypeException ex) {
			throw new HttpMediaTypeNotSupportedException(ex.getMessage());
		}
		if (contentType == null) {
		    //标记请求没有提供媒体类型
			noContentType = true;
			//设置默认的媒体类型为流
			contentType = MediaType.APPLICATION_OCTET_STREAM;
		}
        //获取这个参数的包含类
		Class<?> contextClass = (param != null ? param.getContainingClass() : null);
		//获取参数类型
		Class<T> targetClass = (targetType instanceof Class<?> ? (Class<T>) targetType : null);
		if (targetClass == null) {
		    //解析参数类型
			ResolvableType resolvableType = (param != null ?
					ResolvableType.forMethodParameter(param) : ResolvableType.forType(targetType));
			targetClass = (Class<T>) resolvableType.resolve();
		}
        //获取请求方法
		HttpMethod httpMethod = ((HttpRequest) inputMessage).getMethod();
		Object body = NO_VALUE;

		try {
			inputMessage = new EmptyBodyCheckingHttpInputMessage(inputMessage);
            //寻找支持此参数的消息转换器
			for (HttpMessageConverter<?> converter : this.messageConverters) {
				Class<HttpMessageConverter<?>> converterType = (Class<HttpMessageConverter<?>>) converter.getClass();
				//GenericHttpMessageConverter继承自HttpMessageConverter，它的read方法额外提供设置包含类的参数
				//write方法额外提供了泛型类型参数
				if (converter instanceof GenericHttpMessageConverter) {
					GenericHttpMessageConverter<?> genericConverter = (GenericHttpMessageConverter<?>) converter;
					//判断是否支持
					if (genericConverter.canRead(targetType, contextClass, contentType)) {
						if (logger.isDebugEnabled()) {
							logger.debug("Read [" + targetType + "] as \"" + contentType + "\" with [" + converter + "]");
						}
						if (inputMessage.getBody() != null) {
						    //RequestAdvice前置拦截，RequestAdvice是在创建当前参数解析器时设置的
							inputMessage = getAdvice().beforeBodyRead(inputMessage, param, targetType, converterType);
							//解析参数值
							body = genericConverter.read(targetType, contextClass, inputMessage);
							//RequestAdvice后置拦截
							body = getAdvice().afterBodyRead(body, inputMessage, param, targetType, converterType);
						}
						else {
							body = null;
							//处理空请求体，一般项目中很少使用实现了RequestAdvice接口的ControllerAdvice
							body = getAdvice().handleEmptyBody(body, inputMessage, param, targetType, converterType);
						}
						break;
					}
				}
				else if (targetClass != null) {
				    //差不多的处理方式，只是调用的方法参数不同罢了
					if (converter.canRead(targetClass, contentType)) {
						if (logger.isDebugEnabled()) {
							logger.debug("Read [" + targetType + "] as \"" + contentType + "\" with [" + converter + "]");
						}
						if (inputMessage.getBody() != null) {
							inputMessage = getAdvice().beforeBodyRead(inputMessage, param, targetType, converterType);
							body = ((HttpMessageConverter<T>) converter).read(targetClass, inputMessage);
							body = getAdvice().afterBodyRead(body, inputMessage, param, targetType, converterType);
						}
						else {
							body = null;
							body = getAdvice().handleEmptyBody(body, inputMessage, param, targetType, converterType);
						}
						break;
					}
				}
			}
		}
		catch (IOException ex) {
			throw new HttpMessageNotReadableException("Could not read document: " + ex.getMessage(), ex);
		}

		if (body == NO_VALUE) {
			if (httpMethod == null || !SUPPORTED_METHODS.contains(httpMethod) ||
					(noContentType && inputMessage.getBody() == null)) {
				return null;
			}
			throw new HttpMediaTypeNotSupportedException(contentType, this.allSupportedMediaTypes);
		}

		return body;
	}
```
我们挑选一个HttpMessageConverter进行分析，就拿MappingJackson2HttpMessageConverter进行分析吧

首先我们按照参数解析器筛选，调用消息转换器的顺序进行分析吧

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/69685aeb5b172046dbbc9d105bae6228.png)

```
public boolean org.springframework.http.converter.json.AbstractJackson2HttpMessageConverter.canRead(Type type, Class<?> contextClass, MediaType mediaType) {
        //判断是否是支持的媒体类型，消息转换器的默认媒体类型我们可以自定义设置
		if (!canRead(mediaType)) {
			return false;
		}
		//获取java类型，这个JavaType用于JackSon这个json框架判断是否能够进行序列化的类型
		JavaType javaType = getJavaType(type, contextClass);
		if (!jackson23Available || !logger.isWarnEnabled()) {
			return this.objectMapper.canDeserialize(javaType);
		}
		AtomicReference<Throwable> causeRef = new AtomicReference<Throwable>();
		//如果可以进行反序列化，直接返回true
		if (this.objectMapper.canDeserialize(javaType, causeRef)) {
			return true;
		}
		//如果有错误提示，那么打印日志
		Throwable cause = causeRef.get();
		if (cause != null) {
			String msg = "Failed to evaluate Jackson deserialization for type " + javaType;
			if (logger.isDebugEnabled()) {
				logger.warn(msg, cause);
			}
			else {
				logger.warn(msg + ": " + cause);
			}
		}
		return false;
	}
	
```

> read

```
public Object org.springframework.http.converter.json.AbstractJackson2HttpMessageConverter.read(Type type, Class<?> contextClass, HttpInputMessage inputMessage)
			throws IOException, HttpMessageNotReadableException {

		JavaType javaType = getJavaType(type, contextClass);
		return readJavaType(javaType, inputMessage);
	}

	@SuppressWarnings("deprecation")
	private Object org.springframework.http.converter.json.AbstractJackson2HttpMessageConverter.readJavaType(JavaType javaType, HttpInputMessage inputMessage) {
		try {
			if (inputMessage instanceof MappingJacksonInputMessage) {
				Class<?> deserializationView = ((MappingJacksonInputMessage) inputMessage).getDeserializationView();
				if (deserializationView != null) {
					return this.objectMapper.readerWithView(deserializationView).withType(javaType).
							readValue(inputMessage.getBody());
				}
			}
			//使用Jackson进行json反序列化
			return this.objectMapper.readValue(inputMessage.getBody(), javaType);
		}
		catch (IOException ex) {
			throw new HttpMessageNotReadableException("Could not read document: " + ex.getMessage(), ex);
		}
	}
```
顺便把写也说下，返回值处理的时候就不要不再赘述了

```
public boolean org.springframework.http.converter.json.AbstractJackson2HttpMessageConverter.canWrite(Class<?> clazz, MediaType mediaType) {
        //和read差不多的逻辑
		if (!canWrite(mediaType)) {
			return false;
		}
		if (!jackson23Available || !logger.isWarnEnabled()) {
			return this.objectMapper.canSerialize(clazz);
		}
		AtomicReference<Throwable> causeRef = new AtomicReference<Throwable>();
		if (this.objectMapper.canSerialize(clazz, causeRef)) {
			return true;
		}
		Throwable cause = causeRef.get();
		if (cause != null) {
			String msg = "Failed to evaluate Jackson serialization for type [" + clazz + "]";
			if (logger.isDebugEnabled()) {
				logger.warn(msg, cause);
			}
			else {
				logger.warn(msg + ": " + cause);
			}
		}
		return false;
	}
```
写

```
public final void org.springframework.http.converter.AbstractHttpMessageConverter.write(final T t, MediaType contentType, HttpOutputMessage outputMessage)
			throws IOException, HttpMessageNotWritableException {

		final HttpHeaders headers = outputMessage.getHeaders();
		//设置默认的请求头，比如内容长度，返回的媒体雷星
		addDefaultHeaders(headers, t, contentType);

		if (outputMessage instanceof StreamingHttpOutputMessage) {
			StreamingHttpOutputMessage streamingOutputMessage =
					(StreamingHttpOutputMessage) outputMessage;
			streamingOutputMessage.setBody(new StreamingHttpOutputMessage.Body() {
				@Override
				public void writeTo(final OutputStream outputStream) throws IOException {
				    //将值写到outputMessage的body中
					writeInternal(t, new HttpOutputMessage() {
						@Override
						public OutputStream getBody() throws IOException {
							return outputStream;
						}
						@Override
						public HttpHeaders getHeaders() {
							return headers;
						}
					});
				}
			});
		}
		else {
		    //(*1*)
			writeInternal(t, outputMessage);
			outputMessage.getBody().flush();
		}
	}
	
	//(*1*)
	protected void org.springframework.http.converter.json.AbstractJackson2HttpMessageConverter.writeInternal(Object object, Type type, HttpOutputMessage outputMessage)
			throws IOException, HttpMessageNotWritableException {
        //从媒体类型的参数中获取编码，比如application/json;charset=utf-8,获取到utf-8
		JsonEncoding encoding = getJsonEncoding(outputMessage.getHeaders().getContentType());
		//创建Jackson的json生成器
		JsonGenerator generator = this.objectMapper.getFactory().createGenerator(outputMessage.getBody(), encoding);
		try {
		    //写入前缀，空方法
			writePrefix(generator, object);

			Class<?> serializationView = null;
			FilterProvider filters = null;
			Object value = object;
			JavaType javaType = null;
			
			。。。。。。省略部分代码，直接看到这个方法即可，将value值写到json生成器中，最后flush将所有数据都推到响应体中
			objectWriter.writeValue(generator, value);

			writeSuffix(generator, object);
			generator.flush();

		}
		catch (JsonProcessingException ex) {
			throw new HttpMessageNotWritableException("Could not write content: " + ex.getMessage(), ex);
		}
	}
	
```

好了，参数解析出来之后就要将参数绑定到方法上，那么我们的数据绑定将做什么呢？

```
//webRequest：维护中request和response，target：解析出来的参数值，objectName：参数名称
public final WebDataBinder org.springframework.web.bind.support.DefaultDataBinderFactory.createBinder(NativeWebRequest webRequest, Object target, String objectName)
			throws Exception {
        //创建数据绑定，这样一看每个方法参数都会创建一个属于他们的参数绑定对象
        //new ExtendedServletRequestDataBinder(target, objectName);
		WebDataBinder dataBinder = createBinderInstance(target, objectName, webRequest);
		if (this.initializer != null) {
		    //初始化数据绑定
			this.initializer.initBinder(dataBinder, webRequest);
		}
		//调用参数绑定方法，这时我们可以添加自己的属性编辑器
		initBinder(dataBinder, webRequest);
		return dataBinder;
	}
```
初始化数据绑定实例

```
public void org.springframework.web.bind.support.ConfigurableWebBindingInitializer.initBinder(WebDataBinder binder, WebRequest request) {
        //设置是否允许嵌入路径的自动创建，比如某个list字段，前端提交的表单是house.fruitBasket[1]=banner
        //.fruitBasket[1]这个表示嵌入的路径，如果为false，那么就不会创建fruitBasket实例
		binder.setAutoGrowNestedPaths(this.autoGrowNestedPaths);
		//直接字段访问，默认为false，如果为true，将创建DirectFieldBindingResult，它获取的访问器是DirectFieldAccessor
		//这个字段访问器将直接通过反射破坏字段的私有访问权限，直接调用Field#set方法设值
		if (this.directFieldAccess) {
			binder.initDirectFieldAccess();
		}
		//消息编码处理器，一般用于参数校验，解析错误编码，然后构建字段提示信息
		if (this.messageCodesResolver != null) {
			binder.setMessageCodesResolver(this.messageCodesResolver);
		}
		//设值绑定错误时的处理程序，将会使用上面设置的消息编码解析器来构建自己的错误信息
		//然后设置到BindingResult中
		if (this.bindingErrorProcessor != null) {
			binder.setBindingErrorProcessor(this.bindingErrorProcessor);
		}
		//设置校验器
		if (this.validator != null && binder.getTarget() != null &&
				this.validator.supports(binder.getTarget().getClass())) {
			binder.setValidator(this.validator);
		}
		//设置转换服务
		if (this.conversionService != null) {
			binder.setConversionService(this.conversionService);
		}
		//注册属性编辑器
		if (this.propertyEditorRegistrars != null) {
			for (PropertyEditorRegistrar propertyEditorRegistrar : this.propertyEditorRegistrars) {
				propertyEditorRegistrar.registerCustomEditors(binder);
			}
		}
	}
```
然后再看到调用参数绑定方法的实现

```
public void org.springframework.web.method.annotation.InitBinderDataBinderFactory.initBinder(WebDataBinder binder, NativeWebRequest request) throws Exception {
		for (InvocableHandlerMethod binderMethod : this.binderMethods) {
		    //是否适用于当前参数
		    //(*1*)
			if (isBinderMethodApplicable(binderMethod, binder)) {
			    //调用InvocableHandlerMethod，这个调用过程就不再赘述了
			    //这里SpringMVC提供了一个参数WebDataBinder，这样的话我们可以向它注册自定义的属性编辑器等
				Object returnValue = binderMethod.invokeForRequest(request, null, binder);
				//@InitBinder方法不允许有返回值，因为没有意义
				if (returnValue != null) {
					throw new IllegalStateException("@InitBinder methods should return void: " + binderMethod);
				}
			}
		}
	}
	
	//(*1*)
	protected boolean org.springframework.web.method.annotation.InitBinderDataBinderFactory.isBinderMethodApplicable(HandlerMethod initBinderMethod, WebDataBinder binder) {
	    //获取@InitBinder注解
		InitBinder annot = initBinderMethod.getMethodAnnotation(InitBinder.class);
		Collection<String> names = Arrays.asList(annot.value());
		//如果没有指定参数名，或者指定的参数名是当前参数名，返回true
		return (names.size() == 0 || names.contains(binder.getObjectName()));
	}
```
校验参数值

```
protected void org.springframework.web.servlet.mvc.method.annotation.AbstractMessageConverterMethodArgumentResolver.validateIfApplicable(WebDataBinder binder, MethodParameter methodParam) {
        //获取参数上的所有注解
		Annotation[] annotations = methodParam.getParameterAnnotations();
		for (Annotation ann : annotations) {
		    //如果存在@Validated元注解，那么会返回当前ann注解的代理注解，在spring中元注解就是当前注解的父类
			Validated validatedAnn = AnnotationUtils.getAnnotation(ann, Validated.class);
			//如果没有这个注解，那么查看当前注解是否是以Valid开头的
			if (validatedAnn != null || ann.annotationType().getSimpleName().startsWith("Valid")) {
				Object hints = (validatedAnn != null ? validatedAnn.value() : AnnotationUtils.getValue(ann));
				Object[] validationHints = (hints instanceof Object[] ? (Object[]) hints : new Object[] {hints});
				binder.validate(validationHints);
				break;
			}
		}
	}
```
我们上面的方法中似乎没有看到参数是如果进行绑定的，难道是SpringMVC漏操作了吗？肯定不是的，那是因为我们使用是RequestResponseBodyMethodProcessor
这个类只处理被@RequestBody标注的参数，意思就是将http请求体的内容进行json转化获取xml等其他形式的转换，得到的值不需要进行web参数绑定

所以为了研究参数的绑定我们选用另外一个参数解析器 -》 ModelAttributeMethodProcessor，这个类会处理被@ModelAttribute标注的参数

```
public final Object ModelAttributeMethodProcessor.resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        //获取参数名
		String name = ModelFactory.getNameForParameter(parameter);
		//mavContainer如果不存在这个值，那么会创建一个空的对象返回回来
		Object attribute = (mavContainer.containsAttribute(name) ?
				mavContainer.getModel().get(name) : createAttribute(name, parameter, binderFactory, webRequest));
        //创建web数据绑定
		WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);
		if (binder.getTarget() != null) {
		    //这里就是参数绑定的逻辑，虽然创建参数对象，但是这个对象还是空白的，需要将请求参数的数据绑定上去
			bindRequestParameters(binder, webRequest);
			//校验
			validateIfApplicable(binder, parameter);
			if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
				throw new BindException(binder.getBindingResult());
			}
		}

		// Add resolved attribute and BindingResult at the end of the model
		//将创建的@ModeAttribute注解参数设置以解析到的名字做key设置到map中并获取BindingResult对象
		Map<String, Object> bindingResultModel = binder.getBindingResult().getModel();
		//移除已经存在的属性
		mavContainer.removeAttributes(bindingResultModel);
		//添加到最后
		mavContainer.addAllAttributes(bindingResultModel);
        //进行参数转换
		return binder.convertIfNecessary(binder.getTarget(), parameter.getParameterType(), parameter);
	}
```
参数绑定

```
protected void org.springframework.web.servlet.mvc.method.annotation.ServletModelAttributeMethodProcessor.bindRequestParameters(WebDataBinder binder, NativeWebRequest request) {
        //获取本地（javax包下的那个request）request
		ServletRequest servletRequest = request.getNativeRequest(ServletRequest.class);
		ServletRequestDataBinder servletBinder = (ServletRequestDataBinder) binder;
		//绑定
		//(*1*)
		servletBinder.bind(servletRequest);		
	}
	
	//(*1*)
	public void org.springframework.web.bind.ServletRequestDataBinder.bind(ServletRequest request) {
	    //构建参数，内部构造器会把request中请求参数封装成一个个PropertyValue
		MutablePropertyValues mpvs = new ServletRequestParameterPropertyValues(request);
		//检查这个请求是否为上传请求
		MultipartRequest multipartRequest = WebUtils.getNativeRequest(request, MultipartRequest.class);
		if (multipartRequest != null) {
		    //如果是的话，那么再构建封装MultiFile的PropertyValue
			bindMultipart(multipartRequest.getMultiFileMap(), mpvs);
		}
		//把路径变量的值封装成PropertyValue
		addBindValues(mpvs, request);
		//(*2*)
		doBind(mpvs);
	}
	
	//(*2*)
	protected void org.springframework.web.bind.WebDataBinder.doBind(MutablePropertyValues mpvs) {
	    //检查并处理默认前缀，这个非常有用，如果我们的方法参数有两个对象，比如Person和Animal他们都一个共同的字段name
	    //当前前端提交参数的时候可定不是这样name=Amy&name=dog，SpringMVC不知道设置哪个
	    //那么就需要有前缀这东西，那么前端提交的时候就是person.name=Amy&animal.name=dog
	    //这样我们在SpringMVC中写两个@InitBinder方法，指定处理这两个参数的参数前缀
	    //(*3*)
		checkFieldDefaults(mpvs);
		//检查并处理标记前缀，和checkFieldDefaults类似，只不过被它标识的前缀属性值都被
		//设置成空，比如Boolean就设置为false，数组就设置为空数组，其他的都设置为null
		checkFieldMarkers(mpvs);
		//(*4*)
		super.doBind(mpvs);
	}
	
	//(*3*)
	protected void org.springframework.web.bind.WebDataBinder.checkFieldDefaults(MutablePropertyValues mpvs) {
	    //前缀如果不为空
		if (getFieldDefaultPrefix() != null) {
		    //获取前缀
			String fieldDefaultPrefix = getFieldDefaultPrefix();
			PropertyValue[] pvArray = mpvs.getPropertyValues();
			for (PropertyValue pv : pvArray) {
			    //循环查找是否有以这个为前缀的参数名
				if (pv.getName().startsWith(fieldDefaultPrefix)) {
				    //去掉前缀
					String field = pv.getName().substring(fieldDefaultPrefix.length());
					if (getPropertyAccessor().isWritableProperty(field) && !mpvs.contains(field)) {
					    //如果不包含这个参数，那么设置进去
						mpvs.add(field, pv.getValue());
					}
					//并把原来的移除掉
					mpvs.removePropertyValue(pv);
				}
			}
		}
	}
	
	//(*4*)
	protected void org.springframework.validation.DataBinder.doBind(MutablePropertyValues mpvs) {
	    //检查是否是允许的属性，如果不是将被移除，可以自定义，在@InitBinder方法中设置
		checkAllowedFields(mpvs);
		//检查是否缺少必须的属性，可以自定，在@InitBinder方法中设置
		checkRequiredFields(mpvs);
		//开始设置值，属性的设置，我们在spring源码分析中分析过，此处不再赘述
		applyPropertyValues(mpvs);
	}
```

好了，至此我们的参数就已经创建好了

现在我们回到HandlerMethod的调用

```
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

		ServletWebRequest webRequest = new ServletWebRequest(request, response);
        //获取数据绑定工厂，实际它内部应用的每个InitBinder方法都维护一个默认的参数绑定工厂
		WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
		//创建model工厂
		ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
        //创建ServletInvocableHandlerMethod，并解析handlerMethod的@ResponseStatus
		ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
		//设置方法参数解析器
		invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
		//设置方法返回参数处理器
		invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
		//设置数据绑定工厂
		invocableMethod.setDataBinderFactory(binderFactory);
		//设置参数名解析器
		invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

		ModelAndViewContainer mavContainer = new ModelAndViewContainer();
		//合并FlashMap中的内容，这个FlashMap是SpringMVC在DispatcherServlet的doService方法中筛选出的最佳闪存
		mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
		//初始化model，会将handler类型注解@SessionAttributes指定的属性从session中获取，然后进行合并
		//接下来就是调用model方法，最后将返回的结构存储到mavContainer中
		modelFactory.initModel(webRequest, mavContainer, invocableMethod);
		//当重定向是否忽略调用默认的model中的数据，默认是false
		mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);
        创建异步请求包装类，持有request和response
		AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
		//设置异步请求超时时间，超过这个时间，异步请求被认为失败
		asyncWebRequest.setTimeout(this.asyncRequestTimeout);
        //获取异步管理器
		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
		//设置任务执行线程池，额。。。。。。怎么说呢，感觉它不想线程池，因为每提交一个任务就会创建一个线程然后执行他们
		//而且还不会缓存这些线程对象
		asyncManager.setTaskExecutor(this.taskExecutor);
		//设置请求包装类asyncWebRequest
		asyncManager.setAsyncWebRequest(asyncWebRequest);
		//注册回调拦截器
		asyncManager.registerCallableInterceptors(this.callableInterceptors);
		//设置延迟回调拦截器
		asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);
        
        //是否已经处理完
		if (asyncManager.hasConcurrentResult()) {
		    //获取处理后的结构
			Object result = asyncManager.getConcurrentResult();
			//获取ModelAndViewContainer
			mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
			//将值复位
			asyncManager.clearConcurrentResult();
			if (logger.isDebugEnabled()) {
				logger.debug("Found concurrent result value [" + result + "]");
			}
			//将result的值进行一层包装，变成新的handler，当invocableMethod被调用的时候直接返回，然后通过一系列的
			//返回值处理器进行处理
			invocableMethod = invocableMethod.wrapConcurrentResult(result);
		}
        //调用方法
		invocableMethod.invokeAndHandle(webRequest, mavContainer);
		//如果异步已经启动，返回null
		if (asyncManager.isConcurrentHandlingStarted()) {
			return null;
		}
        //更新model的属性值，如果有重定向属性值的话，会存储outputFlashMap
        //(*1*)
		return getModelAndView(mavContainer, modelFactory, webRequest);
	}
	
	//(*1*)
	private ModelAndView getModelAndView(ModelAndViewContainer mavContainer,
			ModelFactory modelFactory, NativeWebRequest webRequest) throws Exception {
        //更新model中的属性
		modelFactory.updateModel(webRequest, mavContainer);
		//如果已经标记为响应完成，那么直接返回null，将不会进行视图的解析
		if (mavContainer.isRequestHandled()) {
			return null;
		}
		ModelMap model = mavContainer.getModel();
		ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model);
		if (!mavContainer.isViewReference()) {
			mav.setView((View) mavContainer.getView());
		}
		//设置重定向model
		if (model instanceof RedirectAttributes) {
			Map<String, ?> flashAttributes = ((RedirectAttributes) model).getFlashAttributes();
			HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
			RequestContextUtils.getOutputFlashMap(request).putAll(flashAttributes);
		}
		return mav;
	}

```
调用我们ServletInvocableHandlerMethod

```
public void org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletWebRequest webRequest,
			ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
        //调用方法，此处不再赘述
		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
		//设置响应头状态码和消息，这个状态还记得是在创建ServletInvocableHandlerMethod时，它在构造器中解析handlerMethod上的@ResponseStatus中得到的
		setResponseStatus(webRequest);

        //如果返回值为null
		if (returnValue == null) {
		    //如果请求未过期，或者已经设置了响应状态，或者已经处理完成（一般返回json的在handler中就已经处理完了）
			if (isRequestNotModified(webRequest) || hasResponseStatus() || mavContainer.isRequestHandled()) {
			    //那么就设置请求处理已经完成，不需要再做处理了
				mavContainer.setRequestHandled(true);
				return;
			}
		}
		//如果设置了响应缘由，那么也标识为完成
		else if (StringUtils.hasText(this.responseReason)) {
			mavContainer.setRequestHandled(true);
			return;
		}
        //否则设置为false，表示还需进一步处理
		mavContainer.setRequestHandled(false);
		try {
		    //调用返回值处理器处理返回参数
			this.returnValueHandlers.handleReturnValue(
					returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
		}
		catch (Exception ex) {
			if (logger.isTraceEnabled()) {
				logger.trace(getReturnValueHandlingErrorMessage("Error handling return value", returnValue), ex);
			}
			throw ex;
		}
	}
```

筛选HandlerMethodReturnValueHandler

```
public void org.springframework.web.method.support.HandlerMethodReturnValueHandlerComposite.handleReturnValue(Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
        //筛选HandlerMethodReturnValueHandler
        //(*1*)
		HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
		if (handler == null) {
			throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
		}
		//处理返回值
		handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
	}
	
	//(*1*)
	private HandlerMethodReturnValueHandler org.springframework.web.method.support.HandlerMethodReturnValueHandlerComposite.selectHandler(Object value, MethodParameter returnType) {
	    //判断是否是异步值，其处理的returnHandler是AsyncHandlerMethodReturnValueHandler类型
	    //并且可以用于判断这个返回值value是否是WebAsyncTask类型的
		boolean isAsyncValue = isAsyncReturnValue(value, returnType);
		for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
			if (isAsyncValue && !(handler instanceof AsyncHandlerMethodReturnValueHandler)) {
				continue;
			}
			//判断是否支持当前返回参数
			if (handler.supportsReturnType(returnType)) {
				return handler;
			}
		}
		return null;
	}
	
```
返回参数的处理器有很多，我们也挑选一个常用的，它和请求参数处理一样的类，它就是RequestResponseBodyMethodProcessor
首先来看到它支持什么样的返回值

```
public boolean org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor.supportsReturnType(MethodParameter returnType) {
        //判断当前handler类上面是否存在@ResponseBody注解或者方法上是否有@ResponseBody注解
		return (AnnotationUtils.findAnnotation(returnType.getContainingClass(), ResponseBody.class) != null ||
				returnType.getMethodAnnotation(ResponseBody.class) != null);
	}
```
好，已经知道它可以处理什么的类型了，那么接下来就来学习下这个返回值处理器是怎么处理返回值的

```
public void org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor.handleReturnValue(Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
        //直接设置为true，表示请求被处理完成
		mavContainer.setRequestHandled(true);

		// Try even with null return value. ResponseBodyAdvice could get involved.
		//调用消息转换器处理参数
		writeWithMessageConverters(returnValue, returnType, webRequest);
	}
```
我们可以看到，SpringMVC在参数返回值处理时也使用了消息转换器

```
protected <T> void org.springframework.web.servlet.mvc.method.annotation.AbstractMessageConverterMethodProcessor.writeWithMessageConverters(T returnValue, MethodParameter returnType, NativeWebRequest webRequest)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
        //包装成ServletServerHttpRequest，提供了额外的能力，比如获取http请求体
		ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
		//包装响应
		ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);
		//(*1*)
		writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
	}
	
	//(*1*)
	//returnValue：调用方法的返回值，returnType：返回参数类型，inputMessage：请求消息，outputMessage：响应消息
	protected <T> void org.springframework.web.servlet.mvc.method.annotation.AbstractMessageConverterMethodProcessor.writeWithMessageConverters(T returnValue, MethodParameter returnType,
			ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
        //获取返回值类型
		Class<?> returnValueClass = getReturnValueType(returnValue, returnType);
		//获取泛化的参数类型，如果是HttpEntity，将获取其指定的泛型类型
		Type returnValueType = getGenericType(returnType);
		//后去本地请求参数
		HttpServletRequest servletRequest = inputMessage.getServletRequest();
		//获取请求可接受的媒体类型
		List<MediaType> requestedMediaTypes = getAcceptableMediaTypes(servletRequest);
		//获取可返回的媒体类型
		List<MediaType> producibleMediaTypes = getProducibleMediaTypes(servletRequest, returnValueClass, returnValueType);
        //如果存在返回值，但是可返回的媒体类型没有，那没法玩，只能抛错了
		if (returnValue != null && producibleMediaTypes.isEmpty()) {
			throw new IllegalArgumentException("No converter found for return value of type: " + returnValueClass);
		}

		Set<MediaType> compatibleMediaTypes = new LinkedHashSet<MediaType>();
		//获取可兼容的媒体类型
		for (MediaType requestedType : requestedMediaTypes) {
			for (MediaType producibleType : producibleMediaTypes) {
				if (requestedType.isCompatibleWith(producibleType)) {
					compatibleMediaTypes.add(getMostSpecificMediaType(requestedType, producibleType));
				}
			}
		}
		//如果没有找到，并且返回值又不为空的话，只能抛错错误
		if (compatibleMediaTypes.isEmpty()) {
			if (returnValue != null) {
				throw new HttpMediaTypeNotAcceptableException(producibleMediaTypes);
			}
			return;
		}

		List<MediaType> mediaTypes = new ArrayList<MediaType>(compatibleMediaTypes);
		//进行排序
		MediaType.sortBySpecificityAndQuality(mediaTypes);

		MediaType selectedMediaType = null;
		for (MediaType mediaType : mediaTypes) {
		    //寻找具体的媒体类型
			if (mediaType.isConcrete()) {
				selectedMediaType = mediaType;
				break;
			}
			//如果没有具体的媒体类型，那么只能设置一个默认的媒体类型了application/octet-stream
			else if (mediaType.equals(MediaType.ALL) || mediaType.equals(MEDIA_TYPE_APPLICATION)) {
				selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
				break;
			}
		}
        
		if (selectedMediaType != null) {
		    //移除掉媒体类型后面指定权重的值，比如text/html;q=0.9
		    //会把q去掉
			selectedMediaType = selectedMediaType.removeQualityValue();
			for (HttpMessageConverter<?> messageConverter : this.messageConverters) {
				if (messageConverter instanceof GenericHttpMessageConverter) {
				    //寻找能够处理此返回参数的消息转换器
					if (((GenericHttpMessageConverter<T>) messageConverter).canWrite(returnValueType,
							returnValueClass, selectedMediaType)) {
						//ResponseAdvice前置处理
						returnValue = (T) getAdvice().beforeBodyWrite(returnValue, returnType, selectedMediaType,
								(Class<? extends HttpMessageConverter<?>>) messageConverter.getClass(),
								inputMessage, outputMessage);
						if (returnValue != null) {
							addContentDispositionHeader(inputMessage, outputMessage);
							((GenericHttpMessageConverter<T>) messageConverter).write(returnValue,
									returnValueType, selectedMediaType, outputMessage);
							if (logger.isDebugEnabled()) {
								logger.debug("Written [" + returnValue + "] as \"" +
										selectedMediaType + "\" using [" + messageConverter + "]");
							}
						}
						return;
					}
				}
				else if (messageConverter.canWrite(returnValueClass, selectedMediaType)) {
				    //ResponseAdvice前置处理
					returnValue = (T) getAdvice().beforeBodyWrite(returnValue, returnType, selectedMediaType,
							(Class<? extends HttpMessageConverter<?>>) messageConverter.getClass(),
							inputMessage, outputMessage);
					if (returnValue != null) {
					    //添加Content-Disposition响应头
						addContentDispositionHeader(inputMessage, outputMessage);
						//调用转换器
						((HttpMessageConverter<T>) messageConverter).write(returnValue,
								selectedMediaType, outputMessage);
						if (logger.isDebugEnabled()) {
							logger.debug("Written [" + returnValue + "] as \"" +
									selectedMediaType + "\" using [" + messageConverter + "]");
						}
					}
					return;
				}
			}
		}
        //如果能够到达这里，那么直接抛错
		if (returnValue != null) {
			throw new HttpMediaTypeNotAcceptableException(this.allSupportedMediaTypes);
		}
	}
	
```
假设我们使用到转换器是MappingJackson2HttpMessageConverter

```
public final void org.springframework.http.converter.AbstractGenericHttpMessageConverter.write(final T t, final Type type, MediaType contentType, HttpOutputMessage outputMessage)
			throws IOException, HttpMessageNotWritableException {
        //获取响应头
		final HttpHeaders headers = outputMessage.getHeaders();
		//添加返回媒体类型响应头
		addDefaultHeaders(headers, t, contentType);

        //将返回值设置到响应流中
		if (outputMessage instanceof StreamingHttpOutputMessage) {
			StreamingHttpOutputMessage streamingOutputMessage = (StreamingHttpOutputMessage) outputMessage;
			streamingOutputMessage.setBody(new StreamingHttpOutputMessage.Body() {
				@Override
				public void writeTo(final OutputStream outputStream) throws IOException {
				    //这部分内容我们在分析参数解析的消息转换中已经分析过了
					writeInternal(t, type, new HttpOutputMessage() {
						@Override
						public OutputStream getBody() throws IOException {
							return outputStream;
						}
						@Override
						public HttpHeaders getHeaders() {
							return headers;
						}
					});
				}
			});
		}
		else {
		    //这部分内容我们在分析参数解析的消息转换中已经分析过了
			writeInternal(t, type, outputMessage);
			outputMessage.getBody().flush();
		}
	}
```

好了，AbstractHandlerMethodAdapter就到此为止了。


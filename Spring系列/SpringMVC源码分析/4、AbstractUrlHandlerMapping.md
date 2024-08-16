首先我们来看到AbstractUrlHandlerMapping其中的一个实现类BeanNameUrlHandlerMapping
的类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/04427cdfc252367afcfb4e9b88f4dff9.png)

可以看到AbstractHandlerMapping及以上都和AbstractHandlerMethodMapping差不多,就不过多的说明了


```
public void org.springframework.web.servlet.handler.AbstractDetectingUrlHandlerMapping.initApplicationContext() throws ApplicationContextException {
        //(*1*)
		super.initApplicationContext();
		//装配handler
		detectHandlers();
	}

	
	//(*1*)
	protected void org.springframework.web.servlet.handler.AbstractHandlerMapping.initApplicationContext() throws BeansException {
	    //扩展拦截器，可以覆盖原有的拦截器
		extendInterceptors(this.interceptors);
		//从容器中自动装配拦截器
		detectMappedInterceptors(this.adaptedInterceptors);
		initInterceptors();
	}
```

装配handler

```
protected void detectHandlers() throws BeansException {
		if (logger.isDebugEnabled()) {
			logger.debug("Looking for URL mappings in application context: " + getApplicationContext());
		}
		//获取所有的beanName
		String[] beanNames = (this.detectHandlersInAncestorContexts ?
				BeanFactoryUtils.beanNamesForTypeIncludingAncestors(getApplicationContext(), Object.class) :
				getApplicationContext().getBeanNamesForType(Object.class));

		// Take any bean name that we can determine URLs for.
		for (String beanName : beanNames) {
		    //获取url
			String[] urls = determineUrlsForHandler(beanName);
			if (!ObjectUtils.isEmpty(urls)) {
				// URL paths found: Let's consider it a handler.
				//注册
				registerHandler(urls, beanName);
			}
			else {
				if (logger.isDebugEnabled()) {
					logger.debug("Rejected bean name '" + beanName + "': no URL paths identified");
				}
			}
		}
	}
```

determineUrlsForHandler

```
protected String[] org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping.determineUrlsForHandler(String beanName) {
		List<String> urls = new ArrayList<String>();
		//beanName如果以/开头，那么表示请求路径
		if (beanName.startsWith("/")) {
			urls.add(beanName);
		}
		//获取别名
		String[] aliases = getApplicationContext().getAliases(beanName);
		for (String alias : aliases) {
			if (alias.startsWith("/")) {
				urls.add(alias);
			}
		}
		return StringUtils.toStringArray(urls);
	}
```
registerHandler

```
protected void org.springframework.web.servlet.handler.AbstractUrlHandlerMapping.registerHandler(String[] urlPaths, String beanName) throws BeansException, IllegalStateException {
		Assert.notNull(urlPaths, "URL path array must not be null");
		for (String urlPath : urlPaths) {
		    //(*1*)
			registerHandler(urlPath, beanName);
		}
	}
	
	 //(*1*)
	 protected void registerHandler(String urlPath, Object handler) throws BeansException, IllegalStateException {
		Assert.notNull(urlPath, "URL path must not be null");
		Assert.notNull(handler, "Handler object must not be null");
		Object resolvedHandler = handler;

		// Eagerly resolve handler if referencing singleton via name.
		if (!this.lazyInitHandlers && handler instanceof String) {
			String handlerName = (String) handler;
			if (getApplicationContext().isSingleton(handlerName)) {
			    //从容器中获取
				resolvedHandler = getApplicationContext().getBean(handlerName);
			}
		}

		Object mappedHandler = this.handlerMap.get(urlPath);
		if (mappedHandler != null) {
		    //发生歧义
			if (mappedHandler != resolvedHandler) {
				throw new IllegalStateException(
						"Cannot map " + getHandlerDescription(handler) + " to URL path [" + urlPath +
						"]: There is already " + getHandlerDescription(mappedHandler) + " mapped.");
			}
		}
		else {
			if (urlPath.equals("/")) {
				if (logger.isInfoEnabled()) {
					logger.info("Root mapping to " + getHandlerDescription(handler));
				}
				//如果仅仅是一个/，表示根路径，设置根处理器
				setRootHandler(resolvedHandler);
			}
			else if (urlPath.equals("/*")) {
				if (logger.isInfoEnabled()) {
					logger.info("Default mapping to " + getHandlerDescription(handler));
				}
				//如果路径为/*，设置为默认处理器
				setDefaultHandler(resolvedHandler);
			}
			else {
			    //注册map《urlPath， resolvedHandler》
				this.handlerMap.put(urlPath, resolvedHandler);
				if (logger.isInfoEnabled()) {
					logger.info("Mapped URL path [" + urlPath + "] onto " + getHandlerDescription(handler));
				}
			}
		}
	}
```

我们再来看看怎么获取handler

```
protected Object org.springframework.web.servlet.handler.AbstractUrlHandlerMapping.getHandlerInternal(HttpServletRequest request) throws Exception {
        //获取路径
		String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
		//查找handler
		Object handler = lookupHandler(lookupPath, request);
		if (handler == null) {
			// We need to care for the default handler directly, since we need to
			// expose the PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE for it as well.
			Object rawHandler = null;
			//如果是/，那么获取根处理器
			if ("/".equals(lookupPath)) {
				rawHandler = getRootHandler();
			}
			if (rawHandler == null) {
			    //获取默认的
				rawHandler = getDefaultHandler();
			}
			if (rawHandler != null) {
				// Bean name or resolved handler?
				if (rawHandler instanceof String) {
					String handlerName = (String) rawHandler;
					//从容器中获取
					rawHandler = getApplicationContext().getBean(handlerName);
				}
				//校验，空方法
				validateHandler(rawHandler, request);
				//构建HandlerExecutionChain，并添加拦截器，额外会设置一个UriTemplateVariablesHandlerInterceptor，用于
				//暴露uriTemplateVariables到request中
				handler = buildPathExposingHandler(rawHandler, lookupPath, lookupPath, null);
			}
		}
		if (handler != null && logger.isDebugEnabled()) {
			logger.debug("Mapping [" + lookupPath + "] to " + handler);
		}
		else if (handler == null && logger.isTraceEnabled()) {
			logger.trace("No handler mapping found for [" + lookupPath + "]");
		}
		return handler;
	}
```
AbstractUrlHandlerMapping还有其他的实现，我们就分析这个就行了
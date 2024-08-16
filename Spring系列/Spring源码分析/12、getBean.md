<p>
&nbsp;&nbsp;&nbsp;&nbsp;在我们使用spring的时候，我们需要ApplicationContext的时候我们会让某个类实现ApplicationContextAware接口，spring在调用refresh刷新方法的时候会在其内部方法prepareBeanFactory中添加ApplicationContextAwareProcessor后置处理器
，这个后置处理器会在bean的初始化前调用实现了ApplicationContextAware接口的bean。
我们获取这个ApplicationContext一般就是用于getBean，getBean方法最终调用的是AbstractBeanFactory的doGetBean方法，这才是spring真正干活的地方
</p>


```
/**
 * name 用户输入的beanName
 * requiredType 要求的类型
 * args 用于创建bean的工厂方法或构造方法的参数
 * typeCheckOnly 仅仅进行类型检查吗？如果只是用于类型检查，那么不会将当前
 * bean的信息记录到正在创建bean集合中
 */
protected <T> T doGetBean(
			final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
			throws BeansException {
        //去掉&前缀，并将别名转换成原名
		final String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
		/*  {\____/}
         *  ( • - •)
         *  /つ the frist！
         * 从单例缓存中获取bean，包含未完全初始化的bean，具体内容看下面的分* * 析
         */
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			//如果当前bean是FactoryBean，那么需要进行进一步处理
				/*  {\____/}
                 *  ( • - •)
                 *  /つ the second！
                 */
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			//如果不是单例的，那么会在bean创建的时候，把正在创建的bean的beanName放在当前线程上下文中
			//如果发生循环引用，抛错
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			//获取父bean工厂
			BeanFactory parentBeanFactory = getParentBeanFactory();
			//如果当前bean工厂不存在对应的BeanDefinition，那么就到父bean工厂中去寻找
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}
            //如果不仅仅是类型检查，那么标记当前bean正在创建
			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
			    //获取合并的BeanDefinition
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				//确保BeanDefinition它不是抽象的
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				//获取明确指定的依赖
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
				    //明确指定的依赖必须先创建，如果发生循环依赖直接抛错
					for (String dependsOnBean : dependsOn) {
						if (isDependent(beanName, dependsOnBean)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dependsOnBean + "'");
						}
						//将依赖关系注册到集合中
						registerDependentBean(dependsOnBean, beanName);
						//递归调用getBean获取bean对象
						getBean(dependsOnBean);
					}
				}

				// Create bean instance.
				//如果是单例bean，那么通过单例bean的方式创建
				if (mbd.isSingleton()) {
				    
					sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
						@Override
						public Object getObject() throws BeansException {
							try {
							    
							    /*  {\____/}
                                 *  ( • - •)
                                 *  /つ High energy warning！
                                 */
								return createBean(beanName, mbd, args);
							}
							catch (BeansException ex) {
								// Explicitly remove instance from singleton cache: It might have been put there
								// eagerly by the creation process, to allow for circular reference resolution.
								// Also remove any beans that received a temporary reference to the bean.
								destroySingleton(beanName);
								throw ex;
							}
						}
					});
					//这个方法和上面那个一样，用于处理FactoryBean的情况
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
                //如果声明周期是prototype，也就是多例
				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
					    //在当前上下文中标记当前bean正在创建，用于检测循环依赖
						beforePrototypeCreation(beanName);
						//
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
					    //创建完bean，移除当前线程上下文对bean信息的记录
						afterPrototypeCreation(beanName);
					}
					//如果是FactoryBean，那么以工厂Bean的方式处理，否则直接返回
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
				    //如果是其他的生命范围，那么获取其范围名
					String scopeName = mbd.getScope();
					//自定义生命周期是在在刷新方法中的postProcessBeanFactory方法注册的，在web应用中，是通过
					//WebApplicationContextUtils.registerWebApplicationScopes静态方法注册关于web范围的生命周期
					//比如request，session，application
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
					    //在指定范围内获取或创建bean，本节会选择request范围来分析
						Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
							@Override
							public Object getObject() throws BeansException {
								beforePrototypeCreation(beanName);
								try {
									return createBean(beanName, mbd, args);
								}
								finally {
									afterPrototypeCreation(beanName);
								}
							}
						});
						//如果是FactoryBean，会以FactoryBean的方式进行处理，否则返回
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// Check if required type matches the type of the actual bean instance.
		//如果指定的类型和实际的类型存在出入，那么尝试进行转换
		if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
			try {
				return getTypeConverter().convertIfNecessary(bean, requiredType);
			}
			catch (TypeMismatchException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to convert bean '" + name + "' to required type [" +
							ClassUtils.getQualifiedName(requiredType) + "]", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		//返回bean
		return (T) bean;
	}
	
	
	/*  {\____/}
     *  ( • - •)
     *  /つ the frist！
     * beanName bean的名称
     * allowEarlyReference 是否允许早期依赖，用于循环依赖
     */
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
	    //从单例集合中获取
		Object singletonObject = this.singletonObjects.get(beanName);
		//如果么有获取到，判断当前beanName是否已经正在创建了，应为只有正在创建中的bean才可能产生早期对象
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
		    //同步获取，避免由于另一线程导致的不一致行为，比如另一线成刚好使用工厂对象创建了bean，这边也创建了一个，然后就会导致添加了两个早期
		    //bean到集合中
			synchronized (this.singletonObjects) {
			    //从早期单例缓存中获取bean
				singletonObject = this.earlySingletonObjects.get(beanName);
				//如果没有，并且允许访问早期bean对象，那么就从单例工厂中获取对应bean暴露的未完全初始化对象
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						//将未初始化完全的早期bean对象存入到早期单例对象缓存中
						this.earlySingletonObjects.put(beanName, singletonObject);
						//移除暴露早期bean对象单例工厂
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return (singletonObject != NULL_OBJECT ? singletonObject : null);
	}
	
	
	/*  {\____/}
     *  ( • - •)
     *  /つ the second！
     * beanInstance bean对象
     * name 用户指定beanName
     * beanName bean的原名
     * mbd bean对应的BeanDefinition
     */
	protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, RootBeanDefinition mbd) {

		// Don't let calling code try to dereference the factory if the bean isn't a factory.
		//如果用户指定的beanName是有&前缀的，那么就说明用户需要获取FactoryBean对象，但是如果获取到的bean不是FactoryBean，
		//那么不好意思，报错
		if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
			throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
		}

		// Now we have the bean instance, which may be a normal bean or a FactoryBean.
		// If it's a FactoryBean, we use it to create a bean instance, unless the
		// caller actually wants a reference to the factory.
		//如果当前bean不是FactoryBean对象或者，它就是以&开头的，那么直接将对象返回
		if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
			return beanInstance;
		}

        //如果顺利通过前面两关，那么说明这真是FactoryBean对象，并且name是不带&的
		Object object = null;
		if (mbd == null) {
		    //从缓存中获取bean的对象
			object = getCachedObjectForFactoryBean(beanName);
		}
		//如果没有，那么只能通过FactoryBean的getObject方法获取对象
		if (object == null) {
			// Return bean instance from factory.
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// Caches object obtained from FactoryBean if it is a singleton.
			if (mbd == null && containsBeanDefinition(beanName)) {
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			boolean synthetic = (mbd != null && mbd.isSynthetic());
			/*  {\____/}
             *  ( • - •)
             *  /つ the three！
             */
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
	
	
	/*  {\____/}
     *  ( • - •)
     *  /つ the three！
     * factory 工厂bean
     * beanName bean的原名
     * shouldPostProcess 是否需要后置处理
     */
	protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
	    //如果是单例的走这
		if (factory.isSingleton() && containsSingleton(beanName)) {
			synchronized (getSingletonMutex()) {
			    //首先检查缓存是否已经缓存对应FactoryBean的bean对象
				Object object = this.factoryBeanObjectCache.get(beanName);
				if (object == null) {
				    //调用FactoryBean的getObject方法
					object = doGetObjectFromFactoryBean(factory, beanName);
					// Only post-process and store if not put there already during getObject() call above
					// (e.g. because of circular reference processing triggered by custom getBean calls)
					//spring怀疑用户会在FactoryBean中调用自定义的getBean，然后将创建的bean存到了factoryBeanObjectCache集合中
					Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
					//如果已经存在，那么使用已经存在的，然后返回
					if (alreadyThere != null) {
						object = alreadyThere;
					}
					//如果不是spring自动生成的bean，那么会触发初始化后处理
					else {
						if (object != null && shouldPostProcess) {
							try {
								object = postProcessObjectFromFactoryBean(object, beanName);
							}
							catch (Throwable ex) {
								throw new BeanCreationException(beanName,
										"Post-processing of FactoryBean's singleton object failed", ex);
							}
						}
						//加入缓存
						this.factoryBeanObjectCache.put(beanName, (object != null ? object : NULL_OBJECT));
					}
				}
				return (object != NULL_OBJECT ? object : null);
			}
		}
		else {
		    //如果不是单例的，那么直接创建，不进行缓存
			Object object = doGetObjectFromFactoryBean(factory, beanName);
			if (object != null && shouldPostProcess) {
				try {
					object = postProcessObjectFromFactoryBean(object, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
				}
			}
			return object;
		}
	}
	
	
	/*  {\____/}
     *  ( • - •)
     *  /つ High energy warning！
     */
     protected Object org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
		if (logger.isDebugEnabled()) {
			logger.debug("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		// Make sure bean class is actually resolved at this point, and
		// clone the bean definition in case of a dynamically resolved Class
		// which cannot be stored in the shared merged bean definition.
		//确保String类型的beanClass已经被解析成Class对象
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
		    //克隆一个新的RootBeanDefinition对象
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
		try {
		    //用于检查配置中设置的look-mehtod与replace-method是否存在，并且如果只有唯一个方法的话，那么将会
		    //设置MethodOverrides中的方法为未被重载，好处就是spring在对这类bean进行代理的时候就不需要进行方法的匹配了
		    //对应的cglib代理拦截分别为LookupOverrideMethodInterceptor，ReplaceOverrideMethodInterceptor
		    //具体的逻辑本节不做分析，感兴趣的同学可以自行分析。
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			//使用实例化后置前后处理器处理，如果被后置处理器处理了，那么直接返回，跳过spring的创建bean过程
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}
        //开始创建bean
		Object beanInstance = doCreateBean(beanName, mbdToUse, args);
		if (logger.isDebugEnabled()) {
			logger.debug("Finished creating instance of bean '" + beanName + "'");
		}
		return beanInstance;
	}
	
	
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;spring在真正创建bean之前做了很多的其他的事情，他们要么就是根据范围去缓存中去，要么就是对BeanDefinition做一些准备工作，比如重载方法的解析，依赖的注册等。
</p>


```
protected Object org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		//如果是单例的，那么会到factoryBeanInstanceCache这个集合中找已经缓存的BeanWrapper，这个集合里的内容是我们通过类型匹配的时候
		//创建的，比如获取BeanFactoryProcessor与BeanPostProcessor后置处理的时候我们都会进行类型检查匹配，不知道大家是否还记得
		//在第二小节的时候，spring为了匹配单例FactoryBean中返回的对象类型，会调用
		//org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.getSingletonFactoryBeanForTypeCheck(String, RootBeanDefinition)
	    //的这个方法，这个方法只会实例化一个不完全初始化的对象，用于确认FactoryBean返回的对象类型
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		//如果为空，那么需要重新创建，这个创建方法内部逻辑请移步第5节bean的实例化
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
		Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
		    //应用mergedBeanDefinition后置处理，用于给用户修改合并BeanDefinition的机会
		    //但是注意它不会影响到缓存的合并BeanDefinition，因为当前BeanDefinition是clone出来的，前面已经分析过了
		    //忘记了的同学往上翻
			if (!mbd.postProcessed) {
				applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		//如果是单例的，并且允许循环依赖并且当前beanName是处于创建中的，那么将只是实例化未被初始化的bean通过ObjectFactory进行
		//提早曝光，存入到单例工厂集合中
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isDebugEnabled()) {
				logger.debug("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			//存入singletonFactories集合中，在上面分析doGetBean方法的时候，我们看到如果单例集合中没有对应的单例bean
			//就会从早期bean对象集合中找，如果没有找到就到单例工厂（singletonFactories）中寻找，如果找到，调用其
			//getObject方法，返回只是实例化而未被填充属性的bean对象
			addSingletonFactory(beanName, new ObjectFactory<Object>() {
				@Override
				public Object getObject() throws BeansException {
					return getEarlyBeanReference(beanName, mbd, bean);
				}
			});
		}

		// Initialize the bean instance.
		//初始化bean实例
		Object exposedObject = bean;
		try {
		    //初始化第一步：填充属性
			populateBean(beanName, mbd, instanceWrapper);
			if (exposedObject != null) {
			    //初始化第二步：首先处理Aware接口，然后调用BeanPostProcessor初始化前置处理，最后如果实现了InitializingBean，调用afterPropertiesSet方法
			    //如果配置时定义初始化方法，那么反射调用，初始化完调用初始化后置处理
				exposedObject = initializeBean(beanName, exposedObject, mbd);
			}
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}
        //如果允许早期曝光，那么需要进行循环依赖的版本检查
		if (earlySingletonExposure) {
		    //由于当前完整对象还未返回，此处最多可从早期依赖集合中获取到早期的bean对象版本
			Object earlySingletonReference = getSingleton(beanName, false);
			//如果不为空，那么就是说明从早期依赖集合中获取到了早期的bean对象，那么就需要进行版本的对照了
			if (earlySingletonReference != null) {
			    //如果两者的版本一样，那么曝光对象的引用是谁其实无所谓，但是如果版本发生了变化，这个时候就得检查是否
			    //发生了循环依赖
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				//首先判断是否允许注入早期未完全初始化的bean对象，即使这个bean最后被后置处理器进行包装（比如代理）
				//默认情况下spring是不允许的（你自己想想，如果你注入的是原始的bean，而你这个bean你原本就是想被代理增强的，结果循环依赖后
				//就是去了aop能力，却没有任何提示，你肯定会陷入这个bug无法自拔，甚至找不出原因）
				//其次再判断当前bean是否被其他的bean依赖
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
				    //获取依赖当前bean的beanName
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
					//遍历这些依赖当前bean的bean，查看他们是否正在创建或已创建，如果确实正在创建或已创建，那么说明发生了循环依赖
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
						    //把循环依赖当前bean的beanName添加到实际依赖当前bean的集合中
							actualDependentBeans.add(dependentBean);
						}
					}
					//如果不为空，那么spring不允许循环依赖不同版本的依赖对象
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
		    //注册可销毁的bean
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;当bean创建好后，如果有必要的话进行销毁bean的注册，spring使用DisposableBeanAdapter适配器去兼容其他形式的销毁方式
</p>


```
public org.springframework.beans.factory.support.DisposableBeanAdapter.DisposableBeanAdapter(Object bean, String beanName, RootBeanDefinition beanDefinition,
			List<BeanPostProcessor> postProcessors, AccessControlContext acc) {

		Assert.notNull(bean, "Disposable bean must not be null");
		this.bean = bean;
		this.beanName = beanName;
		//用于判断是否是实现了DisposableBean接口的bean并且其扩展的销毁方法不能是destroy，否则扩展的destroy优先级更高
		this.invokeDisposableBean =
				(this.bean instanceof DisposableBean && !beanDefinition.isExternallyManagedDestroyMethod("destroy"));
		//是否允许非public修饰的方法访问
		this.nonPublicAccessAllowed = beanDefinition.isNonPublicAccessAllowed();
		this.acc = acc;
		//推断销毁方法首先看我们在配置文件中是否指定了销毁方法，如果指定的方法名为(inferred)或者未指定方法名并且实现了Closeable接口
		//并且未实现DisposableBean接口，那么尝试查找是否有close方法或者shutdown方法
		String destroyMethodName = inferDestroyMethodIfNecessary(bean, beanDefinition);
		
		//如果配置的销毁方法不为空并且（未实现DisposableBean接口或者实现了DisposableBean接口但方法名未重复为destroy）并且不是外部管理的
		//销毁方法，这里面的方法名是通过@PreDestroy或者其他形式注册进来的，这些方法使用销毁后置处理器处理
		if (destroyMethodName != null && !(this.invokeDisposableBean && "destroy".equals(destroyMethodName)) &&
				!beanDefinition.isExternallyManagedDestroyMethod(destroyMethodName)) {
			//指定销毁方法
			this.destroyMethodName = destroyMethodName;
			//解析销毁方法为method对象
			this.destroyMethod = determineDestroyMethod();
			//如果未解析到销毁方法
			if (this.destroyMethod == null) {
			    //如果强制需要销毁，抛错
				if (beanDefinition.isEnforceDestroyMethod()) {
					throw new BeanDefinitionValidationException("Couldn't find a destroy method named '" +
							destroyMethodName + "' on bean with name '" + beanName + "'");
				}
			}
			else {
			    //如果销毁方法有参数且超过1一个以上参数，抛错
				Class<?>[] paramTypes = this.destroyMethod.getParameterTypes();
				if (paramTypes.length > 1) {
					throw new BeanDefinitionValidationException("Method '" + destroyMethodName + "' of bean '" +
							beanName + "' has more than one parameter - not supported as destroy method");
				}
				//如果销毁方法有参数，并且参数类型不是boolean类型的，抛错
				else if (paramTypes.length == 1 && boolean.class != paramTypes[0]) {
					throw new BeanDefinitionValidationException("Method '" + destroyMethodName + "' of bean '" +
							beanName + "' has a non-boolean parameter - not supported as destroy method");
				}
			}
		}
		//从注册的beanPostProcessor中筛选出实现了DestructionAwareBeanPostProcessor的后置处理器
		this.beanPostProcessors = filterPostProcessors(postProcessors);
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;销毁适配器的destroy方法
</p>


```
public void destroy() {
		if (!CollectionUtils.isEmpty(this.beanPostProcessors)) {
			for (DestructionAwareBeanPostProcessor processor : this.beanPostProcessors) {
			    //调用销毁后置处理器
				processor.postProcessBeforeDestruction(this.bean, this.beanName);
			}
		}
        //如果是实现了DisposableBean接口的，那么直接调用它的destroy()方法
		if (this.invokeDisposableBean) {
			if (logger.isDebugEnabled()) {
				logger.debug("Invoking destroy() on bean with name '" + this.beanName + "'");
			}
			try {
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
						@Override
						public Object run() throws Exception {
							((DisposableBean) bean).destroy();
							return null;
						}
					}, acc);
				}
				else {
					((DisposableBean) bean).destroy();
				}
			}
			catch (Throwable ex) {
				String msg = "Invocation of destroy method failed on bean with name '" + this.beanName + "'";
				if (logger.isDebugEnabled()) {
					logger.warn(msg, ex);
				}
				else {
					logger.warn(msg + ": " + ex);
				}
			}
		}
        //如果解析的销毁方法不为空，那么反射调用
		if (this.destroyMethod != null) {
			invokeCustomDestroyMethod(this.destroyMethod);
		}
		//如果destroyMethodName不为空，解析出对应method方法，然后反射调用
		else if (this.destroyMethodName != null) {
			Method methodToCall = determineDestroyMethod();
			if (methodToCall != null) {
				invokeCustomDestroyMethod(methodToCall);
			}
		}
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;填充属性
</p>


```
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
        //获取配置的属性值
		PropertyValues pvs = mbd.getPropertyValues();

		if (bw == null) {
		    //如果bean包装为空，但属性值不为空，那么抛错
			if (!pvs.isEmpty()) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
			}
			else {
				// Skip property population phase for null instance.
				//对于空对象，直接返回
				return;
			}
		}

		// Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
		// state of the bean before properties are set. This can be used, for example,
		// to support styles of field injection.
		//用于标识是否继续属性填充
		boolean continueWithPropertyPopulation = true;
        //如果当前bean不是系统生成的，并且存在实例化前后处理器，那么调用后置处理器
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
					    //如果后置处理返回false，那么表示无需spring继续填充属性
						continueWithPropertyPopulation = false;
						break;
					}
				}
			}
		}
        //如果为false，直接返回，不再进行属性填充
		if (!continueWithPropertyPopulation) {
			return;
		}
        //处理通过名字或者类型自动装配
		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
			//克隆一份PropertyValue集合包装成MutablePropertyValues对象
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

			// Add property values based on autowire by name if applicable.
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
			    //通过名字自动装配
			    /*  {\____/}
                 *  ( • - •)
                 *  /つ the first！
                 */
				autowireByName(beanName, mbd, bw, newPvs);
			}

			// Add property values based on autowire by type if applicable.
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
			    //通过类型自动装配
				autowireByType(beanName, mbd, bw, newPvs);
			}
            //获取新的属性值集合
			pvs = newPvs;
		}
        //是否存在实例化前后置处理器
		boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
		//是否需要依赖检查
		boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);
        //属性准备的后置处理器，我们可以在这种后置处理器中对属性值做进一步的处理，并且可以添加属性
		if (hasInstAwareBpps || needsDepCheck) {
		    //过滤调用被排除的依赖，比如容器在refresh方法中设置的忽略依赖接口，这些忽略依赖存在ignoredDependencyTypes集合中
			PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			if (hasInstAwareBpps) {
			    //调用属性后置处理
				for (BeanPostProcessor bp : getBeanPostProcessors()) {
					if (bp instanceof InstantiationAwareBeanPostProcessor) {
						InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
						pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvs == null) {
							return;
						}
					}
				}
			}
			//检查依赖，这里的检查依赖指的是属性是否存在依赖，spring中有三种类型的依赖检查
			//1、ALL 表示所以存在的属性描述在pvs中必须存在，否则抛错
			//2、simple类型 表示只检查简单类型，如果简单类型的属性在pvs中不存在，那么抛错
			//3、非简单类型 表示非简单类型在pvs中不存在，抛错
			if (needsDepCheck) {
				checkDependencies(beanName, mbd, filteredPds, pvs);
			}
		}
        //应用属性到bean中
		applyPropertyValues(beanName, mbd, bw, pvs);
	}

```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;属性的装配有两种方式，一种是根据名字自动装配，另一种是通过类型进行自动装配，首先我们来看下根据名字自动装配
</p>


```
protected void org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.autowireByName(
			String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
        
        //获取非简单类型的属性名
        /*  {\____/}
         *  ( • - •)
         *  /つ the first！
         */
		String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
		//遍历非简单类型的属性名
		for (String propertyName : propertyNames) {
		    //如果包含当前属性名的Bean
			if (containsBean(propertyName)) {
			    //那么获取指定名称的bean
				Object bean = getBean(propertyName);
				//添加属性，如果对应属性名的属性已经存在，那么覆盖设置，在覆盖之前会检查这个属性值是否是可以
				//合并的，如果可以合并，那么就合并，比如属性值为MergedArray之类的，会直接将值add进去
				pvs.add(propertyName, bean);
				//注册依赖关系
				registerDependentBean(propertyName, beanName);
				if (logger.isDebugEnabled()) {
					logger.debug("Added autowiring by name from bean name '" + beanName +
							"' via property '" + propertyName + "' to bean named '" + propertyName + "'");
				}
			}
			else {
			    //如果未找到，打印提示日志
				if (logger.isTraceEnabled()) {
					logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
							"' by name: no matching bean found");
				}
			}
		}
	}
	
	
	/*  {\____/}
     *  ( • - •)
     *  /つ the first！
     */
     protected String[] unsatisfiedNonSimpleProperties(AbstractBeanDefinition mbd, BeanWrapper bw) {
		Set<String> result = new TreeSet<String>();
		//获取配置的属性值
		PropertyValues pvs = mbd.getPropertyValues();
		//获取PropertyDescriptor数组，spring是通过jdk的的Introspector.getBeanInfo(beanClass))获取描述，并继承了PropertyDescriptor
		//其继承的类为GenericTypeAwarePropertyDescriptor
		PropertyDescriptor[] pds = bw.getPropertyDescriptors();
		for (PropertyDescriptor pd : pds) {
		    //如果当前属性描述存在set方法，并且未被排除依赖检查，并且pvs中未配置这个属性并且不是简单的类型，那么
		    //这个属性符合自动装配条件
			if (pd.getWriteMethod() != null && !isExcludedFromDependencyCheck(pd) && !pvs.contains(pd.getName()) &&
					!BeanUtils.isSimpleProperty(pd.getPropertyType())) {
				result.add(pd.getName());
			}
		}
		//返回
		return StringUtils.toStringArray(result);
	}
	

	
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;通过类型自动装配
</p>


```
protected void org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.autowireByType(
			String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
        //获取类型转换器
		TypeConverter converter = getCustomTypeConverter();
		if (converter == null) {
		    //如果没有，使用BeanWrapperImpl
			converter = bw;
		}

		Set<String> autowiredBeanNames = new LinkedHashSet<String>(4);
		//和通过名字自动装配一样，通过jdk的Introspector与BeanInfo获取属性名称
		//但是需要注意的是，我们的属性必须要有getter或者setter方法，才能被jdk的工具类获取到对应的属性描述
		String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
		for (String propertyName : propertyNames) {
			try {
			    //获取对应属性名描述，这里的逻辑会处理诸如list[1][2].name这种嵌入的属性路径
			    //返回的是list[1][2]上的name属性描述
				PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
				// Don't try autowiring by type for type Object: never makes sense,
				// even if it technically is a unsatisfied, non-simple property.
				//Object类型的不会做处理
				if (Object.class != pd.getPropertyType()) {
				    //包装这个属性的setter方法
					MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
					// Do not allow eager init for type matching in case of a prioritized post-processor.
					//实现了PriorityOrdered接口的不会进行早期初始化
					boolean eager = !PriorityOrdered.class.isAssignableFrom(bw.getWrappedClass());
					//创建依赖描述，这个依赖描述用于描述某个参数的类型，位置，定义它的class，嵌入的泛型啊之类的后者是字段
					DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
					//这个方法，我们在分析bean的创建的时候会具体分析
					Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
					if (autowiredArgument != null) {
						pvs.add(propertyName, autowiredArgument);
					}
					//注册依赖的beanName，用于循环依赖检测
					for (String autowiredBeanName : autowiredBeanNames) {
						registerDependentBean(autowiredBeanName, beanName);
						if (logger.isDebugEnabled()) {
							logger.debug("Autowiring by type from bean name '" + beanName + "' via property '" +
									propertyName + "' to bean named '" + autowiredBeanName + "'");
						}
					}
					autowiredBeanNames.clear();
				}
			}
			catch (BeansException ex) {
				throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
			}
		}
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;我们来分析一下BeanWrapperImpl
</p>

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ec3fcb0eabb5350aa90d4222a63310e3.png)

PropertyEditorRegistry 用于注册属性编辑器

PropertyEditorRegistrySupport 提供基本的注册实现，注册默认的编辑器，注册自定义编辑器

TypeConverter 定义类型转换

PropertyAccessor 定义获取属性的类型，设置属性，定义一些标识符，比如[ ] .

ConfigurablePropertyAccessor 设置一些可配置项，比如是否自动创建嵌入的属性，比如list[1].name，如果list为null，那么会自动创建list

TypeConverterSupport 有没有发现，在spring中凡是有support后缀的，基本上就是一些提供基础能力的模板类

AbstractPropertyAccessor 综合三个接口的能力，并且提供一些基本实现

AbstractNestablePropertyAccessor 提供可嵌入属性访问的能力，比如list[1].name,可以访问嵌入的list[1]对象的name属性

BeanWrapper 包装目标对象

BeanWrapperImpl 以上所有接口能力的实现者


<p>
&nbsp;&nbsp;&nbsp;&nbsp;我们定义一个User类，它有个属性为 List<List<User>> list
然后我们写个如下的main方法，测试
</p>


```
public static void main(String[] args) {
		//构建bean包装类
		BeanWrapperImpl beanWrapper = new BeanWrapperImpl();
		//设置被包装的对象
		beanWrapper.setWrappedInstance(new User());
		//设置允许嵌入自增长
		beanWrapper.setAutoGrowNestedPaths(true);
		//获取User对象的name属性描述
		PropertyDescriptor propertyDescriptor = beanWrapper.getPropertyDescriptor("list[1][2].name");
		//打印这个属性的属性名
		System.out.println(propertyDescriptor.getName());
		
	}
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;我们来看下BeanWrapperImpl的getPropertyDescriptor方法
</p>


```
public PropertyDescriptor getPropertyDescriptor(String propertyName) throws InvalidPropertyException {
        //获取嵌入的BeanWrapperImpl，比如list[1][2].name，那么嵌入的BeanWrapperImpl的目标对象为list[1][2]的值，属性为name
		BeanWrapperImpl nestedBw = (BeanWrapperImpl) getPropertyAccessorForPropertyPath(propertyName);
		//获取最后的路径，这里为name
		String finalPath = getFinalPath(nestedBw, propertyName);
		//从缓存的内省结果集获取描述，前面已经提到过，spring获取内省结果集使用的是JDK的Introspector和BeanInfo
		PropertyDescriptor pd = nestedBw.getCachedIntrospectionResults().getPropertyDescriptor(finalPath);
		if (pd == null) {
			throw new InvalidPropertyException(getRootClass(), getNestedPath() + propertyName,
					"No property '" + propertyName + "' found");
		}
		return pd;
	}
	
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;从上面的代码中我们可以看到spring调用了getPropertyAccessorForPropertyPath方法，获取其内嵌的对象访问器包装对象
</p>


```
protected AbstractNestablePropertyAccessor getPropertyAccessorForPropertyPath(String propertyPath) {
        //spring的一个工具类，获取第一个层内嵌路径，比如list[1][2].name,获取第一层内嵌路径list[1][2]
		int pos = PropertyAccessorUtils.getFirstNestedPropertySeparatorIndex(propertyPath);
		// Handle nested properties recursively.
		if (pos > -1) {
		    //list[1][2]
			String nestedProperty = propertyPath.substring(0, pos);
			//name
			String nestedPath = propertyPath.substring(pos + 1);
			//构建list[1][2]内嵌属性访问器
			AbstractNestablePropertyAccessor nestedPa = getNestedPropertyAccessor(nestedProperty);
			//递归获取第二层内嵌属性访问器，里面再递归N层
			return nestedPa.getPropertyAccessorForPropertyPath(nestedPath);
		}
		else {
			return this;
		}
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;构建内嵌属性访问器
</p>


```
private AbstractNestablePropertyAccessor org.springframework.beans.AbstractNestablePropertyAccessor.getNestedPropertyAccessor(String nestedProperty) {
		//构建存储第一层内嵌属性访问器
		if (this.nestedPropertyAccessors == null) {
			this.nestedPropertyAccessors = new HashMap<String, AbstractNestablePropertyAccessor>();
		}
		// Get value of bean property.
		//解析内嵌属性list[1][2],PropertyTokenHolder将属性名与key分开存储
		//actualName = list, canonicalName = list[1][2], keys=[1,2]
		PropertyTokenHolder tokens = getPropertyNameTokens(nestedProperty);
		//list[1][2]
		String canonicalName = tokens.canonicalName;
		//获取list[1][2]的值
		//(*1*)
		Object value = getPropertyValue(tokens);
		if (value == null || (value.getClass() == javaUtilOptionalClass && OptionalUnwrapper.isEmpty(value))) {
			if (isAutoGrowNestedPaths()) {
				value = setDefaultValue(tokens);
			}
			else {
				throw new NullValueInNestedPathException(getRootClass(), this.nestedPath + canonicalName);
			}
		}

		// Lookup cached sub-PropertyAccessor, create new one if not found.
		//从缓存中获取
		AbstractNestablePropertyAccessor nestedPa = this.nestedPropertyAccessors.get(canonicalName);
		//如果没有就构建一个
		if (nestedPa == null || nestedPa.getWrappedInstance() !=
				(value.getClass() == javaUtilOptionalClass ? OptionalUnwrapper.unwrap(value) : value)) {
			if (logger.isTraceEnabled()) {
				logger.trace("Creating new nested " + getClass().getSimpleName() + " for property '" + canonicalName + "'");
			}
			//构建嵌入的属性访问器，并设置嵌入属性访问器当前所在的属性路径，将父属性访问器的默认转换器设置进去
			nestedPa = newNestedPropertyAccessor(value, this.nestedPath + canonicalName + NESTED_PROPERTY_SEPARATOR);
			// Inherit all type-specific PropertyEditors.
			//将当前的一些默认编辑器设置给新的嵌入属性访问器
			copyDefaultEditorsTo(nestedPa);
			//将当前的自定义编辑器设置给新的嵌入属性访问器
			copyCustomEditorsTo(nestedPa, canonicalName);
			this.nestedPropertyAccessors.put(canonicalName, nestedPa);
		}
		else {
			if (logger.isTraceEnabled()) {
				logger.trace("Using cached nested property accessor for property '" + canonicalName + "'");
			}
		}
		return nestedPa;
	}
	
	
	//(*1*)
	protected Object getPropertyValue(PropertyTokenHolder tokens) throws BeansException {
	    //list[1][2]
		String propertyName = tokens.canonicalName;
		//list
		String actualName = tokens.actualName;
		//获取属性处理器，属性访问器的成员内部类，用于获取值，内嵌类型等
		PropertyHandler ph = getLocalPropertyHandler(actualName);
		if (ph == null || !ph.isReadable()) {
			throw new NotReadablePropertyException(getRootClass(), this.nestedPath + propertyName);
		}
		try {
		    //获取list属性值，很明显我们在User对象中的list属性是null
			Object value = ph.getValue();
			//keys不为空，值为[1,2]
			if (tokens.keys != null) {
				if (value == null) {
				    //我们已经设置为了true
					if (isAutoGrowNestedPaths()) {
					    //给list属性初始化
					    //(*2*)
						value = setDefaultValue(tokens.actualName);
					}
					else {
						throw new NullValueInNestedPathException(getRootClass(), this.nestedPath + propertyName,
								"Cannot access indexed value of property referenced in indexed " +
										"property path '" + propertyName + "': returned null");
					}
				}
				//list
				String indexedPropertyName = tokens.actualName;
				// apply indexes and map keys
				//循环keys
				for (int i = 0; i < tokens.keys.length; i++) {
				    //我们的是list
					String key = tokens.keys[i];
					if (value == null) {
						throw new NullValueInNestedPathException(getRootClass(), this.nestedPath + propertyName,
								"Cannot access indexed value of property referenced in indexed " +
										"property path '" + propertyName + "': returned null");
					}
					//如果是数组，那么通过反射创建数组，增长数组的长度
					else if (value.getClass().isArray()) {
						int index = Integer.parseInt(key);
						value = growArrayIfNecessary(value, index, indexedPropertyName);
						//获取对应下标的值
						value = Array.get(value, index);
					}
					//如果是list，那么反射解析泛型类型，构建对象填充，List<List<User>>,构建list对象填充，第二层循环的时候
					//构建User对象填充
					else if (value instanceof List) {
						int index = Integer.parseInt(key);
						List<Object> list = (List<Object>) value;
						growCollectionIfNecessary(list, index, indexedPropertyName, ph, i + 1);
						value = list.get(index);
					}
					//Set集合没法自动增长，比如set中添加User对象，如果这个User对象重写了hashcode和equals
					//那么不管怎么样都只能添加一个对象
					else if (value instanceof Set) {
						// Apply index to Iterator in case of a Set.
						Set<Object> set = (Set<Object>) value;
						int index = Integer.parseInt(key);
						if (index < 0 || index >= set.size()) {
							throw new InvalidPropertyException(getRootClass(), this.nestedPath + propertyName,
									"Cannot get element with index " + index + " from Set of size " +
											set.size() + ", accessed using property path '" + propertyName + "'");
						}
						Iterator<Object> it = set.iterator();
						for (int j = 0; it.hasNext(); j++) {
							Object elem = it.next();
							if (j == index) {
								value = elem;
								break;
							}
						}
					}
					//如果是map
					else if (value instanceof Map) {
						Map<Object, Object> map = (Map<Object, Object>) value;
						//获取属性类型，并转化为map类型，解析它的第一个泛型类型，也就是key类型
						Class<?> mapKeyType = ph.getResolvableType().getNested(i + 1).asMap().resolveGeneric(0);
						// IMPORTANT: Do not pass full property name in here - property editors
						// must not kick in for map keys but rather only for map values.
						//构建类型描述
						TypeDescriptor typeDescriptor = TypeDescriptor.valueOf(mapKeyType);
						//将key转化成目标类型
						Object convertedMapKey = convertIfNecessary(null, null, key, mapKeyType, typeDescriptor);
						//获取对应的key的值，如果这个Map是新创建的，那么这个value肯定是个null,然后抛出异常，GG
						value = map.get(convertedMapKey);
					}
					else {
						throw new InvalidPropertyException(getRootClass(), this.nestedPath + propertyName,
								"Property referenced in indexed property path '" + propertyName +
										"' is neither an array nor a List nor a Set nor a Map; returned value was [" + value + "]");
					}
					//组合下标，比如list[1]
					indexedPropertyName += PROPERTY_KEY_PREFIX + key + PROPERTY_KEY_SUFFIX;
				}
			}
			return value;
		。。。。。。省略catch异常
	}
	
	
	 //(*2*)
	 private Object setDefaultValue(String propertyName) {
		PropertyTokenHolder tokens = new PropertyTokenHolder();
		tokens.actualName = propertyName;
		tokens.canonicalName = propertyName;
		//(*3*)
		return setDefaultValue(tokens);
	}
	
	//(*3*)
	private Object setDefaultValue(PropertyTokenHolder tokens) {
	    //(*4*)
		PropertyValue pv = createDefaultPropertyValue(tokens);
		setPropertyValue(tokens, pv);
		//又开始递归调用了获取属性的方法
		return getPropertyValue(tokens);
	}
	
	//(*4*)
	private PropertyValue createDefaultPropertyValue(PropertyTokenHolder tokens) {
	    //获取对应属性的类型描述
	    //(*5*)
		TypeDescriptor desc = getPropertyTypeDescriptor(tokens.canonicalName);
		Class<?> type = desc.getType();
		if (type == null) {
			throw new NullValueInNestedPathException(getRootClass(), this.nestedPath + tokens.canonicalName,
					"Could not determine property type for auto-growing a default value");
		}
		//构建对象
		//(*6*)
		Object defaultValue = newValue(type, desc, tokens.canonicalName);
		//这个PropertyValue我们在分析BeanDefinition的时候见到过
		return new PropertyValue(tokens.canonicalName, defaultValue);
	}
	
	//(*5*)
	public TypeDescriptor getPropertyTypeDescriptor(String propertyName) throws BeansException {
		try {
		    //获取list的内嵌属性访问器，这个代码我们已经分析过了，内部会递归调用
			AbstractNestablePropertyAccessor nestedPa = getPropertyAccessorForPropertyPath(propertyName);
			//获取最后一个点的属性名，比如list[1][2].name，那么这个finalPath为name，但我们这里的示例，这里则是list
			String finalPath = getFinalPath(nestedPa, propertyName);
			//actualName = name, canonicalName = name,keys = null
			PropertyTokenHolder tokens = getPropertyNameTokens(finalPath);
			//获取list的处理器
			PropertyHandler ph = nestedPa.getLocalPropertyHandler(tokens.actualName);
			if (ph != null) {
			    //如果存在keys，那么获取list的泛型，比如List<List<User>>，那么获取的是User类型，很显然，在我们示例中
			    //这里是list
				if (tokens.keys != null) {
					if (ph.isReadable() || ph.isWritable()) {
						return ph.nested(tokens.keys.length);
					}
				}
				else {
				    //返回list类型的描述
					if (ph.isReadable() || ph.isWritable()) {
						return ph.toTypeDescriptor();
					}
				}
			}
		}
		catch (InvalidPropertyException ex) {
			// Consider as not determinable.
		}
		return null;
	}
	
	
	//(*6*)
	private Object newValue(Class<?> type, TypeDescriptor desc, String name) {
		try {
		    //反射构建数组
			if (type.isArray()) {
				Class<?> componentType = type.getComponentType();
				// TODO - only handles 2-dimensional arrays
				if (componentType.isArray()) {
					Object array = Array.newInstance(componentType, 1);
					Array.set(array, 0, Array.newInstance(componentType.getComponentType(), 0));
					return array;
				}
				else {
					return Array.newInstance(componentType, 0);
				}
			}
			//构建集合
			else if (Collection.class.isAssignableFrom(type)) {
				TypeDescriptor elementDesc = (desc != null ? desc.getElementTypeDescriptor() : null);
				return CollectionFactory.createCollection(type, (elementDesc != null ? elementDesc.getType() : null), 16);
			}
			//构建map
			else if (Map.class.isAssignableFrom(type)) {
				TypeDescriptor keyDesc = (desc != null ? desc.getMapKeyTypeDescriptor() : null);
				return CollectionFactory.createMap(type, (keyDesc != null ? keyDesc.getType() : null), 16);
			}
			else {
			    //使用默认构造器构建对象
				return BeanUtils.instantiate(type);
			}
		}
		。。。。。。省略catch代码
	}
	
	
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;序列图
</p>

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/4485ae3c951d893888c2f87091e76ca2.png)

比如List<LIst<User>> list

（1）输入属性路径为 list[1][2].name，spring的思路：先按点分割，获取到第一层的list[1][2]，然后解析list[1][2]，分割成actualName = list，keys为[1,2]，然后先获取list的值，如果为null，获取其类型构建对象，然后循环keys，获取list[1]的类型，如果是list之类，就是获取其泛型类型，构建对象填充，填充好后，list[1][2]对象就创建好了，也就是我们这个例子中的User对象，然后包装成BeanWrapperImpl

（2）用这个返回的BeanWrapperImpl对象继续递归调用，这次以name为属性继续重复（1）的逻辑，直到没有点这种路径属性为止


<p>
&nbsp;&nbsp;&nbsp;&nbsp;好了，不管是配置的参数也好，自动装配的参数也好，现在都已经准备就绪了，就差将这些属性设置到对象中了
</p>

```
protected void org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
		if (pvs == null || pvs.isEmpty()) {
			return;
		}

		MutablePropertyValues mpvs = null;
		List<PropertyValue> original;

		if (System.getSecurityManager() != null) {
			if (bw instanceof BeanWrapperImpl) {
				((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
			}
		}

		if (pvs instanceof MutablePropertyValues) {
			mpvs = (MutablePropertyValues) pvs;
			//如果是已经被转换过了的属性，直接进行设置，我相信经过上面对属性访问器的分析，大家应该都能够想象它
			//是怎么设置属性的了吧！循环每个属性，获取其属性名，然后获取内嵌属性访问器，然后设置值
			if (mpvs.isConverted()) {
				// Shortcut: use the pre-converted values as-is.
				try {
					bw.setPropertyValues(mpvs);
					return;
				}
				catch (BeansException ex) {
					throw new BeanCreationException(
							mbd.getResourceDescription(), beanName, "Error setting property values", ex);
				}
			}
			//如果没有被转化，那么获取原始的属性值，这里面包括一些配置中的最原始属性
			original = mpvs.getPropertyValueList();
		}
		else {
		    //如果不是MutablePropertyValues，可能只是其他的属性集合实现，就当前代码看，MutablePropertyValues是具有
		    //记录是否被转换能力的
			original = Arrays.asList(pvs.getPropertyValues());
		}
        //后去类型转换器，内部又默认的Conversion，默认属性编辑器，甚至自定义的编辑器，转换器
		TypeConverter converter = getCustomTypeConverter();
		if (converter == null) {
			converter = bw;
		}
		//这个BeanDefinition值解析器，我们将在第5节bean的创建中分析，用于解析一些bean的引用
		BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

		// Create a deep copy, resolving any references for values.
		List<PropertyValue> deepCopy = new ArrayList<PropertyValue>(original.size());
		boolean resolveNecessary = false;
		for (PropertyValue pv : original) {
			if (pv.isConverted()) {
				deepCopy.add(pv);
			}
			else {
				String propertyName = pv.getName();
				Object originalValue = pv.getValue();
				//通过BeanDefinition解析器解析原始值，比如RuntimeBeanReference，会解析成对应的引用对象
				Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
				Object convertedValue = resolvedValue;
				//判断是否有setter方法并且不是嵌套的路径，比如有点的，有中括号的
				boolean convertible = bw.isWritableProperty(propertyName) &&
						!PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
				//如果不是嵌套路径，那么好办，直接可以通过其setter方法获取到它的参数类型，然后转换一下设置进去即可
				if (convertible) {
					convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
				}
				// Possibly store converted value in merged bean definition,
				// in order to avoid re-conversion for every created bean instance.
				//如果解析过后的值和原始值是一样的，无需拷贝直接存入存入到深拷贝集合中
				if (resolvedValue == originalValue) {
					if (convertible) {
						pv.setConvertedValue(convertedValue);
					}
					deepCopy.add(pv);
				}
				//如果不是嵌套路径并且解析后的值和原始值是不同的并且原始值是TypedStringValue类型，不是动态的，不是列表类型
				else if (convertible && originalValue instanceof TypedStringValue &&
						!((TypedStringValue) originalValue).isDynamic() &&
						!(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
					pv.setConvertedValue(convertedValue);
					deepCopy.add(pv);
				}
				else {
				    //表示仍需要解析
					resolveNecessary = true;
					//其他类型进行拷贝，不设置转换值，由转化器进行路径转换
					deepCopy.add(new PropertyValue(pv, convertedValue));
				}
			}
		}
		//如果无需解析，那么直接设置里面的所有属性已被转换
		if (mpvs != null && !resolveNecessary) {
			mpvs.setConverted();
		}

		// Set our (possibly massaged) deep copy.
		try {
		    //设置属性值
			bw.setPropertyValues(new MutablePropertyValues(deepCopy));
		}
		catch (BeansException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Error setting property values", ex);
		}
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;接下来我们就进入bean的创建
</p>










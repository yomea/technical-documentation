<p>
&nbsp;&nbsp;&nbsp;&nbsp;筛选好advisor之后，就可以创建aop了
</p>


```
protected Object org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator.createProxy(
			Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {

        //如果BeanFactory是ConfigurableListableBeanFactory，暴露被代理的对象class
		if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
			AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
		}
        //创建代理工厂
		ProxyFactory proxyFactory = new ProxyFactory();
		//代理工厂与当前类继承相同ProxyConfig类
		//将当前类的一些配置属性copy到代理工厂
		proxyFactory.copyFrom(this);
        //是否已指定为代理目标类
		if (!proxyFactory.isProxyTargetClass()) {
		    //检查BeanDefinition的属性org.springframework.aop.framework.autoproxy.AutoProxyUtils.preserveTargetClass
		    //如果为true，就设置cglib代理
			if (shouldProxyTargetClass(beanClass, beanName)) {
				proxyFactory.setProxyTargetClass(true);
			}
			else {
			    //获取接口，如果没有接口，就proxyFactory.setProxyTargetClass(true)
			    //（*1*）
				evaluateProxyInterfaces(beanClass, proxyFactory);
			}
		}
        //(*2*)
		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
		for (Advisor advisor : advisors) {
		    //将advisor设置到代理工厂中
			proxyFactory.addAdvisor(advisor);
		}
        //设置需要代理的目标对象包装
		proxyFactory.setTargetSource(targetSource);
		//自定义代理工厂，由子类实现
		customizeProxyFactory(proxyFactory);
        //设置代理配置是否被冻结，被冻结后，代理配置不能被改变
		proxyFactory.setFrozen(this.freezeProxy);
		//是否返回预过滤advisor
		if (advisorsPreFiltered()) {
			proxyFactory.setPreFiltered(true);
		}
        //获取代理对象
		return proxyFactory.getProxy(getProxyClassLoader());
	}
	
	
	//（*1*）
	protected void evaluateProxyInterfaces(Class<?> beanClass, ProxyFactory proxyFactory) {
	    //获取当前bean的接口
		Class<?>[] targetInterfaces = ClassUtils.getAllInterfacesForClass(beanClass, getProxyClassLoader());
		boolean hasReasonableProxyInterface = false;
		//排除一些spring用于配置的接口，比如InitializingBean，DisposableBean，DisposableBean
		//groovy.lang.GroovyObject
		for (Class<?> ifc : targetInterfaces) {
			if (!isConfigurationCallbackInterface(ifc) && !isInternalLanguageInterface(ifc) &&
					ifc.getMethods().length > 0) {
				//表示有接口
				hasReasonableProxyInterface = true;
				break;
			}
		}
		//如果有接口，那么添加接口，用于代理
		if (hasReasonableProxyInterface) {
			// Must allow for introductions; can't just set interfaces to the target's interfaces only.
			for (Class<?> ifc : targetInterfaces) {
				proxyFactory.addInterface(ifc);
			}
		}
		else {
		    //如果么有接口，那么设置为cglib代理
			proxyFactory.setProxyTargetClass(true);
		}
	}
	
	//(*2*)
	protected Advisor[] buildAdvisors(String beanName, Object[] specificInterceptors) {
		// Handle prototypes correctly...
		//获取程序设置的advice，如果没有实现MethodInterceptor的，会使用适配器适配
		Advisor[] commonInterceptors = resolveInterceptorNames();

		List<Object> allInterceptors = new ArrayList<Object>();
		if (specificInterceptors != null) {
			allInterceptors.addAll(Arrays.asList(specificInterceptors));
			if (commonInterceptors.length > 0) {
			    //是否首先使用通用拦截
				if (this.applyCommonInterceptorsFirst) {
					allInterceptors.addAll(0, Arrays.asList(commonInterceptors));
				}
				else {
					allInterceptors.addAll(Arrays.asList(commonInterceptors));
				}
			}
		}
		if (logger.isDebugEnabled()) {
			int nrOfCommonInterceptors = commonInterceptors.length;
			int nrOfSpecificInterceptors = (specificInterceptors != null ? specificInterceptors.length : 0);
			logger.debug("Creating implicit proxy for bean '" + beanName + "' with " + nrOfCommonInterceptors +
					" common interceptors and " + nrOfSpecificInterceptors + " specific interceptors");
		}
        
		Advisor[] advisors = new Advisor[allInterceptors.size()];
		for (int i = 0; i < allInterceptors.size(); i++) {
		    //循环检查advisor，如果是advisor直接返回，如果是Advice并且是MethodInterceptor，那么创建DefaultPointcutAdvisor
		    //这个默认的advisor的切点是Pointcut.TRUE,表示来者不拒，如果不是MethodInterceptor，那么查看下是否是
		    //public DefaultAdvisorAdapterRegistry() {
        	//	registerAdvisorAdapter(new MethodBeforeAdviceAdapter());
        	//	registerAdvisorAdapter(new AfterReturningAdviceAdapter());
        	//	registerAdvisorAdapter(new ThrowsAdviceAdapter());
        	//}
        	//MethodBeforeAdvice，AfterReturningAdvice，ThrowsAdvice具有对应的适配器，advisor类型依然使用默认的advisor进行包装
			advisors[i] = this.advisorAdapterRegistry.wrap(allInterceptors.get(i));
		}
		return advisors;
	}
	
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;代理工厂创建代理
</p>


```
public Object org.springframework.aop.framework.ProxyFactory.getProxy(ClassLoader classLoader) {
        //(*1*)(*3*)
		return createAopProxy().getProxy(classLoader);
	}
	
	 //(*1*)
	 protected final synchronized AopProxy createAopProxy() {
		if (!this.active) {
			activate();
		}
		//(*2*)
		//getAopProxyFactory() 返回的对象为DefaultAopProxyFactory对象
		return getAopProxyFactory().createAopProxy(this);
	}
	
	//(*2*)
	public AopProxy org.springframework.aop.framework.DefaultAopProxyFactory.createAopProxy(AdvisedSupport config) throws AopConfigException {
	    //如果有接口或者设置成代理目标对象或者使用优化策略，那么使用cglib代理
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			//cglib代理
			return new ObjenesisCglibAopProxy(config);
		}
		else {
		    //jdk动态代理
			return new JdkDynamicAopProxy(config);
		}
	}
	
	//(*3*)我们调个动态代理来看看
	public Object getProxy(ClassLoader classLoader) {
		if (logger.isDebugEnabled()) {
			logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
		}
		//(*4*)
		Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised);
		//寻找equals和hashCode方法，如果找到标记this.equalsDefined = true;，this.hashCodeDefined = true;
		findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
		//使用JDK动态代理创建
		return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;
</p>

```
public static Class<?>[] completeProxiedInterfaces(AdvisedSupport advised) {
        //从配置中获取类的接口
		Class<?>[] specifiedInterfaces = advised.getProxiedInterfaces();
		//如果没有接口
		if (specifiedInterfaces.length == 0) {
			// No user-specified interfaces: check whether target class is an interface.
			Class<?> targetClass = advised.getTargetClass();
			//如果设置了需要代理的目标类，那么重新获取接口
			if (targetClass != null) {
				if (targetClass.isInterface()) {
					advised.setInterfaces(targetClass);
				}
				else if (Proxy.isProxyClass(targetClass)) {
					advised.setInterfaces(targetClass.getInterfaces());
				}
				specifiedInterfaces = advised.getProxiedInterfaces();
			}
		}
		//接口中是否存在SpringProxy接口，用于标记某个类是否由spring生成的代理，如果没有设置，那么
	    //spring会自动加入这个接口，以便标记某个类是由spring生成的代理类
		boolean addSpringProxy = !advised.isInterfaceProxied(SpringProxy.class);
		//如果没有定义Advised接口，并且没有阻止代理类被转换成Advised，那么spring会自动添加这个接口
		boolean addAdvised = !advised.isOpaque() && !advised.isInterfaceProxied(Advised.class);
		int nonUserIfcCount = 0;
		//扩展代理接口个数
		if (addSpringProxy) {
			nonUserIfcCount++;
		}
		//扩展代理接口个数
		if (addAdvised) {
			nonUserIfcCount++;
		}
		Class<?>[] proxiedInterfaces = new Class<?>[specifiedInterfaces.length + nonUserIfcCount];
		System.arraycopy(specifiedInterfaces, 0, proxiedInterfaces, 0, specifiedInterfaces.length);
		//添加SpringProxy接口
		if (addSpringProxy) {
			proxiedInterfaces[specifiedInterfaces.length] = SpringProxy.class;
		}
		//添加Advised接口
		if (addAdvised) {
			proxiedInterfaces[proxiedInterfaces.length - 1] = Advised.class;
		}
		return proxiedInterfaces;
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;好了，如果创建代理，我们已经看到过了，我现在再来看看实现JDK动态代理的JdkDynamicAopProxy
这个类实现了jdk的InvocationHandler接口，所以我们重点研究一下它的invoke方法
</p>


```
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		MethodInvocation invocation;
		Object oldProxy = null;
		boolean setProxyContext = false;

		TargetSource targetSource = this.advised.targetSource;
		Class<?> targetClass = null;
		Object target = null;

		try {
		    //处理equals方法
			if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
				// The target does not implement the equals(Object) method itself.
				return equals(args[0]);
			}
			//处理hashCode方法
			if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
				// The target does not implement the hashCode() method itself.
				return hashCode();
			}
			//this.advised.opaque如果为false，spring会自动添加Advised接口让代理类实现
			//判断当前方法是否被定义在接口中，并且这个接口是Advised的父接口或者是它本身
			//那么反射调用以this.advised为目标对象的方法，这个this.advised是实现了Advised的对象
			if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
					method.getDeclaringClass().isAssignableFrom(Advised.class)) {
				// Service invocations on ProxyConfig with the proxy config...
				//反射调用，反射的目标对象是this.advised，这个advised对象是ProxyFactory对象，它实现了Advised对象
				return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
			}

			Object retVal;
            //判断是否需要暴露代理类，如果需要，那么将当前代理类保存到当前线程中去
			if (this.advised.exposeProxy) {
				// Make invocation available if necessary.
				oldProxy = AopContext.setCurrentProxy(proxy);
				setProxyContext = true;
			}
            
			// May be null. Get as late as possible to minimize the time we "own" the target,
			// in case it comes from a pool.
			//获取被代理目标对象
			target = targetSource.getTarget();
			if (target != null) {
				targetClass = target.getClass();
			}

			// Get the interception chain for this method.
			//获取当前方法的拦截器链
			List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

			// Check whether we have any advice. If we don't, we can fallback on direct
			// reflective invocation of the target, and avoid creating a MethodInvocation.
			//如果没有适合对应方法的拦截器链，那么直接反射调用
			if (chain.isEmpty()) {
				// We can skip creating a MethodInvocation: just invoke the target directly
				// Note that the final invoker must be an InvokerInterceptor so we know it does
				// nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
				Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
				retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
			}
			else {
				// We need to create a method invocation...
				//创建一个方法调用驱动，用于驱动链条的执行
				invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
				// Proceed to the joinpoint through the interceptor chain.
				//驱动链条的执行
				retVal = invocation.proceed();
			}

			// Massage return value if necessary.
			//获取方法的返回值
			Class<?> returnType = method.getReturnType();
			if (retVal != null && retVal == target && returnType.isInstance(proxy) &&
					!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
				// Special case: it returned "this" and the return type of the method
				// is type-compatible. Note that we can't help if the target sets
				// a reference to itself in another returned object.
				retVal = proxy;
			}
			//如果方法返回为null又不是void类型的，还他娘的是基本类型的，那么抛错
			else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
				throw new AopInvocationException(
						"Null return value from advice does not match primitive return type for: " + method);
			}
			return retVal;
		}
		finally {
		    //一般单例的对象就是静态，那么会重新创建的对象，比如原型声明周期的就不是静态的
			if (target != null && !targetSource.isStatic()) {
				// Must have come from TargetSource.
				//释放目标对象
				targetSource.releaseTarget(target);
			}
			//如果暴露了代理对象，那么将原来的值设置回去，恢复现场
			if (setProxyContext) {
				// Restore old proxy.
				AopContext.setCurrentProxy(oldProxy);
			}
		}
	}
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;拦截器链条的构建
</p>


```
public List<Object> org.springframework.aop.framework.DefaultAdvisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
			Advised config, Method method, Class<?> targetClass) {

		// This is somewhat tricky... We have to process introductions first,
		// but we need to preserve order in the ultimate list.
		List<Object> interceptorList = new ArrayList<Object>(config.getAdvisors().length);
		//获取类的class对象
		Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
		//首先进行引介增强匹配，循环所有的advisor的，如果有引介增强的advisor，那么进行类匹配，如果匹配返回true
		//否则false
		boolean hasIntroductions = hasMatchingIntroductions(config, actualClass);
		//获取Advisor适配注册器，前面我们看到过有些Advice没有实现MethodInterceptor，被包装成了DefaultPointcut，
		//所以需要进行适配成MethodInterceptor
		AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
        
		for (Advisor advisor : config.getAdvisors()) {
		    //切点通知者，使用切点进行aop拦截
			if (advisor instanceof PointcutAdvisor) {
				// Add it conditionally.
				PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
				//如果已经进行了预先过滤，那么无需再进行切点匹配，否则再次进行切点匹配
				if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
				    //获取MethodInterceptor，一般直接从advisor中获取advice即可，但是并不是所有的advice都是已经实现了
				    //MethodInterceptor接口的，比如MethodBeforeAdvice，AfterReturningAdvice，ThrowsAdvice
				    //所以要对这些没有实现MethodInterceptor的advice进行适配（适配器模式）
					MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
					//获取方法匹配器，用于匹配方法
					MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
					//判断当前调用的方法是否符合切点表达式
					if (MethodMatchers.matches(mm, method, actualClass, hasIntroductions)) {
						if (mm.isRuntime()) {
							// Creating a new object instance in the getInterceptors() method
							// isn't a problem as we normally cache created chains.
							for (MethodInterceptor interceptor : interceptors) {
								interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
							}
						}
						else {
							interceptorList.addAll(Arrays.asList(interceptors));
						}
					}
				}
			}
			//如果是引介增强Advisor，那么只需进行类匹配就行
			else if (advisor instanceof IntroductionAdvisor) {
				IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
				if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
					Interceptor[] interceptors = registry.getInterceptors(advisor);
					interceptorList.addAll(Arrays.asList(interceptors));
				}
			}
			else {
			    //其他类型的advisor，和上面后去方法拦截是一样的操作，只是不需要进行匹配，因为这类advisor一般就是
			    //DefaultPointcut，他们是没有进行和任何切点表达式绑定的，默认都是Pointcut.TRUE
				Interceptor[] interceptors = registry.getInterceptors(advisor);
				interceptorList.addAll(Arrays.asList(interceptors));
			}
		}

		return interceptorList;
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;准备好拦截器之后，spring构建了一个驱动器，这个驱动器就是我们在上面invoke方法中看到的ReflectiveMethodInvocation，我们来看下它的proceed方法
</p>


```
public Object proceed() throws Throwable {
		//	We start with an index of -1 and increment early.
		//spring使用一个下标currentInterceptorIndex来表示当前执行到的位置，如果所有的拦截器都被执行了
		//那么可以调用连接点方法了
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			return invokeJoinpoint();
		}
        //获取对应下标拦截器
		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
	
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
			// Evaluate dynamic method matcher here: static part will already have
			// been evaluated and found to match.
			InterceptorAndDynamicMethodMatcher dm =
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
			if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
				return dm.interceptor.invoke(this);
			}
			else {
				// Dynamic matching failed.
				// Skip this interceptor and invoke the next in the chain.
				return proceed();
			}
		}
		else {
			// It's an interceptor, so we just invoke it: The pointcut will have
			// been evaluated statically before this object was constructed.
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
		}
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;我们挑其中一个拦截器来看看spring是怎么进行拦截的，比如AspectJAroundAdvice，这是一个环绕通知
</p>


```
public Object org.springframework.aop.aspectj.AspectJAroundAdvice.invoke(MethodInvocation mi) throws Throwable {
		if (!(mi instanceof ProxyMethodInvocation)) {
			throw new IllegalStateException("MethodInvocation is not a Spring ProxyMethodInvocation: " + mi);
		}
		//包装方法调用信息
		ProxyMethodInvocation pmi = (ProxyMethodInvocation) mi;
		//用于作为参数传递给连接点，内部关联ProxyMethodInvocation，我们在连接点方法中
		//调用它的process方法，即可继续下一个通知的调用
		//用户获取连接点信息，比如获取kind类型，this（代理对象），target（被代理对象）等
		ProceedingJoinPoint pjp = lazyGetProceedingJoinPoint(pmi);
		//JoinPointMatch用于储存匹配信息，参数绑定信息
		JoinPointMatch jpm = getJoinPointMatch(pmi);
		//调用通知方法
		return invokeAdviceMethod(pjp, jpm, null, null);
	}
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;调用通知方法
</p>


```
protected Object invokeAdviceMethod(JoinPoint jp, JoinPointMatch jpMatch, Object returnValue, Throwable t)
			throws Throwable {

		return invokeAdviceMethodWithGivenArgs(argBinding(jp, jpMatch, returnValue, t));
	}
	
	
	protected Object[] argBinding(JoinPoint jp, JoinPointMatch jpMatch, Object returnValue, Throwable ex) {
	    //参数绑定，这个方法我们在分析注解advice获取的时候分析过，此处不再赘述
		calculateArgumentBindings();

		// AMC start
		Object[] adviceInvocationArgs = new Object[this.adviceInvocationArgumentCount];
		int numBound = 0;
        
        //如果存在joinPoint参数，那么将jp设置到指定参数位置
		if (this.joinPointArgumentIndex != -1) {
			adviceInvocationArgs[this.joinPointArgumentIndex] = jp;
			numBound++;
		}
		//StaticPart参数，用于是静态方法（静态连接点）
		else if (this.joinPointStaticPartArgumentIndex != -1) {
			adviceInvocationArgs[this.joinPointStaticPartArgumentIndex] = jp.getStaticPart();
			numBound++;
		}
        
        //如果参数绑定不为空，说明进行了参数绑定，比如我们指定了方法变量名，返回值变量名，异常方法名
		if (!CollectionUtils.isEmpty(this.argumentBindings)) {
			// binding from pointcut match
			//如果连接点匹配信息存在，那么获取从切点中解析的绑定参数信息 execution(public com.zhipin.service.*.*(..) && args(a,b,c))
			if (jpMatch != null) {
			    //获取切点参数
				PointcutParameter[] parameterBindings = jpMatch.getParameterBindings();
				for (PointcutParameter parameter : parameterBindings) {
				    //获取参数名
					String name = parameter.getName();
					//从参数绑定中获取参数的位置
					Integer index = this.argumentBindings.get(name);
					//将绑定的参数值设置到对应方法参数位置
					adviceInvocationArgs[index] = parameter.getBinding();
					numBound++;
				}
			}
			// binding from returning clause
			//设置返回值
			if (this.returningName != null) {
			    //获取返回值变量下标
				Integer index = this.argumentBindings.get(this.returningName);
				//将返回值设置到对应下标位置
				adviceInvocationArgs[index] = returnValue;
				numBound++;
			}
			// binding from thrown exception
			if (this.throwingName != null) {
			    //绑定抛出异常参数值
				Integer index = this.argumentBindings.get(this.throwingName);
				adviceInvocationArgs[index] = ex;
				numBound++;
			}
		}
        //如果绑定的参数的个数和实际的参数个数不相符，那么抛出异常
		if (numBound != this.adviceInvocationArgumentCount) {
			throw new IllegalStateException("Required to bind " + this.adviceInvocationArgumentCount +
					" arguments, but only bound " + numBound + " (JoinPointMatch " +
					(jpMatch == null ? "was NOT" : "WAS") + " bound in invocation)");
		}
        //返回参数列表
		return adviceInvocationArgs;
	}
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;参数准备好，那么就可以调用对应的通知方法了
</p>

```
protected Object invokeAdviceMethodWithGivenArgs(Object[] args) throws Throwable {
		Object[] actualArgs = args;
		//处理没有参数的情况
		if (this.aspectJAdviceMethod.getParameterTypes().length == 0) {
			actualArgs = null;
		}
		try {
		    //你懂的
			ReflectionUtils.makeAccessible(this.aspectJAdviceMethod);
			// TODO AopUtils.invokeJoinpointUsingReflection
			//还记的这个切面实例工厂吗？这个实例工厂我们前面分析过，其实它内部就是调用BeanFactory的getBean方法获取切面对象
			return this.aspectJAdviceMethod.invoke(this.aspectInstanceFactory.getAspectInstance(), actualArgs);
		}
		catch (IllegalArgumentException ex) {
			throw new AopInvocationException("Mismatch on arguments to advice method [" +
					this.aspectJAdviceMethod + "]; pointcut expression [" +
					this.pointcut.getPointcutExpression() + "]", ex);
		}
		catch (InvocationTargetException ex) {
			throw ex.getTargetException();
		}
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;好了，切面方法已经被调用了，可是我们发现这个环绕通知没有调用Joinpoint的proceed方法，对于环绕通知我们通常都需要在通知方法参数上面添加一个ProceedingJoinPoint类型的参数，然后在代码中调用它的proceed方法，它的proceed方法又调用过了驱动器ReflectiveMethodInvocation的proceed方法。
好了，aop就了解到这里，其他的通知，如果同学们感兴趣就自己去研究。
</p>








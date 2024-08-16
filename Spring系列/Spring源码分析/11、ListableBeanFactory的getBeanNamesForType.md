<p>
&nbsp;&nbsp;&nbsp;&nbsp;这个方法我们不止一次的看到，比如在查找BeanFactoryProcessor的时候就见到过，现在查找BeanPostprocessor的时候又见到了，现在我们将它单独拎出来分析下
</p>


```java
//方法签名
//第一个参数type表示要查找的bean的类型
//includeNonSingletons 是否考虑非单例bean
//allowEagerInit 是否允许提早初始化
String[] getBeanNamesForType(Class<?> type, boolean includeNonSingletons, boolean allowEagerInit);


//具体的实现代码，我们从DefaultListableBeanFactory中分析
public String[] getBeanNamesForType(Class<?> type, boolean includeNonSingletons, boolean allowEagerInit) {
        //配置还未被冻结或者类型为null或者不允许早期初始化
		if (!isConfigurationFrozen() || type == null || !allowEagerInit) {
			return doGetBeanNamesForType(ResolvableType.forRawClass(type), includeNonSingletons, allowEagerInit);
		}
		//值得注意的是，不管type是否不为空，allowEagerInit是否为true
		//只要isConfigurationFrozen()为false就一定不会走这里
		//因为isConfigurationFrozen()为false的时候表示BeanDefinition
		//可能还会发生更改和添加，所以不能进行缓存
		//如果允许非单例的bean，那么从保存所有bean的集合中获取，否则从
		//单例bean中获取
		Map<Class<?>, String[]> cache =
				(includeNonSingletons ? this.allBeanNamesByType : this.singletonBeanNamesByType);
		String[] resolvedBeanNames = cache.get(type);
		if (resolvedBeanNames != null) {
			return resolvedBeanNames;
		}
		//如果缓存中没有获取到，那么只能重新获取，获取到之后就存入缓存
		resolvedBeanNames = doGetBeanNamesForType(ResolvableType.forRawClass(type), includeNonSingletons, true);
		if (ClassUtils.isCacheSafe(type, getBeanClassLoader())) {
			cache.put(type, resolvedBeanNames);
		}
		return resolvedBeanNames;
	}

```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;doGetBeanNamesForType
</p>


```java
//DefaultListableBeanFactory
private String[] doGetBeanNamesForType(ResolvableType type, boolean includeNonSingletons, boolean allowEagerInit) {
		List<String> result = new ArrayList<String>();

		// Check all bean definitions.
		for (String beanName : this.beanDefinitionNames) {
			// Only consider bean as eligible if the bean name
			// is not defined as alias for some other bean.
			//如果是别名，跳过（这个集合会保存所有的主beanName，并且不会
			//保存别名，别名由BeanFactory中别名map维护，这里个人认为是一种防御性编程）
			if (!isAlias(beanName)) {
				try {
				    //获取合并的BeanDefinition，合并的BeanDefinition是指
				    //spring整合了父BeanDefinition的属性，将其他类型的
				    //BeanDefinition变成了RootBeanDefintion
					RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
					// Only check bean definition if it is complete.
					//抽象的BeanDefinition是不做考虑，抽象的就是拿来继承的
					//如果允许早期初始化，那么直接短路，进入方法体
					//如果不允许早期初始化
					/**
					 *  不允许早期初始化的情况：
					 *  一、mbd.hasBeanClass() || !mbd.isLazyInit() || isAllowEagerClassLoading()
					 *  1、mbd.hasBeanClass()：这个是用来判断类是否已经被加载器加载，因为我们配置一个 bean 的时候，它的类名不一定是直接写死的，
					 *  可能是通过表达式运算之后得来的
					 *  2、!mbd.isLazyInit()：在 mbd.hasBeanClass() 为 false 的情况下，判断该 bean 是否被标记为可以早期初始化，可以的话进行提前
					 *  加载就没有任何问题
					 *  3、isAllowEagerClassLoading()：表示该 bean 是否允许早期类加载，默认是允许，所以在默认情况下，这个表达式返回true
					 *  二、!requiresEagerInitForType(mbd.getFactoryBeanName())
					 *  1、这个方法用来判断当前工厂类是否为实现了 FactoryBean 接口，如果实现了，那么其对应 factoryMethodName 指的
					 *  是{@link org.springframework.beans.factory.FactoryBean#getObject()} 的方法，那么对于这样的 bean 是必须进行
					 *  早期初始化才能获取其 bean 的类型
					 */	
					if (!mbd.isAbstract() && (allowEagerInit ||
							((mbd.hasBeanClass() || !mbd.isLazyInit() || isAllowEagerClassLoading())) &&
									!requiresEagerInitForType(mbd.getFactoryBeanName()))) {
						// In case of FactoryBean, match object created by FactoryBean.
					    //检查当前 beanName 对应的 bean 是否实现了 FactoryBean
					    boolean isFactoryBean = isFactoryBean(beanName, mbd);
					    // 如果允许早期初始化，那么只要允许返回非单例 bean 或者该 bean 就是单例的情况下就会调用最后的 isTypeMatch 方法
					    // 如果当前 bean 是一个 FactoryBean，那么 isTypeMatch 方法会进行早期实例化，再调用其 getObjectType 方法获取类型
					    // 在不允许早期初始化的情况下，如果当前bean是 FactoryBean，那么它只能在已经被创建的情况下（不能提早初始化）调用 isTypeMatch 进行匹配判断
					    // 否则只能宣告匹配失败，返回false
					    boolean matchFound = (allowEagerInit || !isFactoryBean || containsSingleton(beanName)) &&
					        (includeNonSingletons || isSingleton(beanName)) && isTypeMatch(beanName, type);
					    // 如果用户是要查找 FactoryBean 类型，那么在 matchFound 为 false 的情况下，给 beanName 加上 & 符合，用来匹配 FactoryBean
						if (!matchFound && isFactoryBean) {
					        // In case of FactoryBean, try to match FactoryBean instance itself next.
					        beanName = FACTORY_BEAN_PREFIX + beanName;
					        matchFound = (includeNonSingletons || mbd.isSingleton()) && isTypeMatch(beanName, type);
					    }
					    // 找到便记录到result集合中，等待返回
						if (matchFound) {
					        result.add(beanName);
					    }
					}
				}
				catch (CannotLoadBeanClassException ex) {
					if (allowEagerInit) {
						throw ex;
					}
					// Probably contains a placeholder: let's ignore it for type matching purposes.
					if (this.logger.isDebugEnabled()) {
						this.logger.debug("Ignoring bean class loading failure for bean '" + beanName + "'", ex);
					}
					onSuppressedException(ex);
				}
				catch (BeanDefinitionStoreException ex) {
					if (allowEagerInit) {
						throw ex;
					}
					// Probably contains a placeholder: let's ignore it for type matching purposes.
					if (this.logger.isDebugEnabled()) {
						this.logger.debug("Ignoring unresolvable metadata in bean definition '" + beanName + "'", ex);
					}
					onSuppressedException(ex);
				}
			}
		}

		// Check manually registered singletons too.
		//从单例注册集合中获取，这个单例集合是保存spring内部注入的单例对象
		//它们有特点就是没有BeanDefinition
		for (String beanName : this.manualSingletonNames) {
			try {
				// In case of FactoryBean, match object created by FactoryBean.
				//如果是工厂bean，那么调用其getObjectType去匹配是否符合指定类型
				if (isFactoryBean(beanName)) {
					if ((includeNonSingletons || isSingleton(beanName)) && isTypeMatch(beanName, type)) {
						result.add(beanName);
						// Match found for this bean: do not match FactoryBean itself anymore.
						continue;
					}
					// In case of FactoryBean, try to match FactoryBean itself next.
					beanName = FACTORY_BEAN_PREFIX + beanName;
				}
				//如果没有匹配成功，那么匹配工厂类
				// Match raw bean instance (might be raw FactoryBean).
				if (isTypeMatch(beanName, type)) {
					result.add(beanName);
				}
			}
			catch (NoSuchBeanDefinitionException ex) {
				// Shouldn't happen - probably a result of circular reference resolution...
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to check manually registered singleton with name '" + beanName + "'", ex);
				}
			}
		}

		return StringUtils.toStringArray(result);
	}

```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;在查找指定类型的bean的时候，spring会对BeanDefinition进行合并，以保证某个bean信息是完整的，那么它是怎么合并的呢？
</p>


```java
//(1)
protected RootBeanDefinition getMergedLocalBeanDefinition(String beanName) throws BeansException {
		// Quick check on the concurrent map first, with minimal locking.
		//从缓存中获取
		RootBeanDefinition mbd = this.mergedBeanDefinitions.get(beanName);
		if (mbd != null) {
			return mbd;
		}
		//缓存中未找到，就到BeanFactory中寻找
		return getMergedBeanDefinition(beanName, getBeanDefinition(beanName));
	}
	
	//(2)
	protected RootBeanDefinition getMergedBeanDefinition(
			String beanName, BeanDefinition bd, BeanDefinition containingBd)
			throws BeanDefinitionStoreException {
        
		synchronized (this.mergedBeanDefinitions) {
			RootBeanDefinition mbd = null;

		
		    //如果包含BeanDefinition为空，从缓存中获取合并BeanDefinition
		    //应为被包含的BeanDefinition不会被缓存
			if (containingBd == null) {
				mbd = this.mergedBeanDefinitions.get(beanName);
			}
            
            //如果缓存中没有
			if (mbd == null) {
			    //如果父BeanDefinition为空
				if (bd.getParentName() == null) {
					// Use copy of given root bean definition.
					//克隆
					if (bd instanceof RootBeanDefinition) {
						mbd = ((RootBeanDefinition) bd).cloneBeanDefinition();
					}
					else {
					    //复制对应的属性，创建一个RootBeanDefinition对象
						mbd = new RootBeanDefinition(bd);
					}
				}
				else {
					// Child bean definition: needs to be merged with parent.
					BeanDefinition pbd;
					try {
					    //将别名转换成真实的beanName
						String parentBeanName = transformedBeanName(bd.getParentName());
						//如果当前beanName与父beanName不相同，那么递归调用合并方法
						if (!beanName.equals(parentBeanName)) {
							pbd = getMergedBeanDefinition(parentBeanName);
						}
						//如果相同的beanName，那么认为它来自父容器
						else {
							if (getParentBeanFactory() instanceof ConfigurableBeanFactory) {
								pbd = ((ConfigurableBeanFactory) getParentBeanFactory()).getMergedBeanDefinition(parentBeanName);
							}
							else {
								throw new NoSuchBeanDefinitionException(bd.getParentName(),
										"Parent name '" + bd.getParentName() + "' is equal to bean name '" + beanName +
										"': cannot be resolved without an AbstractBeanFactory parent");
							}
						}
					}
					catch (NoSuchBeanDefinitionException ex) {
						throw new BeanDefinitionStoreException(bd.getResourceDescription(), beanName,
								"Could not resolve parent bean definition '" + bd.getParentName() + "'", ex);
					}
					// Deep copy with overridden values.
					//复制父BeanDefinition的属性构建新的RootBeanDefinition
					mbd = new RootBeanDefinition(pbd);
					//将子BeanDefinition的属性覆盖地设置
					mbd.overrideFrom(bd);
				}

				// Set default singleton scope, if not configured before.
				//如果没有指定scope，那么设置默认的scope为单例
				if (!StringUtils.hasLength(mbd.getScope())) {
					mbd.setScope(RootBeanDefinition.SCOPE_SINGLETON);
				}

				//如果当前bean为被包含bean，它的scope是单例的而其包含bean的scope不是单例的，那么继承包含bean的scope
				if (containingBd != null && !containingBd.isSingleton() && mbd.isSingleton()) {
					mbd.setScope(containingBd.getScope());
				}

				// Only cache the merged bean definition if we're already about to create an
				// instance of the bean, or at least have already created an instance before.
				//如果当前bean不是被包含的bean，那么进行缓存
				//为什么不缓存被包含bean呢？被包含bean换句话，就是别人的属性
				//属性可能会发生改变
				if (containingBd == null && isCacheBeanMetadata()) {
					this.mergedBeanDefinitions.put(beanName, mbd);
				}
			}

			return mbd;
		}
	}
	
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;当我们不允许spring提早初始化bean的时候，我们会设置allowEagerInit为false，当时问题来了，如果当前我们要匹配的bean是一个工厂bean怎么办，没关系，我们加个&前缀表示，我要获取FactoryBean类型的bean，当时如果这个bean的BeanDefinition指定了factoryBeanName，也就是这种
</p>


```xml
<bean id="validatorFactory" class="com.zhipin.validator.ValidatorFactory">
 </bean>
 
 <bean id="requiredValidator" factory-bean="validatorFactory" factory-method="getValidator">
    <constructor-arg value="required"></constructor-arg>  
 </bean>
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;requiredValidator必填校验器由beanId为validatorFactory的工厂创建，调用其getValidator方法创建，这本来没有什么，但是如果这个ValidatorFactory实现了spring的FactoryBean接口，那么意义就不一样了，变成了如下流程：
</p>

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/7e82d33756dc394fb2b16e63551df497.png)

<p>
&nbsp;&nbsp;&nbsp;&nbsp;可以看到，如果个ValidatorFactory实现了spring的FactoryBean接口，调用的是FactoryBean.getObejct()返回实例的getValidator方法
如果想要匹配这个getValidator方法的返回类型，那么就必须初始化ValidatorFactory对象，然后调用getObjectType返回对象类型，然后判断getValidator的返回值类型。那么问题来了，spring是如何判断当前beanName对应的BeanDefinition是个工厂bean呢？
</p>

```java
public boolean org.springframework.beans.factory.support.AbstractBeanFactory.isFactoryBean(String name) throws NoSuchBeanDefinitionException {
        //处理别名，获取原名
		String beanName = transformedBeanName(name);
        //从单例集合singletonObjects中获取缓存的单例对象，第二参数表示不允许早期引用
        //所谓早期引用就是只是创建了一个对象，但填充属性和初始化的bean
		Object beanInstance = getSingleton(beanName, false);
		//如果存在，并且它是个FactoryBean，返回true
		if (beanInstance != null) {
			return (beanInstance instanceof FactoryBean);
		}
		//如果注册了一个null对象，直接返回false
		else if (containsSingleton(beanName)) {
			// null instance registered
			return false;
		}

		// No singleton instance found -> check bean definition.
		//如果当前BeanFactory中没有注册对应的bean，那么到父BeanFactory中寻找
		if (!containsBeanDefinition(beanName) && getParentBeanFactory() instanceof ConfigurableBeanFactory) {
			// No bean definition found in this factory -> delegate to parent.
			return ((ConfigurableBeanFactory) getParentBeanFactory()).isFactoryBean(name);
		}
        //获取合并BeanDefinition进行判断
		return isFactoryBean(beanName, getMergedLocalBeanDefinition(beanName));
	}
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;getMergedLocalBeanDefinition方法，我们在前面已经分析过了，就是为了整合当前bean的所有信息。以下为判断是否为FactoryBean的方法
</p>


```java
//（1）
protected boolean org.springframework.beans.factory.support.AbstractBeanFactory.isFactoryBean(String beanName, RootBeanDefinition mbd) {
        //判断当前bean类型
		Class<?> beanType = predictBeanType(beanName, mbd, FactoryBean.class);
		return (beanType != null && FactoryBean.class.isAssignableFrom(beanType));
	}
	
	
//（2）判断bean类型
protected Class<?> org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.predictBeanType(String beanName, RootBeanDefinition mbd, Class<?>... typesToMatch) {
        //判断beanName对应的类型
		Class<?> targetType = determineTargetType(beanName, mbd, typesToMatch);

		// Apply SmartInstantiationAwareBeanPostProcessors to predict the
		// eventual type after a before-instantiation shortcut.
	    //SmartInstantiationAwareBeanPostProcessor这个后置处理器可以决定bean的类型和用于实例化bean的构造函数
		if (targetType != null && !mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
					SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
					Class<?> predicted = ibp.predictBeanType(targetType, beanName);
					//如果返回的类型不为空，并且如果是要判断是否为FactoryBean，那么会再不符合条件的情况下继续调用下一个后置处理器
					//如果是其他类型，直接返回
					if (predicted != null && (typesToMatch.length != 1 || FactoryBean.class != typesToMatch[0] ||
							FactoryBean.class.isAssignableFrom(predicted))) {
						return predicted;
					}
				}
			}
		}
		return targetType;
	}

//（3）
protected Class<?> determineTargetType(String beanName, RootBeanDefinition mbd, Class<?>... typesToMatch) {
        //获取缓存的类型，如果有，直接返回，没有就需要继续解析
		Class<?> targetType = mbd.getTargetType();
		if (targetType == null) {
		    //判断是否有指定工厂方法，没有就通过加载器加载beanClass，返回类型
		    //这里面如果指定了loadTimeWeaver的临时的加载器，那么会判断是否需要进行增强，如果增强会返回增强后的类对象
		    //这个临时加载器，我们在预备容器时判断是否存在loadTimeWeaver时，设置进来的
		    //如果存在工厂方法，那么需要解析工厂方法的返回值类型
			targetType = (mbd.getFactoryMethodName() != null ? getTypeForFactoryMethod(beanName, mbd, typesToMatch) :
					resolveBeanClass(mbd, beanName, typesToMatch));
			if (ObjectUtils.isEmpty(typesToMatch) || getTempClassLoader() == null) {
				mbd.setTargetType(targetType);
			}
		}
		return targetType;
	}

//（4）从工厂方法中判断类型
protected Class<?> getTypeForFactoryMethod(String beanName, RootBeanDefinition mbd, Class<?>... typesToMatch) {
        //从缓存中获取其属性
		Class<?> preResolved = mbd.resolvedFactoryMethodReturnType;
		if (preResolved != null) {
			return preResolved;
		}

		Class<?> factoryClass;
		boolean isStatic = true;
        
		String factoryBeanName = mbd.getFactoryBeanName();
		if (factoryBeanName != null) {
		    //如果指定的工厂beanName是自己，报错
			if (factoryBeanName.equals(beanName)) {
				throw new BeanDefinitionStoreException(mbd.getResourceDescription(), beanName,
						"factory-bean reference points back to the same bean definition");
			}
			// Check declared factory method return type on factory class.
			//获取factoryBeanName对应的bean的class
			factoryClass = getType(factoryBeanName);
			//如果存在factoryBeanName，那么对应的方法不为静态方法
			isStatic = false;
		}
		else {
			// Check declared factory method return type on bean class.
			//直接解析beanName对应bean的class
			factoryClass = resolveBeanClass(mbd, beanName, typesToMatch);
		}

		if (factoryClass == null) {
			return null;
		}

		// If all factory methods have the same return type, return that type.
		// Can't clearly figure out exact method due to type converting / autowiring!
		Class<?> commonType = null;
		boolean cache = false;
		//获取xml配置时指定的构造参数
		int minNrOfArgs = mbd.getConstructorArgumentValues().getArgumentCount();
		//获取当前工厂类的所有具体方法（当前类->父类->接口的default方法）
		//如果发生方法覆盖，那么取返回类型最具体的那个一个
		//在java中覆盖的方法的返回类型不能比被覆盖方法的返回类型更宽松
		//也就是这个覆盖方法的返回类型要么与父类的被覆盖返回值类型一样
		//要么是其返回值的子类
		Method[] candidates = ReflectionUtils.getUniqueDeclaredMethods(factoryClass);
		for (Method factoryMethod : candidates) {
		    //找到符合当前bean指定的工厂方法的方法
			if (Modifier.isStatic(factoryMethod.getModifiers()) == isStatic &&
					factoryMethod.getName().equals(mbd.getFactoryMethodName()) &&
					factoryMethod.getParameterTypes().length >= minNrOfArgs) {
					//这里可能有人会问为啥是factoryMethod.getParameterTypes().length >= minNrOfArgs
					//因为我们配置方法参数可能没有全部指定，特别是发生注解注入与xml相结合的，哈哈，这就是需求，习惯就好
				// Declared type variables to inspect?
				if (factoryMethod.getTypeParameters().length > 0) {
					try {
						// Fully resolve parameter names and argument values.
						//获取方法的参数类型
						Class<?>[] paramTypes = factoryMethod.getParameterTypes();
						String[] paramNames = null;
						//通过asm解析字节码获取参数名称
						ParameterNameDiscoverer pnd = getParameterNameDiscoverer();
						if (pnd != null) {
							paramNames = pnd.getParameterNames(factoryMethod);
						}
						ConstructorArgumentValues cav = mbd.getConstructorArgumentValues();
						Set<ConstructorArgumentValues.ValueHolder> usedValueHolders =
								new HashSet<ConstructorArgumentValues.ValueHolder>(paramTypes.length);
						Object[] args = new Object[paramTypes.length];
						for (int i = 0; i < args.length; i++) {
						    //获取符合条件的参数，首先根据下标，类型，名字匹配
						    //如果没有匹配到，那么到通用参数集合中获取
						    //根据类型，名称匹配
							ConstructorArgumentValues.ValueHolder valueHolder = cav.getArgumentValue(
									i, paramTypes[i], (paramNames != null ? paramNames[i] : null), usedValueHolders);
							if (valueHolder == null) {
							    //通过参数值进行类型匹配
								valueHolder = cav.getGenericArgumentValue(null, null, usedValueHolders);
							}
							//如果匹配成功，设置对应的参数值，没有匹配到值就是默认的null
							if (valueHolder != null) {
								args[i] = valueHolder.getValue();
								//加入到已被使用集合中，下次不再被重复使用
								usedValueHolders.add(valueHolder);
							}
						}
						//解析返回值，如果存在泛型，那么会通过参数来解析返回类型
						//比如 public <T> T test(T arg1);
						//通过参数arg1参数判断返回的值类型，没有的直接使用反射method.getReturnType()
						//（*1*）
						Class<?> returnType = AutowireUtils.resolveReturnTypeForFactoryMethod(
								factoryMethod, args, getBeanClassLoader());
						if (returnType != null) {
							cache = true;
							//这里和下面的无参方法处理是一样的
							commonType = ClassUtils.determineCommonAncestor(returnType, commonType);
						}
					}
					catch (Throwable ex) {
						if (logger.isDebugEnabled()) {
							logger.debug("Failed to resolve generic return type for factory method: " + ex);
						}
					}
				}
				else {
				    //如果方法参数为空的，走这，这个方法的逻辑很简单就是将当前方法的返回值和上一个方法的返回值进行isAssignableFrom，通俗点就是找爸爸，如果不是父子关系，那么向上追踪
				    //找共同的父级
					commonType = ClassUtils.determineCommonAncestor(factoryMethod.getReturnType(), commonType);
				}
			}
		}

		if (commonType != null) {
			// Clear return type found: all factory methods return same type.
			//缓存解析过的类型
			if (cache) {
				mbd.resolvedFactoryMethodReturnType = commonType;
			}
			return commonType;
		}
		else {
		    //未找到返回null
			// Ambiguous return types found: return null to indicate "not determinable".
			return null;
		}
	}
	
	
	//（*1*）
	AutowireUtils.resolveReturnTypeForFactoryMethod
	public static Class<?> resolveReturnTypeForFactoryMethod(Method method, Object[] args, ClassLoader classLoader) {
		Assert.notNull(method, "Method must not be null");
		Assert.notNull(args, "Argument array must not be null");
		Assert.notNull(classLoader, "ClassLoader must not be null");
        //获取泛型变量参数，比如T
		TypeVariable<Method>[] declaredTypeVariables = method.getTypeParameters();
		//获取参数化返回类型
		Type genericReturnType = method.getGenericReturnType();
		//获取参数化参数类型
		Type[] methodParameterTypes = method.getGenericParameterTypes();
		Assert.isTrue(args.length == methodParameterTypes.length, "Argument array does not match parameter count");

		// Ensure that the type variable (e.g., T) is declared directly on the method
		// itself (e.g., via <T>), not on the enclosing class or interface.
		boolean locallyDeclaredTypeVariableMatchesReturnType = false;
		//判断是否存在参数泛型会返回泛型相同的情况，比如 public <T> T test(T t)
		for (TypeVariable<Method> currentTypeVariable : declaredTypeVariables) {
			if (currentTypeVariable.equals(genericReturnType)) {
				locallyDeclaredTypeVariableMatchesReturnType = true;
				break;
			}
		}
        //如果存在参数泛型与返回泛型一样的，进入此方法，否则直接method.getReturnType
		if (locallyDeclaredTypeVariableMatchesReturnType) {
			for (int i = 0; i < methodParameterTypes.length; i++) {
				Type methodParameterType = methodParameterTypes[i];
				Object arg = args[i];
				//如果找到参数与返回类型一样的
				if (methodParameterType.equals(genericReturnType)) {
				    //如果是TypedStringValue类型的，返回其真实类型，这个类型的对象我们一般在解析property标签的时候会使用
				    //甚至我们自己定义的BeanDefinition，可以设置这种类型的值，最后BeanFactory使用conversion进行转换
					if (arg instanceof TypedStringValue) {
						TypedStringValue typedValue = ((TypedStringValue) arg);
						if (typedValue.hasTargetType()) {
							return typedValue.getTargetType();
						}
						try {
							return typedValue.resolveTargetType(classLoader);
						}
						catch (ClassNotFoundException ex) {
							throw new IllegalStateException("Failed to resolve value type [" +
									typedValue.getTargetTypeName() + "] for factory method argument", ex);
						}
					}
					// Only consider argument type if it is a simple value...
					//如果不是BeanMetadataElement类型的数据，直接返回class类型
					//RuntimeBeanReference，ManagableArray等都是BeanMetadataElement类型的数据
					if (arg != null && !(arg instanceof BeanMetadataElement)) {
						return arg.getClass();
					}
					
					return method.getReturnType();
				}
				//如果是参数化类型，像List<String>这类就属于参数化类型
				else if (methodParameterType instanceof ParameterizedType) {
					ParameterizedType parameterizedType = (ParameterizedType) methodParameterType;
					//获取这是类型，比如List<String>,它会获取到String.class
					Type[] actualTypeArguments = parameterizedType.getActualTypeArguments();
					for (Type typeArg : actualTypeArguments) {
					    //如果存在相等的泛型
						if (typeArg.equals(genericReturnType)) {
						    //Class<T> 这种
							if (arg instanceof Class) {
								return (Class<?>) arg;
							}
							else {
								String className = null;
								//由于这里的参数只是从配置文件中解析过来的，对应的值不一定和参数中的类型一样
								//这里把String类型的值认为是一个className。比如有个工厂方法public <T> test(T t, List<T> list)
								//那么这个String类型值指定的是这个List中T的类型
								if (arg instanceof String) {
									className = (String) arg;
								}
								else if (arg instanceof TypedStringValue) {
									TypedStringValue typedValue = ((TypedStringValue) arg);
									String targetTypeName = typedValue.getTargetTypeName();
									//如果未指定目标类型或者指定了当前这个值是一个class，那么这个值被认为是一个className
									if (targetTypeName == null || Class.class.getName().equals(targetTypeName)) {
										className = typedValue.getValue();
									}
								}
								if (className != null) {
									try {
										return ClassUtils.forName(className, classLoader);
									}
									catch (ClassNotFoundException ex) {
										throw new IllegalStateException("Could not resolve class name [" + arg +
												"] for factory method argument", ex);
									}
								}
								// Consider adding logic to determine the class of the typeArg, if possible.
								// For now, just fall back...
								return method.getReturnType();
							}
						}
					}
				}
			}
		}

		// Fall back...
		return method.getReturnType();
	}
	

```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;如果我们配置的BeanDefinition具有factoryBean属性，那么它会调用getType方法，通过这个方法获取到factorymethod所在的类，getType方法和isTypeMatch方法代码类似，这是getType把获取到的类型进行了返回，而isTypeMatch直接进行了判断。这个两个方法的代码有基本上是差不多的，不知道为什么会被分成两个独立的方法。这里只分析getType就行了，isTypeMatch不再赘述。
</p>


```java
//这个方法会返回getBean(String beanName)的最终类型
public Class<?> org.springframework.beans.factory.support.AbstractBeanFactory.getType(String name) throws NoSuchBeanDefinitionException {
        //如果是&+beanName，那么首先去掉&前缀，然后处理别名
		String beanName = transformedBeanName(name);

		// Check manually registered singletons.
		Object beanInstance = getSingleton(beanName, false);
		if (beanInstance != null) {
		    //如果是工厂bean，并且不是想要获取FactoryBean对象，那么调用getObjectType方法返回对象类型
			if (beanInstance instanceof FactoryBean && !BeanFactoryUtils.isFactoryDereference(name)) {
				return getTypeForFactoryBean((FactoryBean<?>) beanInstance);
			}
			else {
				return beanInstance.getClass();
			}
		}
		//注册了null对象
		else if (containsSingleton(beanName) && !containsBeanDefinition(beanName)) {
			// null instance registered
			return null;
		}

		else {
			// No singleton instance found -> check bean definition.
			//如果本地不包含这个BeanDefinition，那么从父工厂中获取
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// No bean definition found in this factory -> delegate to parent.
				return parentBeanFactory.getType(originalBeanName(name));
			}
            //获取合并的BeanDefinition
			RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);

			//获取被装饰的BeanDefinition
			BeanDefinitionHolder dbd = mbd.getDecoratedDefinition();
			//先排除它是FactoryBean的情况，FactoryBean的情况统一到后面去处理
			if (dbd != null && !BeanFactoryUtils.isFactoryDereference(name)) {
				RootBeanDefinition tbd = getMergedBeanDefinition(dbd.getBeanName(), dbd.getBeanDefinition(), mbd);
				//这个方法我们已经见过了，递归调用
				Class<?> targetClass = predictBeanType(dbd.getBeanName(), tbd);
				//如果获取到了类型，并且不是工厂bean，那么直接返回
				if (targetClass != null && !FactoryBean.class.isAssignableFrom(targetClass)) {
					return targetClass;
				}
			}
            //判断类型，值得注意的是这个方法只会判断到&beanName级别
            //如果不是工厂bean，那么自然就是最终的getBean类型
            //FactoryBean<T> beanType指FactoryBean， 最终类型为T
			Class<?> beanClass = predictBeanType(beanName, mbd);

			// Check bean class whether we're dealing with a FactoryBean.
			//判断当前beanType是否为工厂bean类型，如果是工厂bean类型，那么需要继续处理，获取其最终类型
			if (beanClass != null && FactoryBean.class.isAssignableFrom(beanClass)) {
				if (!BeanFactoryUtils.isFactoryDereference(name)) {
					// If it's a FactoryBean, we want to look at what it creates, not at the factory class.
					return getTypeForFactoryBean(beanName, mbd);
				}
				else {
					return beanClass;
				}
			}
			else {
				return (!BeanFactoryUtils.isFactoryDereference(name) ? beanClass : null);
			}
		}
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;当beanName对应的beanType类型为FactoryBean的时候，我们要获取到最终类型，这里有个概念就是beanType和上面的getType这个方法为啥返回的类型的意思不一样，那到底啥是beanType呢？spring创建bean有两步，第一步创建beanName对应的bean，第二步判断是否是FactoryBean，如果是，那么会调用FactoryBean.getObject获取最终对象，所以这里指的beanType是beanName第一步对应的bean类型，而不是因为FactoryBean导致的最终类型。
注意如果前面指定获取allowEagerInit类型为false,是不会走到这里的，如果为true就可以走到这
</p>


```java
protected Class<?> org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.getTypeForFactoryBean(String beanName, RootBeanDefinition mbd) {
        //方法内部类，本身用于内部类是final的，但是内部状态允许发生改变
		class Holder { Class<?> value = null; }
		final Holder objectType = new Holder();
		String factoryBeanName = mbd.getFactoryBeanName();
		final String factoryMethodName = mbd.getFactoryMethodName();
        
		if (factoryBeanName != null) {
			if (factoryMethodName != null) {
				// Try to obtain the FactoryBean's object type without instantiating it at all.
				BeanDefinition fbDef = getBeanDefinition(factoryBeanName);
				if (fbDef instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) fbDef).hasBeanClass()) {
					// CGLIB subclass methods hide generic parameters; look at the original user class.
					//获取最原始的类，如果是cglib代理，那么获取其父类
					Class<?> fbClass = ClassUtils.getUserClass(((AbstractBeanDefinition) fbDef).getBeanClass());
					// Find the given factory method, taking into account that in the case of
					// @Bean methods, there may be parameters present.
					//循环当前class的所有方法，找到名字一样，返回值为FactoryBean的方法
					//获取其返回值的泛型类型
					ReflectionUtils.doWithMethods(fbClass,
							new ReflectionUtils.MethodCallback() {
								@Override
								public void doWith(Method method) throws IllegalArgumentException, IllegalAccessException {
									if (method.getName().equals(factoryMethodName) &&
											FactoryBean.class.isAssignableFrom(method.getReturnType())) {
										objectType.value = GenericTypeResolver.resolveReturnTypeArgument(method, FactoryBean.class);
									}
								}
							});
					if (objectType.value != null && Object.class != objectType.value) {
						return objectType.value;
					}
				}
			}
			
			//如果没有获取到对应FactoryBean将要返回的类型，也就是用户并没有指定其返回的泛型类型
			//判断当前bean是否已经在创建了，如果已经在创建了，那么说明
			//允许初始化，否则直接返回
			if (!isBeanEligibleForMetadataCaching(factoryBeanName)) {
				return null;
			}
		}
        //创建实例，但不会进行完全的初始化
		FactoryBean<?> fb = (mbd.isSingleton() ?
				getSingletonFactoryBeanForTypeCheck(beanName, mbd) :
				getNonSingletonFactoryBeanForTypeCheck(beanName, mbd));

		if (fb != null) {
			// Try to obtain the FactoryBean's object type from this early stage of the instance.
			objectType.value = getTypeForFactoryBean(fb);
			if (objectType.value != null) {
				return objectType.value;
			}
		}

		// No type found - fall back to full creation of the FactoryBean instance.
		//如果还是没有找到，那么只能进行完全的创建了
		return super.getTypeForFactoryBean(beanName, mbd);
	}
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;上面这段代码为了找到FactoryBean的返回类型真是煞费苦心，如果当前bean存在factory-bean属性，那么先通过方法的返回值的泛型获取类型，如果获取不到，判断当前bean是否正在创建，如果在的话就通过实例化并不完全初始化的方式调用getObjectType获取，如果还是获取不到，就只能通过完全初始化的方式获取了。

既然分析到了这，那么我们继续分析一下它是如果创建工厂bean的实例的，这里涉及到了单例与非单例的初始化，他们的创建过程其实是差不多的，只是非单例的正在初始化状态是保存在当前线程中的，单例的是保存在全局的集合中，所以这里只对单例的做分析，非单例的不再赘述。
</p>


```java
private FactoryBean<?> org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.getSingletonFactoryBeanForTypeCheck(String beanName, RootBeanDefinition mbd) {
		synchronized (getSingletonMutex()) {
		    //从缓存中获取
			BeanWrapper bw = this.factoryBeanInstanceCache.get(beanName);
			if (bw != null) {
				return (FactoryBean<?>) bw.getWrappedInstance();
			}
			//如果正在创建，直接返回null，不允许重复创建，这个时候会走完全创建
			if (isSingletonCurrentlyInCreation(beanName) ||
					(mbd.getFactoryBeanName() != null && isSingletonCurrentlyInCreation(mbd.getFactoryBeanName()))) {
				return null;
			}
			Object instance = null;
			try {
				// Mark this bean as currently in creation, even if just partially.
				//标记当前这个beanName正在创建
				beforeSingletonCreation(beanName);
				// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
				//使用实例化前后后置处理器（InstantiationAwareBeanPostProcessor）处理，一般很少会有人在bean的实例化前搞事情
				instance = resolveBeforeInstantiation(beanName, mbd);
				if (instance == null) {
				    //创建bean实例
					bw = createBeanInstance(beanName, mbd, null);
					instance = bw.getWrappedInstance();
				}
			}
			finally {
				// Finished partial creation of this bean.
				//移除正在创建标记
				afterSingletonCreation(beanName);
			}
			//判断是否为FactoryBean类型，否则直接报错
			FactoryBean<?> fb = getFactoryBean(beanName, instance);
			if (bw != null) {
			    //保存到缓存中，下次实例化对象的时候可以直接获取
				this.factoryBeanInstanceCache.put(beanName, bw);
			}
			return fb;
		}
	}

```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;上面的代码并不复杂，先从缓存中获取对应的包装对象，如果没有找到就要重新创建，创建才是最复杂的地方！由于创建的过程相当复杂，分开成单独的一个小节去分析。下面是判断是否为FactoryBean类型的序列图
</p>

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/91a5a2c20c966240314fdf8f11cdc99c.png)

<p>
&nbsp;&nbsp;&nbsp;&nbsp;首先判断对应BeanDefinition是否存在工厂方法，如果不存在，很好说，直接判断beanClass即可，如果存在工厂方法，那么又得判断是否存在factorybeanname属性，如果不存在，那么这是一个静态工厂，判断其方法的返回类型是否为FactoryBean类型即可，如果存在，那么事情就稍微复杂了，如果对应的factorybeanname是一个普通的类，那么也只需要判断其返回返回值类型即可，但是如果这个factorybeanname是一个实现了FactoryBean接口的类，那么我们需要实例化，然后调用FactoryBean的getObjectType方法获取到对应的对象类型，在判断它的工厂方法的返回类型。
</p>





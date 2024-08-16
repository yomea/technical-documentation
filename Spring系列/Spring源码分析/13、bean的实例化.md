
<p>
&nbsp;&nbsp;&nbsp;&nbsp;第三小节spring为了判断工厂方法返回bean的类型，进行了工厂实例化，并调用工厂方法创建实例。
</p>

```
//(1)
protected BeanWrapper AbstractAutowireCapableBeanFactory.createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
		// Make sure bean class is actually resolved at this point.
		//解析beanClass，将String的beanClass进行表达式处理后加载成Class对象
		Class<?> beanClass = resolveBeanClass(mbd, beanName);
        
        //如果这个class不可见，抛错
		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}
        //如果存在工厂方法，那么使用工厂方法策略去构建
		if (mbd.getFactoryMethodName() != null)  {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		。。。。。。省略部分代码，此部分代码为通过构造器构建的策略，本节主要分析工厂方法bean的创建
	}
	
	//(2)
	protected BeanWrapper instantiateUsingFactoryMethod(
			String beanName, RootBeanDefinition mbd, Object[] explicitArgs) {
        //new出构造器解析器，调用其使用工厂方法实例化方法
		return new ConstructorResolver(this).instantiateUsingFactoryMethod(beanName, mbd, explicitArgs);
	}
	
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;ConstructorResolver.instantiateUsingFactoryMethod(String, RootBeanDefinition, Object[])
方法的第一个参数表示bean的名字，第二个参数为当前bean的合并BeanDefinition，第三个参数表示指定的方法参数，这个方法非常复杂，代码也很长，需要一定的耐心去读。
</p>


```
public BeanWrapper instantiateUsingFactoryMethod(
			final String beanName, final RootBeanDefinition mbd, final Object[] explicitArgs) {
        //创建包装类
		BeanWrapperImpl bw = new BeanWrapperImpl();
		//设置默认的Conversion转换器和propertyEditor
		this.beanFactory.initBeanWrapper(bw);

		Object factoryBean;
		Class<?> factoryClass;
		boolean isStatic;

		String factoryBeanName = mbd.getFactoryBeanName();
		//存在factoryBeanName属性
		if (factoryBeanName != null) {
		    //如果引用了自己，抛错
			if (factoryBeanName.equals(beanName)) {
				throw new BeanDefinitionStoreException(mbd.getResourceDescription(), beanName,
						"factory-bean reference points back to the same bean definition");
			}
			//获取factoryBeanName指定的实例
			factoryBean = this.beanFactory.getBean(factoryBeanName);
			if (factoryBean == null) {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"factory-bean '" + factoryBeanName + "' (or a BeanPostProcessor involved) returned null");
			}
			//已经被创建，抛错
			if (mbd.isSingleton() && this.beanFactory.containsSingleton(beanName)) {
				throw new IllegalStateException("About-to-be-created singleton instance implicitly appeared " +
						"through the creation of the factory bean that its bean definition points to");
			}
			//获取factoryBeanName指向对象的class对象，并将方法修饰设置为不是静态的
			factoryClass = factoryBean.getClass();
			isStatic = false;
		}
		else {
			// It's a static factory method on the bean class.
			//如果没有factorybeanname，那么就是静态工厂方法
			if (!mbd.hasBeanClass()) {
				throw new BeanDefinitionStoreException(mbd.getResourceDescription(), beanName,
						"bean definition declares neither a bean class nor a factory-bean reference");
			}
			factoryBean = null;
			factoryClass = mbd.getBeanClass();
			isStatic = true;
		}

		Method factoryMethodToUse = null;
		ArgumentsHolder argsHolderToUse = null;
		Object[] argsToUse = null;
        //如果指定的参数不为空，那么使用指定的参数，这个参数是由用户传递进来的
        //我们在调用getBean的时候可以指定这个参数
		if (explicitArgs != null) {
			argsToUse = explicitArgs;
		}
		else {
			Object[] argsToResolve = null;
			synchronized (mbd.constructorArgumentLock) {
			    //从缓存中获取已经解析过的方法或构造方法
				factoryMethodToUse = (Method) mbd.resolvedConstructorOrFactoryMethod;
				if (factoryMethodToUse != null && mbd.constructorArgumentsResolved) {
					// Found a cached factory method...
					//获取已经解析过并且已经缓存的构造参数
					argsToUse = mbd.resolvedConstructorArguments;
					if (argsToUse == null) {
					    //如果没有缓存已经解析好的参数，那么从部分准备好的参数中获取
					    //所谓部分解析好就是从最原始的配置参数中筛选好了对应可用的构造（方法）参数
						argsToResolve = mbd.preparedConstructorArguments;
					}
				}
			}
			//如果已经存在准备好的参数，那么进行解析成最终可用的参数
			if (argsToResolve != null) {
				argsToUse = resolvePreparedArguments(beanName, mbd, bw, factoryMethodToUse, argsToResolve);
			}
		}

		if (factoryMethodToUse == null || argsToUse == null) {
			// Need to determine the factory method...
			// Try all methods with this name to see if they match the given arguments.
			//获取原始的class，如果是cglib代理类，那么会返回它的父类
			factoryClass = ClassUtils.getUserClass(factoryClass);
            //获取所有的方法
			Method[] rawCandidates = getCandidateMethods(factoryClass, mbd);
			List<Method> candidateSet = new ArrayList<Method>();
			for (Method candidate : rawCandidates) {
			    //通过名字和是否静态修饰符，筛选出符合条件的工厂方法
				if (Modifier.isStatic(candidate.getModifiers()) == isStatic && mbd.isFactoryMethod(candidate)) {
					candidateSet.add(candidate);
				}
			}
			Method[] candidates = candidateSet.toArray(new Method[candidateSet.size()]);
			//排序，public desc，参数长度 desc
			AutowireUtils.sortFactoryMethods(candidates);

			ConstructorArgumentValues resolvedValues = null;
			//判断是否为构造器装配模式
			boolean autowiring = (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR);
			//类型权重
			int minTypeDiffWeight = Integer.MAX_VALUE;
			//用于保存具有歧义工厂方法对象
			Set<Method> ambiguousFactoryMethods = null;

			int minNrOfArgs;
			//是否存在用户指定参数，存在以用户指定的参数个数作为最小参数个数
			if (explicitArgs != null) {
				minNrOfArgs = explicitArgs.length;
			}
			else {
				// We don't have arguments passed in programmatically, so we need to resolve the
				// arguments specified in the constructor arguments held in the bean definition.
				//从配置中获取构造参数
				ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
				resolvedValues = new ConstructorArgumentValues();
				
				//解析配置的构造参数，使用BeanDefinitionValueResolver进行解析，将解析好的存到resolvedValues中
				minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
			}

			List<Exception> causes = null;

			for (int i = 0; i < candidates.length; i++) {
				Method candidate = candidates[i];
				//获取方法的参数类型
				Class<?>[] paramTypes = candidate.getParameterTypes();

				if (paramTypes.length >= minNrOfArgs) {
					ArgumentsHolder argsHolder;

					if (resolvedValues != null) {
						// Resolved constructor arguments: type conversion and/or autowiring necessary.
						try {
							String[] paramNames = null;
							ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
							if (pnd != null) {
								paramNames = pnd.getParameterNames(candidate);
							}
							//对已经解析过的参数进行类型转换，如果有未找到对应方法参数的配置参数，那么判断是否启用了构造器自动转配
							//如果启用了，使用自动装配去BeanFactory中获取对应的参数
							argsHolder = createArgumentArray(
									beanName, mbd, resolvedValues, bw, paramTypes, paramNames, candidate, autowiring);
						}
						catch (UnsatisfiedDependencyException ex) {
							if (this.beanFactory.logger.isTraceEnabled()) {
								this.beanFactory.logger.trace("Ignoring factory method [" + candidate +
										"] of bean '" + beanName + "': " + ex);
							}
							if (i == candidates.length - 1 && argsHolderToUse == null) {
								if (causes != null) {
									for (Exception cause : causes) {
										this.beanFactory.onSuppressedException(cause);
									}
								}
								throw ex;
							}
							else {
								// Swallow and try next overloaded factory method.
								if (causes == null) {
									causes = new LinkedList<Exception>();
								}
								causes.add(ex);
								continue;
							}
						}
					}

					else {
						// Explicit arguments given -> arguments length must match exactly.
						//精确匹配用户指定的参数个数，否则跳过
						if (paramTypes.length != explicitArgs.length) {
							continue;
						}
						//设置用户指定的参数为
						/**public ArgumentsHolder(Object[] args) {
                    	  *		this.rawArguments = args;
                    	  *  	this.arguments = args;
                    	  *  	this.preparedArguments = args;
                    	  * 	}
                    	  */
						argsHolder = new ArgumentsHolder(explicitArgs);
					}
                   /* 1、宽松模式：将会判断当前方法参数类型与配置的参数值的类型相差多少，循环向上寻找父类，每次加2，直到找到相同的类型或者父类为null,如果定义的方法参
            数类型不是配置参数值类型的父类，那么直接返回Integer的最大值，转换后参数值与未被转换时的参数值进行类型差异性权重对比，返回差异性小的权重值
            
            》》 所以对于空松模式，spring选择参数值的类型越接近于方法参数类型的方法
            
            2、严格模式：转换过类型后的参数值与未进行转换的参数值都与方法的参数类型形成子与父关系的，那么去那种为Integer.MAX_VALUE - 1024
            转换后的参数值出现与方法参数不是子与父关系的，权重为Integer.MAX_VALUE，未转换的参数值出现与方法参数不是子与父关系的
            权重为Integer.MAX_VALUE - 512
            
            》》 所以对于严格模式，转换后的参数与未转换后的参数都能与方法的参数类型形成子与父关系的优先，其次就是转换后的参数值出现与方法参数形成子与父关系
            的，而未转换的参数值出现与方法参数不是子与父关系的优先
            */
					int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
							argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
					// Choose this factory method if it represents the closest match.
					//如果找到比上一次方法的minTypeDiffWeight还要低，那么当前方法被暂时选用
					if (typeDiffWeight < minTypeDiffWeight) {
						factoryMethodToUse = candidate;
						argsHolderToUse = argsHolder;
						argsToUse = argsHolder.arguments;
						minTypeDiffWeight = typeDiffWeight;
						ambiguousFactoryMethods = null;
					}
					// Find out about ambiguity: In case of the same type difference weight
					// for methods with the same number of parameters, collect such candidates
					// and eventually raise an ambiguity exception.
					// However, only perform that check in non-lenient constructor resolution mode,
					// and explicitly ignore overridden methods (with the same parameter signature).
					//如果当前方法的权重不满足，并且严格模式下权重和上一个方法权重是一样的，方法参数个数，类型都一样
					//那么说明出现了两个都满足的方法，这个时候就出现了分歧，不知道选用哪个方法
					else if (factoryMethodToUse != null && typeDiffWeight == minTypeDiffWeight &&
							!mbd.isLenientConstructorResolution() &&
							paramTypes.length == factoryMethodToUse.getParameterTypes().length &&
							!Arrays.equals(paramTypes, factoryMethodToUse.getParameterTypes())) {
						if (ambiguousFactoryMethods == null) {
							ambiguousFactoryMethods = new LinkedHashSet<Method>();
							ambiguousFactoryMethods.add(factoryMethodToUse);
						}
						ambiguousFactoryMethods.add(candidate);
					}
				}
			}
            //如果经过上一轮过程，没有找到任何符合的方法，那么准备报错
			if (factoryMethodToUse == null) {
				List<String> argTypes = new ArrayList<String>(minNrOfArgs);
				if (explicitArgs != null) {
				    //添加参数类型，用于报错
					for (Object arg : explicitArgs) {
						argTypes.add(arg != null ? arg.getClass().getSimpleName() : "null");
					}
				}
				else {
					Set<ValueHolder> valueHolders = new LinkedHashSet<ValueHolder>(resolvedValues.getArgumentCount());
					valueHolders.addAll(resolvedValues.getIndexedArgumentValues().values());
					valueHolders.addAll(resolvedValues.getGenericArgumentValues());
					//使用解析过的参数构建报错信息
					for (ValueHolder value : valueHolders) {
						String argType = (value.getType() != null ? ClassUtils.getShortName(value.getType()) :
								(value.getValue() != null ? value.getValue().getClass().getSimpleName() : "null"));
						argTypes.add(argType);
					}
				}
				String argDesc = StringUtils.collectionToCommaDelimitedString(argTypes);
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"No matching factory method found: " +
						(mbd.getFactoryBeanName() != null ?
							"factory bean '" + mbd.getFactoryBeanName() + "'; " : "") +
						"factory method '" + mbd.getFactoryMethodName() + "(" + argDesc + ")'. " +
						"Check that a method with the specified name " +
						(minNrOfArgs > 0 ? "and arguments " : "") +
						"exists and that it is " +
						(isStatic ? "static" : "non-static") + ".");
			}
			//如果找到了方法，但是你他妈居然是void返回类型，这是哪门子工厂方法，不多说，报错
			else if (void.class == factoryMethodToUse.getReturnType()) {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Invalid factory method '" + mbd.getFactoryMethodName() +
						"': needs to have a non-void return type!");
			}
			//存在有歧义的方法，spring表示蒙圈，我不懂你到底要哪个，需求不明确，不想和说话，并向你抛出异常
			else if (ambiguousFactoryMethods != null) {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Ambiguous factory method matches found in bean '" + beanName + "' " +
						"(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " +
						ambiguousFactoryMethods);
			}
            //将解析好的数据缓存到BeanDefinition中
			if (explicitArgs == null && argsHolderToUse != null) {
				argsHolderToUse.storeCache(mbd, factoryMethodToUse);
			}
		}

		try {
			Object beanInstance;

			if (System.getSecurityManager() != null) {
				final Object fb = factoryBean;
				final Method factoryMethod = factoryMethodToUse;
				final Object[] args = argsToUse;
				beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
					@Override
					public Object run() {
						return beanFactory.getInstantiationStrategy().instantiate(
								mbd, beanName, beanFactory, fb, factoryMethod, args);
					}
				}, beanFactory.getAccessControlContext());
			}
			else {
			    //直接反射调用方法实例化
				beanInstance = this.beanFactory.getInstantiationStrategy().instantiate(
						mbd, beanName, this.beanFactory, factoryBean, factoryMethodToUse, argsToUse);
			}

			if (beanInstance == null) {
				return null;
			}
			bw.setWrappedInstance(beanInstance);
			return bw;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean instantiation via factory method failed", ex);
		}
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;spring对方法参数的解析分为三个阶段，第一阶段是对最原始的xml配置的参数解析，比如解析ref变成RuntimeBeanReference，第二阶段就是对配置参数进行引用替换，比如RuntimeBeanReference被解析成真正引用的那个bean对象，第三阶段就是对参数进行类型转换。第一阶段我们在分析xml的解析的时候已经接触过了，不再赘述。现在我们来看看第二阶段。上面一段代码中有连个地方进行了第二个阶段的解析，他们分别是resolvePreparedArguments方法，resolveConstructorArguments方法，这两个方法实现的功能差不多，我只要分析其中的resolveConstructorArguments方法即可。
</p>


```
private int org.springframework.beans.factory.support.ConstructorResolver.resolveConstructorArguments(String beanName, RootBeanDefinition mbd, BeanWrapper bw,
			ConstructorArgumentValues cargs, ConstructorArgumentValues resolvedValues) {

        //获取类型转换器，自定义转化器一般为空，所以这里返回BeanWrapperImpl这个转换器
		TypeConverter converter = (this.beanFactory.getCustomTypeConverter() != null ?
				this.beanFactory.getCustomTypeConverter() : bw);
		//创建BeanDefinition值解析器
		BeanDefinitionValueResolver valueResolver =
				new BeanDefinitionValueResolver(this.beanFactory, beanName, mbd, converter);

		int minNrOfArgs = cargs.getArgumentCount();

		for (Map.Entry<Integer, ConstructorArgumentValues.ValueHolder> entry : cargs.getIndexedArgumentValues().entrySet()) {
			int index = entry.getKey();
			if (index < 0) {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Invalid constructor argument index: " + index);
			}
			if (index > minNrOfArgs) {
				minNrOfArgs = index + 1;
			}
			ConstructorArgumentValues.ValueHolder valueHolder = entry.getValue();
			//如果有已经转换过的值，那么直接使用
			if (valueHolder.isConverted()) {
				resolvedValues.addIndexedArgumentValue(index, valueHolder);
			}
			else {
			    //进行解析
				Object resolvedValue =
						valueResolver.resolveValueIfNecessary("constructor argument", valueHolder.getValue());
			    //将解析后的值构建一个新的ValueHolder，存到resolvedValues集合中
				ConstructorArgumentValues.ValueHolder resolvedValueHolder =
						new ConstructorArgumentValues.ValueHolder(resolvedValue, valueHolder.getType(), valueHolder.getName());
				resolvedValueHolder.setSource(valueHolder);
				resolvedValues.addIndexedArgumentValue(index, resolvedValueHolder);
			}
		}
        //解析泛化参数，和解析有下标的方法参数一样的逻辑
		for (ConstructorArgumentValues.ValueHolder valueHolder : cargs.getGenericArgumentValues()) {
			if (valueHolder.isConverted()) {
				resolvedValues.addGenericArgumentValue(valueHolder);
			}
			else {
				Object resolvedValue =
						valueResolver.resolveValueIfNecessary("constructor argument", valueHolder.getValue());
				ConstructorArgumentValues.ValueHolder resolvedValueHolder =
						new ConstructorArgumentValues.ValueHolder(resolvedValue, valueHolder.getType(), valueHolder.getName());
				resolvedValueHolder.setSource(valueHolder);
				resolvedValues.addGenericArgumentValue(resolvedValueHolder);
			}
		}

		return minNrOfArgs;
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;我们可以看到spring解析配置参数是通过一个叫BeanDefinitionValueResolver的对象，下面就是其解析方法
</p>


```
//argName 表示参数名，用于打印错误消息
//value 当前原始配置参数值
public Object resolveValueIfNecessary(Object argName, Object value) {
		// We must check each value to see whether it requires a runtime reference
		// to another bean to be resolved.
		//如果是引用类型，那么到BeanFactory中获取
		if (value instanceof RuntimeBeanReference) {
			RuntimeBeanReference ref = (RuntimeBeanReference) value;
			//(*1*)调用getBean
			return resolveReference(argName, ref);
		}
		//如果是idRef配置的id引用，那么只需检验这个id是否存在，然后返回
		else if (value instanceof RuntimeBeanNameReference) {
			String refName = ((RuntimeBeanNameReference) value).getBeanName();
			refName = String.valueOf(doEvaluate(refName));
			if (!this.beanFactory.containsBean(refName)) {
				throw new BeanDefinitionStoreException(
						"Invalid bean name '" + refName + "' in bean reference for " + argName);
			}
			return refName;
		}
		//如果是BeanDefinition持有类型，那么获取BeanDefinition进行创建对象
		else if (value instanceof BeanDefinitionHolder) {
			// Resolve BeanDefinitionHolder: contains BeanDefinition with name and aliases.
			BeanDefinitionHolder bdHolder = (BeanDefinitionHolder) value;
			//(*2*)
			return resolveInnerBean(argName, bdHolder.getBeanName(), bdHolder.getBeanDefinition());
		}
		//如果是BeanDefinition，和上面的BeanDefinitionHolder一样，创建这个内嵌的BeanDefinition对象
		else if (value instanceof BeanDefinition) {
			// Resolve plain BeanDefinition, without contained name: use dummy name.
			BeanDefinition bd = (BeanDefinition) value;
			String innerBeanName = "(inner bean)" + BeanFactoryUtils.GENERATED_BEAN_NAME_SEPARATOR +
					ObjectUtils.getIdentityHexString(bd);
			//(*2*)
			return resolveInnerBean(argName, innerBeanName, bd);
		}
		//如果是数组类型，那么获取其类型，构建指定类型的数组，然后遍历数组元素，调用调用当前resolveValueIfNecessary方法
		else if (value instanceof ManagedArray) {
			// May need to resolve contained runtime references.
			ManagedArray array = (ManagedArray) value;
			Class<?> elementType = array.resolvedElementType;
			if (elementType == null) {
				String elementTypeName = array.getElementTypeName();
				if (StringUtils.hasText(elementTypeName)) {
					try {
						elementType = ClassUtils.forName(elementTypeName, this.beanFactory.getBeanClassLoader());
						array.resolvedElementType = elementType;
					}
					catch (Throwable ex) {
						// Improve the message by showing the context.
						throw new BeanCreationException(
								this.beanDefinition.getResourceDescription(), this.beanName,
								"Error resolving array type for " + argName, ex);
					}
				}
				else {
					elementType = Object.class;
				}
			}
			return resolveManagedArray(argName, (List<?>) value, elementType);
		}
		//处理List，跟Array的差不多
		else if (value instanceof ManagedList) {
			// May need to resolve contained runtime references.
			return resolveManagedList(argName, (List<?>) value);
		}
		//跟处理List差不多
		else if (value instanceof ManagedSet) {
			// May need to resolve contained runtime references.
			return resolveManagedSet(argName, (Set<?>) value);
		}
		//解析map，对key和value分别递归调用当前resolveValueIfNecessary方法
		else if (value instanceof ManagedMap) {
			// May need to resolve contained runtime references.
			return resolveManagedMap(argName, (Map<?, ?>) value);
		}
		//解析properties，properties的key和value为字符串类型
		else if (value instanceof ManagedProperties) {
			Properties original = (Properties) value;
			Properties copy = new Properties();
			for (Map.Entry<Object, Object> propEntry : original.entrySet()) {
				Object propKey = propEntry.getKey();
				Object propValue = propEntry.getValue();
				//如果是TypedStringValue进行spring表达式执行
				if (propKey instanceof TypedStringValue) {
					propKey = evaluate((TypedStringValue) propKey);
				}
				if (propValue instanceof TypedStringValue) {
					propValue = evaluate((TypedStringValue) propValue);
				}
				copy.put(propKey, propValue);
			}
			return copy;
		}
		else if (value instanceof TypedStringValue) {
			// Convert value to target type here.
			TypedStringValue typedStringValue = (TypedStringValue) value;
			Object valueObject = evaluate(typedStringValue);
			try {
			    //进行类型转换
				Class<?> resolvedTargetType = resolveTargetType(typedStringValue);
				if (resolvedTargetType != null) {
					return this.typeConverter.convertIfNecessary(valueObject, resolvedTargetType);
				}
				else {
					return valueObject;
				}
			}
			catch (Throwable ex) {
				// Improve the message by showing the context.
				throw new BeanCreationException(
						this.beanDefinition.getResourceDescription(), this.beanName,
						"Error converting typed String value for " + argName, ex);
			}
		}
		else {
		    //其他类型，用表达式执行后返回
			return evaluate(value);
		}
	}
	
	
	
	//(*1*)
	private Object resolveReference(Object argName, RuntimeBeanReference ref) {
		try {
			String refName = ref.getBeanName();
			//表达式解析
			refName = String.valueOf(doEvaluate(refName));
			//如果指定从父容器中获取，那么只从父容器中获取
			if (ref.isToParent()) {
				if (this.beanFactory.getParentBeanFactory() == null) {
					throw new BeanCreationException(
							this.beanDefinition.getResourceDescription(), this.beanName,
							"Can't resolve reference to bean '" + refName +
							"' in parent factory: no parent factory available");
				}
				return this.beanFactory.getParentBeanFactory().getBean(refName);
			}
			else {
			    //调用getBean获取对象
				Object bean = this.beanFactory.getBean(refName);
				//保存依赖，refName（被依赖）-》Set(主动依赖refName的beanName)
				//this.beanName -》 Set(被依赖refName)
				//用于循环依赖判断
				this.beanFactory.registerDependentBean(refName, this.beanName);
				return bean;
			}
		}
		catch (BeansException ex) {
			throw new BeanCreationException(
					this.beanDefinition.getResourceDescription(), this.beanName,
					"Cannot resolve reference to bean '" + ref.getBeanName() + "' while setting " + argName, ex);
		}
	}
	
	//(*2*)
	private Object resolveInnerBean(Object argName, String innerBeanName, BeanDefinition innerBd) {
		RootBeanDefinition mbd = null;
		try {
		    //获取合并BeanDefinition
			mbd = this.beanFactory.getMergedBeanDefinition(innerBeanName, innerBd, this.beanDefinition);
			// Check given bean name whether it is unique. If not already unique,
			// add counter - increasing the counter until the name is unique.
			String actualInnerBeanName = innerBeanName;
			if (mbd.isSingleton()) {
			    //如果是单例的，会从查看当前名称是否已经被占用，如果被占用，使用技术的方式向上递增  innerBeanName + "#" + 2
				actualInnerBeanName = adaptInnerBeanName(innerBeanName);
			}
			//注册到被包含bean中，顺便设置依赖关系到依赖map集合中
			this.beanFactory.registerContainedBean(actualInnerBeanName, this.beanName);
			// Guarantee initialization of beans that the inner bean depends on.
			//新初始化显示表明依赖的bean
			String[] dependsOn = mbd.getDependsOn();
			if (dependsOn != null) {
				for (String dependsOnBean : dependsOn) {
				    //注册依赖关系
					this.beanFactory.registerDependentBean(dependsOnBean, actualInnerBeanName);
					//创建依赖对象
					this.beanFactory.getBean(dependsOnBean);
				}
			}
			// Actually create the inner bean instance now...
			//创建嵌入的bean
			Object innerBean = this.beanFactory.createBean(actualInnerBeanName, mbd, null);
			if (innerBean instanceof FactoryBean) {
				boolean synthetic = mbd.isSynthetic();
				//如果是工厂bean，还要调用FactoryBean的getObject方法
				return this.beanFactory.getObjectFromFactoryBean(
						(FactoryBean<?>) innerBean, actualInnerBeanName, !synthetic);
			}
			else {
				return innerBean;
			}
		}
		catch (BeansException ex) {
			throw new BeanCreationException(
					this.beanDefinition.getResourceDescription(), this.beanName,
					"Cannot create inner bean '" + innerBeanName + "' " +
					(mbd != null && mbd.getBeanClassName() != null ? "of type [" + mbd.getBeanClassName() + "] " : "") +
					"while setting " + argName, ex);
		}
	}
	
	
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;第二阶段完毕，我们再来看看第三阶段，类型的转换
</p>


```
private ArgumentsHolder org.springframework.beans.factory.support.ConstructorResolver.createArgumentArray(
			String beanName, RootBeanDefinition mbd, ConstructorArgumentValues resolvedValues,
			BeanWrapper bw, Class<?>[] paramTypes, String[] paramNames, Object methodOrCtor,
			boolean autowiring) throws UnsatisfiedDependencyException {
        //方法类型
		String methodType = (methodOrCtor instanceof Constructor ? "constructor" : "factory method");
		//获取转换器
		TypeConverter converter = (this.beanFactory.getCustomTypeConverter() != null ?
				this.beanFactory.getCustomTypeConverter() : bw);
        //参数暂存，用于存储配置参数，解析参数，转换（可用）参数
		ArgumentsHolder args = new ArgumentsHolder(paramTypes.length);
		//用保存转换（可用）参数
		Set<ConstructorArgumentValues.ValueHolder> usedValueHolders =
				new HashSet<ConstructorArgumentValues.ValueHolder>(paramTypes.length);
		Set<String> autowiredBeanNames = new LinkedHashSet<String>(4);
        //循环方法参数
		for (int paramIndex = 0; paramIndex < paramTypes.length; paramIndex++) {
		    //参数类型
			Class<?> paramType = paramTypes[paramIndex];
			//参数名
			String paramName = (paramNames != null ? paramNames[paramIndex] : null);
			// Try to find matching constructor argument value, either indexed or generic.
			//通过下标，参数类型，参数名到下标集合中寻找合适的ValueHolder，如果找不到就通过参数类型，参数名从泛化集合中查找
			ConstructorArgumentValues.ValueHolder valueHolder =
					resolvedValues.getArgumentValue(paramIndex, paramType, paramName, usedValueHolders);
			// If we couldn't find a direct match and are not supposed to autowire,
			// let's try the next generic, untyped argument value as fallback:
			// it could match after type conversion (for example, String -> int).
			//如果找不到，并且不是构造器装配模式，获取剩下还未被使用的泛化参数
			if (valueHolder == null && !autowiring) {
				valueHolder = resolvedValues.getGenericArgumentValue(null, null, usedValueHolders);
			}
			if (valueHolder != null) {
				// We found a potential match - let's give it a try.
				// Do not consider the same value definition multiple times!
				//加入到可用ValueHolder参数集合中
				usedValueHolders.add(valueHolder);
				//获取解析参数
				Object originalValue = valueHolder.getValue();
				Object convertedValue;
				//如果是已经被转换过的参数，获取转换后的参数，设置到预备参数中
				if (valueHolder.isConverted()) {
					convertedValue = valueHolder.getConvertedValue();
					args.preparedArguments[paramIndex] = convertedValue;
				}
				else {
				    //获取最原始参数，也就是配置参数
					ConstructorArgumentValues.ValueHolder sourceHolder =
							(ConstructorArgumentValues.ValueHolder) valueHolder.getSource();
					//获取配置参数
					Object sourceValue = sourceHolder.getValue();
					try {
					    //将解析参数转换成指定类型参数（转换（可用）参数）
						convertedValue = converter.convertIfNecessary(originalValue, paramType,
								MethodParameter.forMethodOrConstructor(methodOrCtor, paramIndex));
						//指定是否需要解析
						args.resolveNecessary = true;
						//将需要解析的配置参数设置到这里
						args.preparedArguments[paramIndex] = sourceValue;
						
					}
					catch (TypeMismatchException ex) {
						throw new UnsatisfiedDependencyException(
								mbd.getResourceDescription(), beanName, paramIndex, paramType,
								"Could not convert " + methodType + " argument value of type [" +
								ObjectUtils.nullSafeClassName(valueHolder.getValue()) +
								"] to required type [" + paramType.getName() + "]: " + ex.getMessage());
					}
				}
				//设置转换后的值
				args.arguments[paramIndex] = convertedValue;
				//设置解析参数
				args.rawArguments[paramIndex] = originalValue;
			}
			else {
				// No explicit match found: we're either supposed to autowire or
				// have to fail creating an argument array for the given constructor.
				//如果没有找到合适的参数，那么需要进行自动装配，如果不是构造参数自动装配的，直接报错
				if (!autowiring) {
					throw new UnsatisfiedDependencyException(
							mbd.getResourceDescription(), beanName, paramIndex, paramType,
							"Ambiguous " + methodType + " argument types - " +
							"did you specify the correct bean references as " + methodType + " arguments?");
				}
				try {
				    //封装成方法参数对象
					MethodParameter param = MethodParameter.forMethodOrConstructor(methodOrCtor, paramIndex);
					//解析自动装配参数
					//(*1*)
					Object autowiredArgument = resolveAutowiredArgument(param, beanName, autowiredBeanNames, converter);
					//设置参数
					args.rawArguments[paramIndex] = autowiredArgument;
					args.arguments[paramIndex] = autowiredArgument;
					args.preparedArguments[paramIndex] = new AutowiredArgumentMarker();
					args.resolveNecessary = true;
				}
				catch (BeansException ex) {
					throw new UnsatisfiedDependencyException(
							mbd.getResourceDescription(), beanName, paramIndex, paramType, ex);
				}
			}
		}

		for (String autowiredBeanName : autowiredBeanNames) {
		    //设置依赖关系
			this.beanFactory.registerDependentBean(autowiredBeanName, beanName);
			if (this.beanFactory.logger.isDebugEnabled()) {
				this.beanFactory.logger.debug("Autowiring by type from bean name '" + beanName +
						"' via " + methodType + " to bean named '" + autowiredBeanName + "'");
			}
		}

		return args;
	}
	
	//(*1*)
    protected Object resolveAutowiredArgument(
			MethodParameter param, String beanName, Set<String> autowiredBeanNames, TypeConverter typeConverter) {
        
		return this.beanFactory.resolveDependency(
				new DependencyDescriptor(param, true), beanName, autowiredBeanNames, typeConverter);
	}	
	
	//依赖描述
	public DependencyDescriptor(MethodParameter methodParameter, boolean required, boolean eager) {
		Assert.notNull(methodParameter, "MethodParameter must not be null");
		
		this.methodParameter = methodParameter;
		//获取当前参数是定义在哪个个类中的
		this.declaringClass = methodParameter.getDeclaringClass();
		//获取当前参数是被包含在哪个类中的，一般和this.declaringClass一样，但是如果这个参数是在父类中的，那么
		//这个包容类是子类
		this.containingClass = methodParameter.getContainingClass();
		//如果是方法，那么设置方法名和方法参数类型
		if (this.methodParameter.getMethod() != null) {
			this.methodName = methodParameter.getMethod().getName();
			this.parameterTypes = methodParameter.getMethod().getParameterTypes();
		}
		else {
			this.parameterTypes = methodParameter.getConstructor().getParameterTypes();
		}
		//参数下标
		this.parameterIndex = methodParameter.getParameterIndex();
		//表示必填
		this.required = required;
		//是否饥饿的,如果是在类型匹配的时候就会发生工厂bean的创建
		this.eager = eager;
	}
	
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;有的时候，我们在配置方法参数的时候，没有指定全，可能有部分参数需要自动装配，下面就是解析自动装配依赖的入口
</p>


```
public Object org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DependencyDescriptor descriptor, String beanName,
			Set<String> autowiredBeanNames, TypeConverter typeConverter) throws BeansException {
        //设置参数名解析器
		descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());
		//如果依赖的类型是java8中的Optional<T>,创建OptionalDependencyFactory，并将原来的依赖描述拷贝一份，内嵌计数加1，因为
		//他是Optional<T>类型，它真正类型是T，内部调用doResolveDependency方法，将这个方法的返回值设置到Optional中
		if (descriptor.getDependencyType().equals(javaUtilOptionalClass)) {
			return new OptionalDependencyFactory().createOptionalDependency(descriptor, beanName);
		}
		//如果依赖类型是对象工厂，在前面我们曾经见到过，spring把Request，Response，session等web对象封装成了ObjectFactory类型
		else if (ObjectFactory.class == descriptor.getDependencyType()) {
		    //返回DependencyObjectFactory对象，它的getObject方法内部调用了DefaultListableBeanFactory的doResolveDependency方法
			return new DependencyObjectFactory(descriptor, beanName);
		}
		//如果是javax.inject.Provider类型的依赖
		else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
		    //返回DependencyProvider对象，其实这个DependencyProvider又继承了DependencyObjectFactory，DependencyProvider
		    //的get方法调用了DependencyObjectFactory的getObject的方法，真正调用doResolveDependency的是getObject方法
			return new DependencyProviderFactory().createDependencyProvider(descriptor, beanName);
		}
		else {
		    //这里的自动装配候选解析器我们在解析@Component时就看到过，spring注入的是ContextAnnotationAutowireCandidateResolver类型
		    //的候选解析器，这里调用getLazyResolutionProxyIfNecessary方法，给参数构建懒加载代理对象
		    //构建懒加载代理的条件就是对应参数上是否有@Lazy注解，并且值为true
		    //(*1*)
			Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(descriptor, beanName);
			if (result == null) {
			    //直接从BeanFactory中获取对象
				result = doResolveDependency(descriptor, beanName, autowiredBeanNames, typeConverter);
			}
			return result;
		}
	}
	
	//(*1*)
	public Object getLazyResolutionProxyIfNecessary(DependencyDescriptor descriptor, String beanName) {
	    //(*2*)
		return (isLazy(descriptor) ? buildLazyResolutionProxy(descriptor, beanName) : null);
	}
	
	//(*2*)
	protected Object buildLazyResolutionProxy(final DependencyDescriptor descriptor, final String beanName) {
		Assert.state(getBeanFactory() instanceof DefaultListableBeanFactory,
				"BeanFactory needs to be a DefaultListableBeanFactory");
		final DefaultListableBeanFactory beanFactory = (DefaultListableBeanFactory) getBeanFactory();
		//包装目标对象
		TargetSource ts = new TargetSource() {
			@Override
			public Class<?> getTargetClass() {
				return descriptor.getDependencyType();
			}
			@Override
			public boolean isStatic() {
				return false;
			}
			@Override
			public Object getTarget() {
			    //动态从BeanFactory中获取对应bean对象
				Object target = beanFactory.doResolveDependency(descriptor, beanName, null, null);
				if (target == null) {
					throw new NoSuchBeanDefinitionException(descriptor.getDependencyType(),
							"Optional dependency not present for lazy injection point");
				}
				return target;
			}
			@Override
			public void releaseTarget(Object target) {
			}
		};
		//构建代理工厂
		ProxyFactory pf = new ProxyFactory();
		//设置targetSource
		pf.setTargetSource(ts);
		//获取依赖类型
		Class<?> dependencyType = descriptor.getDependencyType();
		//如果是接口，那么将会使用动态代理
		if (dependencyType.isInterface()) {
			pf.addInterface(dependencyType);
		}
		//构建代理对象
		return pf.getProxy(beanFactory.getBeanClassLoader());
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;下面是涉及到代理的类图和序列图
</p>

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ba5bc7a66c4291f053c7176e1e0375a0.png)

TargetClassAware 获取被代理类的class对象

Advised 获取advice，advisor，设置代理接口，属于配置类

proxyConfig 代理配置，比如是否使用cglib代理，是否使用优化

AdvisedSupport 除了继承的能力之外，又定义了是否预过滤的概念

ProxyCreatorSupport 增加了通知监听器

ProxyFactory 创建代理

<p>
&nbsp;&nbsp;&nbsp;&nbsp;下面是代理的时序图
</p>

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/325a05c985d79bac8fa3e553c0c66007.png)

首先需要构建TargetSource，表示需要被代理的对象，然后如果有接口，就设置接口，接口用于JDK的动态代理，将当前代理配置作为构造参数去构造AopProxy对象，目前spring中实现的AopProxy代理有ObjenesisCglibAopProxy，JdkDynamicAopProxy，前者用于cglib动态代理，后者用于JDK动态代理


<p>
&nbsp;&nbsp;&nbsp;&nbsp;接下来我们继续分析下spring是怎么到BeanFactory中获取数据的
</p>


```
public Object org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DependencyDescriptor descriptor, String beanName,
			Set<String> autowiredBeanNames, TypeConverter typeConverter) throws BeansException {
        //获取依赖的类型
		Class<?> type = descriptor.getDependencyType();
		//处理@Vlaue注解，比如 Test(@Value("${a}") String a) 或者 @Value Test(String a, String b)
		//@Value定义在方法上且方法参数上没有@Value，表示这个方法的所有参数都被指定@Value
		Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
		if (value != null) {
			if (value instanceof String) {
			    //使用字符串解析器解析占位符，这个占位符解析器我们在分析BeanDefinition属性，类型等占位符解析的时候看到过，他是通过ConfigurableListableBeanFactory的
			    //addEmbeddedValueResolver加进去的
				String strVal = resolveEmbeddedValue((String) value);
				BeanDefinition bd = (beanName != null && containsBean(beanName) ? getMergedBeanDefinition(beanName) : null);
				//从指定BeanDefinition中的scope和BeanFactory范围执行表达式
				value = evaluateBeanDefinitionString(strVal, bd);
			}
			//转换成指定类型的值
			TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
			return (descriptor.getField() != null ?
					converter.convertIfNecessary(value, type, descriptor.getField()) :
					converter.convertIfNecessary(value, type, descriptor.getMethodParameter()));
		}
        //依赖类型是数组
		if (type.isArray()) {
		    //获取数组类型
			Class<?> componentType = type.getComponentType();
			//拷贝属性，构建一个新的DependencyDescriptor
			DependencyDescriptor targetDesc = new DependencyDescriptor(descriptor);
			//增加嵌入级别
			targetDesc.increaseNestingLevel();
			//(*1*)获取自动装配候选者
			Map<String, Object> matchingBeans = findAutowireCandidates(beanName, componentType, targetDesc);
			//如果没有获取到，并且这个bean是必须滴，那么就抛错，否则返回null
			if (matchingBeans.isEmpty()) {
				if (descriptor.isRequired()) {
					raiseNoSuchBeanDefinitionException(componentType, "array of " + componentType.getName(), descriptor);
				}
				return null;
			}
			//将匹配到的beanName存到自动装配beanName集合中，以便后面进行保存到依赖集合中
			if (autowiredBeanNames != null) {
				autowiredBeanNames.addAll(matchingBeans.keySet());
			}
			TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
			//转换类型
			Object result = converter.convertIfNecessary(matchingBeans.values(), type);
			if (getDependencyComparator() != null && result instanceof Object[]) {
			    //进行排序
				Arrays.sort((Object[]) result, adaptDependencyComparator(matchingBeans));
			}
			return result;
		}
		//如果依赖的是一个集合类型，并且依赖指定的类型为集合接口
		else if (Collection.class.isAssignableFrom(type) && type.isInterface()) {
		    //获取集合类型
			Class<?> elementType = descriptor.getCollectionType();
			//类型为空
			if (elementType == null) {
				if (descriptor.isRequired()) {
					throw new FatalBeanException("No element type declared for collection [" + type.getName() + "]");
				}
				return null;
			}
			//clone一份
			DependencyDescriptor targetDesc = new DependencyDescriptor(descriptor);
			//并且嵌入层级加1，以便下次获取嵌入类型为集合中的具体元素类型
			targetDesc.increaseNestingLevel();
			//获取符合类型的bean
			Map<String, Object> matchingBeans = findAutowireCandidates(beanName, elementType, targetDesc);
			//空，要么抛错，要么返回null
			if (matchingBeans.isEmpty()) {
				if (descriptor.isRequired()) {
					raiseNoSuchBeanDefinitionException(elementType, "collection of " + elementType.getName(), descriptor);
				}
				return null;
			}
			//加入依赖注册集合中
			if (autowiredBeanNames != null) {
				autowiredBeanNames.addAll(matchingBeans.keySet());
			}
			//转换类型并排序
			TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
			Object result = converter.convertIfNecessary(matchingBeans.values(), type);
			if (getDependencyComparator() != null && result instanceof List) {
				Collections.sort((List<?>) result, adaptDependencyComparator(matchingBeans));
			}
			return result;
		}
		//Map类型
		else if (Map.class.isAssignableFrom(type) && type.isInterface()) {
		    //获取map的key类型
			Class<?> keyType = descriptor.getMapKeyType();
			//如果不是String类型，那么将不被处理
			if (String.class != keyType) {
				if (descriptor.isRequired()) {
					throw new FatalBeanException("Key type [" + keyType + "] of map [" + type.getName() +
							"] must be [java.lang.String]");
				}
				return null;
			}
			//获取值类型
			Class<?> valueType = descriptor.getMapValueType();
			if (valueType == null) {
				if (descriptor.isRequired()) {
					throw new FatalBeanException("No value type declared for map [" + type.getName() + "]");
				}
				return null;
			}
			//clone一份
			DependencyDescriptor targetDesc = new DependencyDescriptor(descriptor);
			//并且嵌入层级加1
			targetDesc.increaseNestingLevel();
			//获取符合的bean类型
			Map<String, Object> matchingBeans = findAutowireCandidates(beanName, valueType, targetDesc);
			if (matchingBeans.isEmpty()) {
				if (descriptor.isRequired()) {
					raiseNoSuchBeanDefinitionException(valueType, "map with value type " + valueType.getName(), descriptor);
				}
				return null;
			}
			if (autowiredBeanNames != null) {
				autowiredBeanNames.addAll(matchingBeans.keySet());
			}
			//返回beanName-》bean对象
			return matchingBeans;
		}
		else {
		    //获取符合条件的候选者
			Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
			if (matchingBeans.isEmpty()) {
				if (descriptor.isRequired()) {
					raiseNoSuchBeanDefinitionException(type, "", descriptor);
				}
				return null;
			}
			//如果获取到的bean对象大于1个
			if (matchingBeans.size() > 1) {
			    //首先判断BeanDefinition的isPrimary是否为true，如果为true优先选择，这个isPrimary的值在xml配置文件中可以设置对应primary为true，在注解中
			    //可以设置@Primary注解标注它具有优先权，当spring发现有多个primary时，或者没有找到primary时，抛错
			    //其次就是使用@Priority注解来获取优先级更高的，spring中数字值越小，其优先级越高
			    //再其次就是类名与字段名后者方法参数名来选择，也就是根据名称自动装配
				String primaryBeanName = determineAutowireCandidate(matchingBeans, descriptor);
				if (primaryBeanName == null) {
					throw new NoUniqueBeanDefinitionException(type, matchingBeans.keySet());
				}
				if (autowiredBeanNames != null) {
					autowiredBeanNames.add(primaryBeanName);
				}
				//获取优先级高的bean
				return matchingBeans.get(primaryBeanName);
			}
			// We have exactly one match.
			//如果只知道一个匹配的，那么返回它即可
			Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
			if (autowiredBeanNames != null) {
			    //保存到自动装配beanName集合中，用于注解依赖
				autowiredBeanNames.add(entry.getKey());
			}
			return entry.getValue();
		}
	}
	
	
	//(*1*)
	protected Map<String, Object> org.springframework.beans.factory.support.DefaultListableBeanFactory.findAutowireCandidates(
			String beanName, Class<?> requiredType, DependencyDescriptor descriptor) {
        //（*2*）从BeanFactroy和它的父BeanFactory中获取对象类型beanName
		String[] candidateNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
				this, requiredType, true, descriptor.isEager());
		Map<String, Object> result = new LinkedHashMap<String, Object>(candidateNames.length);
		//这里额外的依赖对象是在我们分析容器的prepareBeanFactory，postProcessBeanFactory的时候加进来的
		//比如ServletRequest--》RequestObjectFactory，ServletResponse--》ResponseObjectFactory，HttpSession-》SessionObjectFactory
		for (Class<?> autowiringType : this.resolvableDependencies.keySet()) {
			if (autowiringType.isAssignableFrom(requiredType)) {
			   
				Object autowiringValue = this.resolvableDependencies.get(autowiringType);
				
				 //(*3*) 如果是ObjectFactory类型，并且
				autowiringValue = AutowireUtils.resolveAutowiringValue(autowiringValue, requiredType);
				if (requiredType.isInstance(autowiringValue)) {
					result.put(ObjectUtils.identityToString(autowiringValue), autowiringValue);
					break;
				}
			}
		}
		//循环符合类型匹配的beanName
		for (String candidateName : candidateNames) {
		    //是否引用了自己（不允许引用自己，否则将会一直在创建）并且这个bean确实可以做为候选的自动装配bean
			if (!isSelfReference(beanName, candidateName) && isAutowireCandidate(candidateName, descriptor)) {
			    //以beanName -》 bean对象的形式保存，getBean方法是我们用的非常熟悉的一个方法，它内部逻辑就有我们现讲的一部分内容
				result.put(candidateName, getBean(candidateName));
			}
		}
		//如果没有找到任何符合条件的bean，进行退化匹配，这里主要用于判断某个bean是否可以作为候选者的泛型匹配，在泛型为不能解析，比如泛型T
		//spring没法知道它具体是什么类型，对于这种一般情况下，spring一般不会匹配，所以只有在匹配不到任何对象的情况下，才会退而求其次进行匹配
		//比如TestBean<String>与TestBean<T>这种
		if (result.isEmpty()) {
			DependencyDescriptor fallbackDescriptor = descriptor.forFallbackMatch();
			for (String candidateName : candidateNames) {
				if (!candidateName.equals(beanName) && isAutowireCandidate(candidateName, fallbackDescriptor)) {
					result.put(candidateName, getBean(candidateName));
				}
			}
		}
		return result;
	}
	
	
	//(*2*)
	public static String[] beanNamesForTypeIncludingAncestors(
			ListableBeanFactory lbf, Class<?> type, boolean includeNonSingletons, boolean allowEagerInit) {

		Assert.notNull(lbf, "ListableBeanFactory must not be null");
		//我们在第11节中分析过这个函数，这个函数还是挺复杂的
		String[] result = lbf.getBeanNamesForType(type, includeNonSingletons, allowEagerInit);
		//如果这个BeanFactory是可以有继承关系的，那么也从父BeanFactory中获取
		if (lbf instanceof HierarchicalBeanFactory) {
			HierarchicalBeanFactory hbf = (HierarchicalBeanFactory) lbf;
			//是否为可枚举的BeanFactory
			if (hbf.getParentBeanFactory() instanceof ListableBeanFactory) {
			    //枚举能够找到的beanName
				String[] parentResult = beanNamesForTypeIncludingAncestors(
						(ListableBeanFactory) hbf.getParentBeanFactory(), type, includeNonSingletons, allowEagerInit);
				List<String> resultList = new ArrayList<String>();
				resultList.addAll(Arrays.asList(result));
				for (String beanName : parentResult) {
					if (!resultList.contains(beanName) && !hbf.containsLocalBean(beanName)) {
						resultList.add(beanName);
					}
				}
				result = StringUtils.toStringArray(resultList);
			}
		}
		return result;
	}
	
	//(*3*)
	public static Object resolveAutowiringValue(Object autowiringValue, Class<?> requiredType) {
	    //如果这个值是对象工厂，并且当前要求的类型不是对象工厂，那么要么创建代理，要么直接调用对象工厂getObject方法
		if (autowiringValue instanceof ObjectFactory && !requiredType.isInstance(autowiringValue)) {
			ObjectFactory<?> factory = (ObjectFactory<?>) autowiringValue;
			//如果这个ObjectFactory是可序列化的并且要求的类型是个接口，那么创建基于这个要求类型的代理
			//将工厂对象传入，每次调用都会请求对象工厂的getObject方法，举个例子，我们需要注入HttpServletRequest
			//request在spring对应的对象工厂是RequestObjectFactory，它的getObject方法每次都从当前线程的上下文中获取request对象，以保证线程安全
			if (autowiringValue instanceof Serializable && requiredType.isInterface()) {
				autowiringValue = Proxy.newProxyInstance(requiredType.getClassLoader(),
						new Class<?>[] {requiredType}, new ObjectFactoryDelegatingInvocationHandler(factory));
			}
			else {
			    //如果不符合代理条件，直接返回对象
				return factory.getObject();
			}
		}
		//直接返回正在装配的对象
		return autowiringValue;
	}
	
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;通过类型匹配到要返回的beanName还不够，因为对于用户而言，我并不需要所有符合类型的bean，我可能有多个实现，但是我只需要其中一个，设置有些bean，我压根就不让别人使用，它不能作为自动装配的候选者。
</p>


```
protected boolean isAutowireCandidate(String beanName, DependencyDescriptor descriptor, AutowireCandidateResolver resolver)
			throws NoSuchBeanDefinitionException {
        //去掉&前缀
		String beanDefinitionName = BeanFactoryUtils.transformedBeanName(beanName);
		//如果是具有BeanDefinition的，获取合并的BeanDefinition
		if (containsBeanDefinition(beanDefinitionName)) {
		    //(*1*)
			return isAutowireCandidate(beanName, getMergedLocalBeanDefinition(beanDefinitionName), descriptor, resolver);
		}
		
		//如果是它没有BeanDefinition，只是spring或者用户直接注册到BeanFactory的，那么构建一个BeanDefinition
		else if (containsSingleton(beanName)) {
			return isAutowireCandidate(beanName, new RootBeanDefinition(getType(beanName)), descriptor, resolver);
		}
		//如果父bean工厂是实现了默认可枚举BeanFactory，DefaultListableBeanFactory这个bean工厂额外提供了自动转配候选解析器，指定是否可覆盖BeanDefinition
		//是否允许循环依赖
		else if (getParentBeanFactory() instanceof DefaultListableBeanFactory) {
			// No bean definition found in this factory -> delegate to parent.
			return ((DefaultListableBeanFactory) getParentBeanFactory()).isAutowireCandidate(beanName, descriptor, resolver);
		}
		//如果不是默认可枚举bean工厂，但它是可配置的可枚举bean工厂，那么也就是它没有自动装配候选解析器，不能够把当前bean工厂的解析器通过参数传递
		else if (getParentBeanFactory() instanceof ConfigurableListableBeanFactory) {
			// If no DefaultListableBeanFactory, can't pass the resolver along.
			return ((ConfigurableListableBeanFactory) getParentBeanFactory()).isAutowireCandidate(beanName, descriptor);
		}
		else {
			return true;
		}
	}
	
	
	//(*1*)
	protected boolean isAutowireCandidate(String beanName, RootBeanDefinition mbd,
			DependencyDescriptor descriptor, AutowireCandidateResolver resolver) {
        //去掉&前缀
		String beanDefinitionName = BeanFactoryUtils.transformedBeanName(beanName);
		//将String类型BeanClass转换成Class对象
		resolveBeanClass(mbd, beanDefinitionName);
		//如果这个BeanDefinition被明确指定它的工厂方法是唯一的，那么只需要通过方法名，参数个数，类型确定即可
		//这个是一个简化版的获取工厂方法的步骤
		if (mbd.isFactoryMethodUnique) {
			boolean resolve;
			synchronized (mbd.constructorArgumentLock) {
				resolve = (mbd.resolvedConstructorOrFactoryMethod == null);
			}
			if (resolve) {
				new ConstructorResolver(this).resolveFactoryMethodIfPossible(mbd);
			}
		}
		//(*2*)
		return resolver.isAutowireCandidate(
				new BeanDefinitionHolder(mbd, beanName, getAliases(beanDefinitionName)), descriptor);
	}
	
	//(*2*)在分析自定标签component-scan时，spring给DefaultListableBeanFactory设置了一个自动装配候选解析器ContextAnnotationAutowireCandidateResolver
	//它继承自QualifierAnnotationAutowireCandidateResolver
	public boolean org.springframework.beans.factory.annotation.QualifierAnnotationAutowireCandidateResolver.isAutowireCandidate(BeanDefinitionHolder bdHolder, DependencyDescriptor descriptor) {
	    //调用父类泛型匹配，首先先判断它是否允许被用于自动装配（这是我们在配置文件可指定的）
	    //其次就是判断当前依赖除了类型匹配外，其泛型是否也是匹配的，比如List<Number>与List<Integer>是匹配的，但是List<Number>与List<String>是不匹配的
		boolean match = super.isAutowireCandidate(bdHolder, descriptor);
		//如果以上是匹配的，那么再进行Qualifier（预选）注解的匹配，通常的使用场景是我们需要注入的类型出现了两个以上的候选者，这个时候我们可以添加
		//@Qualifier注解指定需要哪个bean作为注入对象，Qualifier是可扩展的，我们可以在配置文件中指定属于自己定义的Qualifier，并指定其用于匹配比较的值
		if (match && descriptor != null) {
		    //检查当前依赖的上的注解是否存在对应的Qualifier，非常简单就是遍历注解，判断其注解class对象是否相等，是否是注解上的元注解
		    //（元注解在分析注解配置类的时候已经讲过，不记得的，回去看）
		    //然后检查对应Qualifier的值和设置的值属性，或者beanName是否相等
			match = checkQualifiers(bdHolder, descriptor.getAnnotations());
			if (match) {
			    //如果这个依赖是在工厂方法或者构造器上的，那么还要判断方法或构造器上的Qualifier是否满足条件
				MethodParameter methodParam = descriptor.getMethodParameter();
				if (methodParam != null) {
					Method method = methodParam.getMethod();
					//如果方法为null，那么就是构造器，或者说这个方法返回值为void（比如setter方法）
					if (method == null || void.class == method.getReturnType()) {
					    //(*3*)
						match = checkQualifiers(bdHolder, methodParam.getMethodAnnotations());
					}
				}
			}
		}
		return match;
	}
	
	
	//(*3*)
	protected boolean org.springframework.beans.factory.annotation.QualifierAnnotationAutowireCandidateResolver.checkQualifier(
			BeanDefinitionHolder bdHolder, Annotation annotation, TypeConverter typeConverter) {
        //当前依赖上找到的Qualifier注解
		Class<? extends Annotation> type = annotation.annotationType();
		//当前符合类型的BeanDefinition
		RootBeanDefinition bd = (RootBeanDefinition) bdHolder.getBeanDefinition();
        //获取当前Qualifier类型的AutowireCandidateQualifier对象
		AutowireCandidateQualifier qualifier = bd.getQualifier(type.getName());
		if (qualifier == null) {
		    //如果没有，那么获取去掉包名
			qualifier = bd.getQualifier(ClassUtils.getShortName(type));
		}
		//还是没有
		if (qualifier == null) {
			// First, check annotation on factory method, if applicable
			//从当前候选的bean的已解析的工厂方法或者构造器上面获取指定qualifier注解
			Annotation targetAnnotation = getFactoryMethodAnnotation(bd, type);
			//还是没有，就从被装饰的那个bean中找
			if (targetAnnotation == null) {
				RootBeanDefinition dbd = getResolvedDecoratedDefinition(bd);
				if (dbd != null) {
					targetAnnotation = getFactoryMethodAnnotation(dbd, type);
				}
			}
			//还是没有，那直接从候选bean的最终类型上找
			if (targetAnnotation == null) {
				// Look for matching annotation on the target class
				if (getBeanFactory() != null) {
					try {
					    //获取候选beanName的最终类型
						Class<?> beanType = getBeanFactory().getType(bdHolder.getBeanName());
						if (beanType != null) {
							targetAnnotation = AnnotationUtils.getAnnotation(ClassUtils.getUserClass(beanType), type);
						}
					}
					catch (NoSuchBeanDefinitionException ex) {
						// Not the usual case - simply forget about the type check...
					}
				}
				//还他娘的没有，比如getBeanFactory()返回null，那么直接获取当前获选bean的bean类型
				if (targetAnnotation == null && bd.hasBeanClass()) {
					targetAnnotation = AnnotationUtils.getAnnotation(ClassUtils.getUserClass(bd.getBeanClass()), type);
				}
			}
			//如果有了，那么比较两者的注解类型是否相等即可，这种策略是比较依赖的注解类型是否和候选者的工厂方法或者构造方法，类上的qualifier注释是否一致
			if (targetAnnotation != null && targetAnnotation.equals(annotation)) {
				return true;
			}
		}
        //获取依赖的所有属性
		Map<String, Object> attributes = AnnotationUtils.getAnnotationAttributes(annotation);
		//如果属性为空，并且没有注入用于对比的qualifier，直接返回false
		if (attributes.isEmpty() && qualifier == null) {
			// If no attributes, the qualifier must be present
			return false;
		}
		//遍历每个属性与样板qualifier进行对比
		for (Map.Entry<String, Object> entry : attributes.entrySet()) {
			String attributeName = entry.getKey();
			Object expectedValue = entry.getValue();
			Object actualValue = null;
			// Check qualifier first
			//如果存在qualifier，那么进行对比
			if (qualifier != null) {
				actualValue = qualifier.getAttribute(attributeName);
			}
			//如果没有值，退化从候选bean中获取对应属性值
			if (actualValue == null) {
				// Fall back on bean definition attribute
				actualValue = bd.getAttribute(attributeName);
			}
			//如果还是没有，并且值是String的，那么比较是否是指定候选bean的beanName
			if (actualValue == null && attributeName.equals(AutowireCandidateQualifier.VALUE_KEY) &&
					expectedValue instanceof String && bdHolder.matchesName((String) expectedValue)) {
				// Fall back on bean name (or alias) match
				continue;
			}
			//如果还是没有值，由存在对应的qualifier，那么退化获取其默认值，注解中有个default关键词用于指定默认值
			if (actualValue == null && qualifier != null) {
				// Fall back on default, but only if the qualifier is present
				actualValue = AnnotationUtils.getDefaultValue(annotation, attributeName);
			}
			//如果有值了，那么进行类型转换
			if (actualValue != null) {
				actualValue = typeConverter.convertIfNecessary(actualValue, expectedValue.getClass());
			}
			//不相等，返回false，相等，继续下一个属性的判断
			if (!expectedValue.equals(actualValue)) {
				return false;
			}
		}
		return true;
	}
	
	
	
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;以上就是使用工厂方法进行bean创建的过程，趁热打铁，接下来我们继续分析spring通过构造方法进行创建bean的过程，既然我们已经分析了spring是如何找到工厂方法创建bean的，那么如果找到创建bean的构造器，我相信大家都能够猜到了，没错，他们的实现是差不多的，只是有那么一点微小的不同罢了
</p>


```
/**
 * beanName 将要构造的bean的beanName
 * mbd 将要构造的bean的BeanDefinition
 * chosenCtors 候选的构造器，如果没有传递，spring将扫描bean的所有构造器
 * explicitArgs 指定的构造参数，如果没有指定，spring将使用配置的构造参数，甚至使用自动装配的参数
 */
public BeanWrapper autowireConstructor(final String beanName, final RootBeanDefinition mbd,
			Constructor<?>[] chosenCtors, final Object[] explicitArgs) {
        //这里和工厂方法是一样的
		BeanWrapperImpl bw = new BeanWrapperImpl();
		this.beanFactory.initBeanWrapper(bw);

		Constructor<?> constructorToUse = null;
		ArgumentsHolder argsHolderToUse = null;
		Object[] argsToUse = null;
        //使用指定的参数
		if (explicitArgs != null) {
			argsToUse = explicitArgs;
		}
		else {
			Object[] argsToResolve = null;
			synchronized (mbd.constructorArgumentLock) {
				constructorToUse = (Constructor<?>) mbd.resolvedConstructorOrFactoryMethod;
				//已经缓存了可用的构造器并且构造参数已被解析
				if (constructorToUse != null && mbd.constructorArgumentsResolved) {
					// Found a cached constructor...
					argsToUse = mbd.resolvedConstructorArguments;
					//如果为空，那么使用预备构造参数，预备参数不是最终参数，认真看了工厂方法解析的同学，我相信能够明白，这里不再赘述
					if (argsToUse == null) {
						argsToResolve = mbd.preparedConstructorArguments;
					}
				}
			}
			if (argsToResolve != null) {
			    //解析预备参数
				argsToUse = resolvePreparedArguments(beanName, mbd, bw, constructorToUse, argsToResolve);
			}
		}
        
		if (constructorToUse == null) {
			// Need to resolve the constructor.
			//是否为构造器自动装配
			boolean autowiring = (chosenCtors != null ||
					mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR);
			ConstructorArgumentValues resolvedValues = null;
            
			int minNrOfArgs;
			if (explicitArgs != null) {
				minNrOfArgs = explicitArgs.length;
			}
			else {
			    //获取配置的构造参数
				ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
				resolvedValues = new ConstructorArgumentValues();
				//获取解析过后的构造参数
				minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
			}

			// Take specified constructors, if any.
			Constructor<?>[] candidates = chosenCtors;
			//如果已经传入了可选的构造器，那么直接跳过
			//如果没有传入可选的构造其，那么反射获取
			if (candidates == null) {
				Class<?> beanClass = mbd.getBeanClass();
				try {
					candidates = (mbd.isNonPublicAccessAllowed() ?
							beanClass.getDeclaredConstructors() : beanClass.getConstructors());
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Resolution of declared constructors on bean Class [" + beanClass.getName() +
									"] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
				}
			}
			//根据public，参数长度降序排序
			AutowireUtils.sortConstructors(candidates);
			int minTypeDiffWeight = Integer.MAX_VALUE;
			Set<Constructor<?>> ambiguousConstructors = null;
			LinkedList<UnsatisfiedDependencyException> causes = null;
            
			for (int i = 0; i < candidates.length; i++) {
				Constructor<?> candidate = candidates[i];
				Class<?>[] paramTypes = candidate.getParameterTypes();
                //如果已经找到可用的构造器，并且当前循环的构造器参数个数已经比可用参数个数要少，那么直接退出循环，不要找了
				if (constructorToUse != null && argsToUse.length > paramTypes.length) {
					// Already found greedy constructor that can be satisfied ->
					// do not look any further, there are only less greedy constructors left.
					break;
				}
				if (paramTypes.length < minNrOfArgs) {
					continue;
				}

				ArgumentsHolder argsHolder;
				if (resolvedValues != null) {
					try {
					    //使用@ConstructorProperties标注构造参数的名称，如果没有spring只能自己通过放射的方式获取参数名
						String[] paramNames = ConstructorPropertiesChecker.evaluate(candidate, paramTypes.length);
						if (paramNames == null) {
							ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
							if (pnd != null) {
								paramNames = pnd.getParameterNames(candidate);
							}
						}
						//创建参数数组，处理已经解析的参数和需要自动装配的参数，和工厂方法是一样的
						argsHolder = createArgumentArray(
								beanName, mbd, resolvedValues, bw, paramTypes, paramNames, candidate, autowiring);
					}
					catch (UnsatisfiedDependencyException ex) {
						if (this.beanFactory.logger.isTraceEnabled()) {
							this.beanFactory.logger.trace(
									"Ignoring constructor [" + candidate + "] of bean '" + beanName + "': " + ex);
						}
						// Swallow and try next constructor.
						if (causes == null) {
							causes = new LinkedList<UnsatisfiedDependencyException>();
						}
						causes.add(ex);
						continue;
					}
				}
				else {
					// Explicit arguments given -> arguments length must match exactly.
					//使用指定参数时，参数个数必须匹配
					if (paramTypes.length != explicitArgs.length) {
						continue;
					}
					argsHolder = new ArgumentsHolder(explicitArgs);
				}
                //这里的权重设置和工厂方法是一样的，不再赘述
				int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
						argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
				// Choose this constructor if it represents the closest match.
				if (typeDiffWeight < minTypeDiffWeight) {
					constructorToUse = candidate;
					argsHolderToUse = argsHolder;
					argsToUse = argsHolder.arguments;
					minTypeDiffWeight = typeDiffWeight;
					ambiguousConstructors = null;
				}
				else if (constructorToUse != null && typeDiffWeight == minTypeDiffWeight) {
					if (ambiguousConstructors == null) {
						ambiguousConstructors = new LinkedHashSet<Constructor<?>>();
						ambiguousConstructors.add(constructorToUse);
					}
					ambiguousConstructors.add(candidate);
				}
			}
            //如果没有找到，抛错
			if (constructorToUse == null) {
				if (causes != null) {
					UnsatisfiedDependencyException ex = causes.removeLast();
					for (Exception cause : causes) {
						this.beanFactory.onSuppressedException(cause);
					}
					throw ex;
				}
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Could not resolve matching constructor " +
						"(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities)");
			}
			else if (ambiguousConstructors != null && !mbd.isLenientConstructorResolution()) {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Ambiguous constructor matches found in bean '" + beanName + "' " +
						"(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " +
						ambiguousConstructors);
			}

			if (explicitArgs == null) {
				argsHolderToUse.storeCache(mbd, constructorToUse);
			}
		}

		try {
			Object beanInstance;

			if (System.getSecurityManager() != null) {
				final Constructor<?> ctorToUse = constructorToUse;
				final Object[] argumentsToUse = argsToUse;
				beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
					@Override
					public Object run() {
						return beanFactory.getInstantiationStrategy().instantiate(
								mbd, beanName, beanFactory, ctorToUse, argumentsToUse);
					}
				}, beanFactory.getAccessControlContext());
			}
			else {
			    //使用指定的构造器构造对象，当然，如果存在LookMethod，replaceMethod方法，那么还需要进行cglib代理
				beanInstance = this.beanFactory.getInstantiationStrategy().instantiate(
						mbd, beanName, this.beanFactory, constructorToUse, argsToUse);
			}

			bw.setWrappedInstance(beanInstance);
			return bw;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean instantiation via constructor failed", ex);
		}
	}
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;可以看到，其实spring筛选构造器和筛选工厂方法是差不多的逻辑。其细节不再赘述
</p>











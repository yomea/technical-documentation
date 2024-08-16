<p>
&nbsp;&nbsp;&nbsp;&nbsp;aop标签属于自定义标签，所以对于这类标签需要实现对应命名空间和标签解析器
</p>


```
public void org.springframework.aop.config.AopNamespaceHandler.init() {
		// In 2.0 XSD as well as in 2.1 XSD.
		//注册解析config标签的解析器
		registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
		//注册解析aspectj-autoproxy标签的解析器
		registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
		//注册解析范围代理的解析器
		registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());

		// Only in 2.0 XSD: moved to context namespace as of 2.1
		//注册spring-configured的解析器
		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;关于aspectj-autoproxy标签的解析我们在前面配置文件的解析的时候分析过，这个标签主要指定是否暴露代理对象并构建一个BeanDefinition注册到工厂中，这是一个后置处理器，其类名为AnnotationAwareAspectJAutoProxyCreator，下面是它的继承结构
</p>

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/029fddf63e9976ecc9afaf05c9bd1197.png)

从类继承图来看，这个类继承了BeanPostProcessor，BeanPostProcessor我们已经很熟悉了，在bean的创建的时候让用户参与其中，干预bean的创建，Aware接口就不多说了，这个看到过太多次了，我们主要谈谈，AOP这一块。

ProxyConfig 代理配置，用于储存是否使用cglib代理，是否暴露依赖等

AopInfrastructureBean 标记接口，表示一个aop基础bean，这类bean即使符合表达式也不会被代理

ProxyProcessorSupport 实现基础能力，比如寻找接口，如果没有获取到合适的接口，那么会设置配置为cglib代理

AbstractAutoProxyCreator 模板类，实现一些基本功能，实现了实例化前处理逻辑，对自定义的目标源代理

AbstractAdvisorAutoProxyCreator 这个类主要实现了获取Advisor的能力

AspectJAwareAdvisorAutoProxyCreator 定义了根据优先级，定义顺序，是否来自同一个切面的排序，定义跳过逻辑

AnnotationAwareAspectJAutoProxyCreator 增加注解aop的能力


<p>
&nbsp;&nbsp;&nbsp;&nbsp;虽然上面提到了AnnotationAwareAspectJAutoProxyCreator，但是我们还是要先从它的父类AspectJAwareAdvisorAutoProxyCreator开始，如果我们配置的是config标签，那么spring注册到BeanFactory的就是这个父类，首先看到解析config标签的ConfigBeanDefinitionParser
</p>


```
<aop:config>
	
	<aop:aspect id="myAop" ref="aopTest">
		<aop:pointcut id="target" expression="execution(* com.zhipin.service.*.*(..))" />
		<aop:before method="before" pointcut-ref="target" />
		<aop:after method="after" pointcut-ref="target" />
		<aop:around method="around" pointcut-ref="target" />
	</aop:aspect>

	
	<aop:advisor advice-ref="txAdvice"	pointcut="execution(* com.zhipin.service.*.*(..))" />
		
</aop:config>
<bean id="aopTest" class="com.zhipin.aop.AopTest"></bean>


<tx:advice id="txAdvice" transaction-manager="txManager">
	<tx:attributes>
		<tx:method name="save*" propagation="REQUIRED" />
		<tx:method name="delete*" propagation="REQUIRED" />
		<tx:method name="update*" propagation="REQUIRED" />
		<tx:method name="*" propagation="SUPPORTS" read-only="true" />
	</tx:attributes>
</tx:advice>
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
	<property name="dataSource" ref="dataSource" />
</bean>

```


```
public BeanDefinition org.springframework.aop.config.ConfigBeanDefinitionParser.parse(Element element, ParserContext parserContext) {
        //创建复合组件定义
		CompositeComponentDefinition compositeDef =
				new CompositeComponentDefinition(element.getTagName(), parserContext.extractSource(element));
		parserContext.pushContainingComponent(compositeDef);

        //创建AspectJAwareAdvisorAutoProxyCreator的BeanDefinition
		configureAutoProxyCreator(parserContext, element);
        
        //获取其子元素
		List<Element> childElts = DomUtils.getChildElements(element);
		for (Element elt: childElts) {
			String localName = parserContext.getDelegate().getLocalName(elt);
			//处理切点标签
			if (POINTCUT.equals(localName)) {
				parsePointcut(elt, parserContext);
			}
			//处理通知者（一组通知组成的，甚至只有一个通知）
			else if (ADVISOR.equals(localName)) {
				parseAdvisor(elt, parserContext);
			}
			//处理切面
			else if (ASPECT.equals(localName)) {
				parseAspect(elt, parserContext);
			}
		}

		parserContext.popAndRegisterContainingComponent();
		return null;
	}
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;我们先来看看对pointcut标签的解析
</p>


```
private AbstractBeanDefinition org.springframework.aop.config.ConfigBeanDefinitionParser.parsePointcut(Element pointcutElement, ParserContext parserContext) {
        //解析切点的唯一标识，这个标识用于通知对其引用，通知才知道要将自己置于何处
		String id = pointcutElement.getAttribute(ID);
		//切点表达式
		String expression = pointcutElement.getAttribute(EXPRESSION);

		AbstractBeanDefinition pointcutDefinition = null;

		try {
			this.parseState.push(new PointcutEntry(id));
			//创建切点BeanDefinition，对应的被定义的类为AspectJExpressionPointcut
			pointcutDefinition = createPointcutDefinition(expression);
			//设置原始来源
			pointcutDefinition.setSource(parserContext.extractSource(pointcutElement));

			String pointcutBeanName = id;
			//注册到BeanFactory，如果有id，那么将id作为切点BeanDefinition的beanName，否则生成beanName
			//生成BeanName的生成器在前面提到过，在这里使用的是DefaultBeanNameGenerator，具体的生成方式不再赘述
			if (StringUtils.hasText(pointcutBeanName)) {
				parserContext.getRegistry().registerBeanDefinition(pointcutBeanName, pointcutDefinition);
			}
			else {
				pointcutBeanName = parserContext.getReaderContext().registerWithGeneratedName(pointcutDefinition);
			}
            //注册切点组件定义
			parserContext.registerComponent(
					new PointcutComponentDefinition(pointcutBeanName, pointcutDefinition, expression));
		}
		finally {
			this.parseState.pop();
		}

		return pointcutDefinition;
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;解析advisor
</p>


```
private void org.springframework.aop.config.ConfigBeanDefinitionParser.parseAdvisor(Element advisorElement, ParserContext parserContext) {
        //创建DefaultBeanFactoryPointcutAdvisor的BeanDefinition
        //(*1*)
		AbstractBeanDefinition advisorDef = createAdvisorBeanDefinition(advisorElement, parserContext);
		String id = advisorElement.getAttribute(ID);

		try {
			this.parseState.push(new AdvisorEntry(id));
			//注册通知者到BeanFactory
			String advisorBeanName = id;
			if (StringUtils.hasText(advisorBeanName)) {
				parserContext.getRegistry().registerBeanDefinition(advisorBeanName, advisorDef);
			}
			else {
				advisorBeanName = parserContext.getReaderContext().registerWithGeneratedName(advisorDef);
			}
            //如果是pointcut属性，那么返回AspectJExpressionPointcut类的BeanDefinition
            //如果是pointcut-ref属性，那么返回切点名
			Object pointcut = parsePointcutProperty(advisorElement, parserContext);
			//添加被包含切点bean定义
			if (pointcut instanceof BeanDefinition) {
				advisorDef.getPropertyValues().add(POINTCUT, pointcut);
				parserContext.registerComponent(
						new AdvisorComponentDefinition(advisorBeanName, advisorDef, (BeanDefinition) pointcut));
			}
			//如果是引用的其他的切点，那么包装成RuntimeBeanReference，这个类型属性值，会在应用属性值的时候通过
			//BeanDefinitionValueResolver进行解析，我们在bean的创建时分析过，不再赘述
			else if (pointcut instanceof String) {
				advisorDef.getPropertyValues().add(POINTCUT, new RuntimeBeanReference((String) pointcut));
				parserContext.registerComponent(
						new AdvisorComponentDefinition(advisorBeanName, advisorDef));
			}
		}
		finally {
			this.parseState.pop();
		}
	}
	
	
	//(*1*)
	private AbstractBeanDefinition createAdvisorBeanDefinition(Element advisorElement, ParserContext parserContext) {
	    //创建DefaultBeanFactoryPointcutAdvisor的BeanDefinition
		RootBeanDefinition advisorDefinition = new RootBeanDefinition(DefaultBeanFactoryPointcutAdvisor.class);
		advisorDefinition.setSource(parserContext.extractSource(advisorElement));

        //获取advice-ref属性（引用的通知）
		String adviceRef = advisorElement.getAttribute(ADVICE_REF);
		if (!StringUtils.hasText(adviceRef)) {
			parserContext.getReaderContext().error(
					"'advice-ref' attribute contains empty value.", advisorElement, this.parseState.snapshot());
		}
		else {
		    //设置属性，这里对通知的引用使用的是RuntimeBeanNameReference进行包装，表示这个属性值在BeanFactory中是否有对应bean，如果没有
		    //将抛错
			advisorDefinition.getPropertyValues().add(
					ADVICE_BEAN_NAME, new RuntimeBeanNameReference(adviceRef));
		}
        //如果有order属性，那么获取（用于对个通知者存在时排序）
		if (advisorElement.hasAttribute(ORDER_PROPERTY)) {
			advisorDefinition.getPropertyValues().add(
					ORDER_PROPERTY, advisorElement.getAttribute(ORDER_PROPERTY));
		}

		return advisorDefinition;
	}
	
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;最后来看下aspect的解析
</p>


```
private void parseAspect(Element aspectElement, ParserContext parserContext) {
        //解析切面id
		String aspectId = aspectElement.getAttribute(ID);
		//解析切面beanName
		String aspectName = aspectElement.getAttribute(REF);

		try {
			this.parseState.push(new AspectEntry(aspectId, aspectName));
			List<BeanDefinition> beanDefinitions = new ArrayList<BeanDefinition>();
			List<BeanReference> beanReferences = new ArrayList<BeanReference>();
            //获取declare-parents标签，这个标签用于引介增强
			List<Element> declareParents = DomUtils.getChildElementsByTagName(aspectElement, DECLARE_PARENTS);
			for (int i = METHOD_INDEX; i < declareParents.size(); i++) {
				Element declareParentsElement = declareParents.get(i);
				//(*1*)
				beanDefinitions.add(parseDeclareParents(declareParentsElement, parserContext));
			}

			// We have to parse "advice" and all the advice kinds in one loop, to get the
			// ordering semantics right.
			NodeList nodeList = aspectElement.getChildNodes();
			boolean adviceFoundAlready = false;
			for (int i = 0; i < nodeList.getLength(); i++) {
				Node node = nodeList.item(i);
				//标签名是否是before，after，after-returning，after-throwing，around
				if (isAdviceNode(node, parserContext)) {
				    //是否已找到通知
					if (!adviceFoundAlready) {
					    //找到了通知，却没有设置切面，报错
						adviceFoundAlready = true;
						if (!StringUtils.hasText(aspectName)) {
							parserContext.getReaderContext().error(
									"<aspect> tag needs aspect bean reference via 'ref' attribute when declaring advices.",
									aspectElement, this.parseState.snapshot());
							return;
						}
						//添加运行时bean引用
						beanReferences.add(new RuntimeBeanReference(aspectName));
					}
					//构建对应的通知BeanDefinition
					AbstractBeanDefinition advisorDefinition = parseAdvice(
							aspectName, i, aspectElement, (Element) node, parserContext, beanDefinitions, beanReferences);
					//保存解析后的通知BeanDefinition
					beanDefinitions.add(advisorDefinition);
				}
			}

			AspectComponentDefinition aspectComponentDefinition = createAspectComponentDefinition(
					aspectElement, aspectId, beanDefinitions, beanReferences, parserContext);
			parserContext.pushContainingComponent(aspectComponentDefinition);

			List<Element> pointcuts = DomUtils.getChildElementsByTagName(aspectElement, POINTCUT);
			for (Element pointcutElement : pointcuts) {
			    //解析单独的切点标签
				parsePointcut(pointcutElement, parserContext);
			}

			parserContext.popAndRegisterContainingComponent();
		}
		finally {
			this.parseState.pop();
		}
	}
	
	
	//(*1*)
	private AbstractBeanDefinition parseDeclareParents(Element declareParentsElement, ParserContext parserContext) {
	    //定义DeclareParentsAdvisor的BeanDefinition
		BeanDefinitionBuilder builder = BeanDefinitionBuilder.rootBeanDefinition(DeclareParentsAdvisor.class);
		//解析构造参数，implement-interface
		builder.addConstructorArgValue(declareParentsElement.getAttribute(IMPLEMENT_INTERFACE));
		//解析构造参数，types-matching
		builder.addConstructorArgValue(declareParentsElement.getAttribute(TYPE_PATTERN));

        //解析default-impl，默认实现类名
		String defaultImpl = declareParentsElement.getAttribute(DEFAULT_IMPL);
		//解析delegate-ref，默认实现引用
		String delegateRef = declareParentsElement.getAttribute(DELEGATE_REF);
        
        //代理实现引用和默认实现不能同时存在
		if (StringUtils.hasText(defaultImpl) && !StringUtils.hasText(delegateRef)) {
			builder.addConstructorArgValue(defaultImpl);
		}
		//包装成RuntimeBeanReference
		else if (StringUtils.hasText(delegateRef) && !StringUtils.hasText(defaultImpl)) {
			builder.addConstructorArgReference(delegateRef);
		}
		else {
			parserContext.getReaderContext().error(
					"Exactly one of the " + DEFAULT_IMPL + " or " + DELEGATE_REF + " attributes must be specified",
					declareParentsElement, this.parseState.snapshot());
		}
        //创建对应的BeanDefinition，并注册
		AbstractBeanDefinition definition = builder.getBeanDefinition();
		definition.setSource(parserContext.extractSource(declareParentsElement));
		parserContext.getReaderContext().registerWithGeneratedName(definition);
		return definition;
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;下面我们继续看spring是如果解析after，before等通知的
</p>


```
private AbstractBeanDefinition org.springframework.aop.config.ConfigBeanDefinitionParser.parseAdvice(
			String aspectName, int order, Element aspectElement, Element adviceElement, ParserContext parserContext,
			List<BeanDefinition> beanDefinitions, List<BeanReference> beanReferences) {

		try {
			this.parseState.push(new AdviceEntry(parserContext.getDelegate().getLocalName(adviceElement)));

			// create the method factory bean
			//MethodLocatingFactoryBean，除了实现了FactoryBean还实现了BeanFactoryAware
			RootBeanDefinition methodDefinition = new RootBeanDefinition(MethodLocatingFactoryBean.class);
			methodDefinition.getPropertyValues().add("targetBeanName", aspectName);
			methodDefinition.getPropertyValues().add("methodName", adviceElement.getAttribute("method"));
			methodDefinition.setSynthetic(true);

			// create instance factory definition
			//创建切面工厂，用于从BeanFactory中获取切面对象
			RootBeanDefinition aspectFactoryDef =
					new RootBeanDefinition(SimpleBeanFactoryAwareAspectInstanceFactory.class);
			aspectFactoryDef.getPropertyValues().add("aspectBeanName", aspectName);
			aspectFactoryDef.setSynthetic(true);

			// register the pointcut
			//(*1*)
			AbstractBeanDefinition adviceDef = createAdviceDefinition(
					adviceElement, parserContext, aspectName, order, methodDefinition, aspectFactoryDef,
					beanDefinitions, beanReferences);

			// configure the advisor
			//创建AspectJPointcutAdvisor的BeanDefinition
			RootBeanDefinition advisorDefinition = new RootBeanDefinition(AspectJPointcutAdvisor.class);
			advisorDefinition.setSource(parserContext.extractSource(adviceElement));
			//添加通知构造参数，使用AspectJPointcutAdvisor包装具体的通知
			advisorDefinition.getConstructorArgumentValues().addGenericArgumentValue(adviceDef);
			if (aspectElement.hasAttribute(ORDER_PROPERTY)) {
				advisorDefinition.getPropertyValues().add(
						ORDER_PROPERTY, aspectElement.getAttribute(ORDER_PROPERTY));
			}

			// register the final advisor
			//注册AspectJPointcutAdvisor通知者
			parserContext.getReaderContext().registerWithGeneratedName(advisorDefinition);

			return advisorDefinition;
		}
		finally {
			this.parseState.pop();
		}
	}
	
	//(*1*)
	private AbstractBeanDefinition createAdviceDefinition(
			Element adviceElement, ParserContext parserContext, String aspectName, int order,
			RootBeanDefinition methodDef, RootBeanDefinition aspectFactoryDef,
			List<BeanDefinition> beanDefinitions, List<BeanReference> beanReferences) {
        //(*2*)
		RootBeanDefinition adviceDefinition = new RootBeanDefinition(getAdviceClass(adviceElement, parserContext));
		adviceDefinition.setSource(parserContext.extractSource(adviceElement));

        //添加通知所属切面beanName
		adviceDefinition.getPropertyValues().add(ASPECT_NAME_PROPERTY, aspectName);
		//设置定义顺序
		adviceDefinition.getPropertyValues().add(DECLARATION_ORDER_PROPERTY, order);
        
        //设置returning返回属性，用于承载返回值的变量
		if (adviceElement.hasAttribute(RETURNING)) {
			adviceDefinition.getPropertyValues().add(
					RETURNING_PROPERTY, adviceElement.getAttribute(RETURNING));
		}
		//设置throwing抛出异常属性，设置抛出异常时用于承载数据的变量
		if (adviceElement.hasAttribute(THROWING)) {
			adviceDefinition.getPropertyValues().add(
					THROWING_PROPERTY, adviceElement.getAttribute(THROWING));
		}
		//设置arg-names参数名属性，用于承载连接点传入的参数名
		if (adviceElement.hasAttribute(ARG_NAMES)) {
			adviceDefinition.getPropertyValues().add(
					ARG_NAMES_PROPERTY, adviceElement.getAttribute(ARG_NAMES));
		}
        //添加MethodLocatingFactoryBean的BeanDefinition构造参数
		ConstructorArgumentValues cav = adviceDefinition.getConstructorArgumentValues();
		cav.addIndexedArgumentValue(METHOD_INDEX, methodDef);
        //解析通知标签的pointcut属性
		Object pointcut = parsePointcutProperty(adviceElement, parserContext);
		//设置切点构造参数
		if (pointcut instanceof BeanDefinition) {
			cav.addIndexedArgumentValue(POINTCUT_INDEX, pointcut);
			beanDefinitions.add((BeanDefinition) pointcut);
		}
		else if (pointcut instanceof String) {
			RuntimeBeanReference pointcutRef = new RuntimeBeanReference((String) pointcut);
			cav.addIndexedArgumentValue(POINTCUT_INDEX, pointcutRef);
			beanReferences.add(pointcutRef);
		}
        //设置切面工厂参数
		cav.addIndexedArgumentValue(ASPECT_INSTANCE_FACTORY_INDEX, aspectFactoryDef);
        //返回通知BeanDefinition
		return adviceDefinition;
	}
	
	//(*2*)
	private Class<?> getAdviceClass(Element adviceElement, ParserContext parserContext) {
		String elementName = parserContext.getDelegate().getLocalName(adviceElement);
		//如果是before，返回AspectJMethodBeforeAdvice通知
		if (BEFORE.equals(elementName)) {
			return AspectJMethodBeforeAdvice.class;
		}
		//如果是after，返回AspectJAfterAdvice通知
		else if (AFTER.equals(elementName)) {
			return AspectJAfterAdvice.class;
		}
		//如果是after-returning，返回AspectJAfterReturningAdvice通知
		else if (AFTER_RETURNING_ELEMENT.equals(elementName)) {
			return AspectJAfterReturningAdvice.class;
		}
		//如果是after-throwing，返回AspectJAfterThrowingAdvice通知
		else if (AFTER_THROWING_ELEMENT.equals(elementName)) {
			return AspectJAfterThrowingAdvice.class;
		}
		//如果是around，返回AspectJAroundAdvice通知
		else if (AROUND.equals(elementName)) {
			return AspectJAroundAdvice.class;
		}
		else {
			throw new IllegalArgumentException("Unknown advice kind [" + elementName + "].");
		}
	}
	
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;spring将切面包装成AspectJExpressionPointcut，将advisor标签包装成了DefaultBeanFactoryPointcutAdvisor，将advice包装成了AspectJPointcutAdvisor
</p>

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/84b67f44aa2d224be7f0f1a4dfe4b9a9.png)

Pointcut 切点接口，用于获取方法匹配器和类过滤器匹配器

ExpressionPointcut 继承Pointcut接口，额外定义了获取切点表示的能力

AbstractExpressionPointcut 定义了设置location，看注释的含义是设置调试的位置，本人未动，如果有看懂的请赐教

MethodMatcher 匹配方法

ClassFilter 匹配类

AspectJExpressionPointcut 实现用切点表达式匹配


<p>
&nbsp;&nbsp;&nbsp;&nbsp;下面是解析advisor标签构建的Bean的继承结构
</p>

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/8cebd216dc85b68d85fa32bcc506cece.png)

Advisor 通知者，持有通知甚至是切点，理解为通知的holder就行了

PointcutAdvisor 继承自Advisor，增加获取pointcut的能力

AbstractPointcutAdvisor 增加获取order的能力

AbstractBeanFactoryPointcutAdvisor 实现获取advice的能力

DefaultBeanFactoryPointcutAdvisor 实现获取切点的能力


<p>
&nbsp;&nbsp;&nbsp;&nbsp;下面是解析advice标签构建的AspectJPointcutAdvisor的继承结构
</p>

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/063bdb61039519698c6d32f985f096a1.png)

Advisor 通知者，持有通知甚至是切点，理解为通知的holder就行了

PointcutAdvisor 继承自Advisor，增加获取pointcut的能力

AspectJPointcutAdvisor 获取Advice，获取pointcut


<p>
&nbsp;&nbsp;&nbsp;&nbsp;xml配置aop到此就告一段落了，我们再来看下注解是如果配置aop的，aop注解的解析是由AnnotationAwareAspectJAutoProxyCreator来实现的，从上面的类图，我可以看到这个类实现BeanPostProcessor，InstantiationAwareBeanPostProcessor，SmartInstantiationAwareBeanPostProcessor接口
</p>

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/b54c49c14b1ffc2d7b7be7882d98770a.png)

BeanPostProcessor

```
public interface BeanPostProcessor {
    //bean初始化前调用
	Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
    //bean初始化后调用
	Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;

}
```

InstantiationAwareBeanPostProcessor

```
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {

    //bean实例化前调用
	Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException;
    //bean实例化后调用
	boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException;
    //bean填充属性前调用
	PropertyValues postProcessPropertyValues(
			PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName)
			throws BeansException;

}
```

SmartInstantiationAwareBeanPostProcessor

```
public interface SmartInstantiationAwareBeanPostProcessor extends InstantiationAwareBeanPostProcessor {

	//判定类型时调用，在aop中是判断代理类的类型
	Class<?> predictBeanType(Class<?> beanClass, String beanName) throws BeansException;

	//创建bean需要获取构造器创建时调用
	Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName) throws BeansException;

	//获取早期bean引用，如果是在aop的实现中，那么是代理早期bean
	Object getEarlyBeanReference(Object bean, String beanName) throws BeansException;

}

```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;我们分别过下以上各个接口方法在AnnotationAwareAspectJAutoProxyCreator上是怎么实现的，我们先来看看predictBeanType方法，这个方法被实现在AnnotationAwareAspectJAutoProxyCreator的父类AbstractAutoProxyCreator中
</p>


```
public Class<?> predictBeanType(Class<?> beanClass, String beanName) {
		if (this.proxyTypes.isEmpty()) {
			return null;
		}
		//根据beanClass与beanName合成一个缓存key
		Object cacheKey = getCacheKey(beanClass, beanName);
		//通过缓存key获取对应bean的代理类型
		return this.proxyTypes.get(cacheKey);
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;determineCandidateConstructors方法，这个方法也是被实现在AbstractAutoProxyCreator中
</p>

```
public Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName) throws BeansException {
        //直接返回了null
		return null;
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;getEarlyBeanReference方法，这个方法也是被实现在AbstractAutoProxyCreator中
</p>


```
public Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
        //生成缓存key
		Object cacheKey = getCacheKey(bean.getClass(), beanName);
		//判断是否已经存在这个缓存key，如果不存在，就添加进去，主要是用于后面需要直接创建代理时做判断
		if (!this.earlyProxyReferences.contains(cacheKey)) {
			this.earlyProxyReferences.add(cacheKey);
		}
		//创建代理，此处不细说，后面分析初始化后置的时候会详细讲解
		return wrapIfNecessary(bean, beanName, cacheKey);
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;postProcessBeforeInstantiation方法，这个方法也是被实现在AbstractAutoProxyCreator中
</p>


```
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        //和前面的一样生成缓存key
		Object cacheKey = getCacheKey(beanClass, beanName);
        //如果没有beanName或者不是自定义目标对象
		if (beanName == null || !this.targetSourcedBeans.contains(beanName)) {
		    //是否已经包含，如果已经创建过代理或被标记为基础bean，或者需要跳过的bean，那么直接返回
			if (this.advisedBeans.containsKey(cacheKey)) {
				return null;
			}
			//如果当前bean类型为Advice，Advisor，AopInfrastructureBean，将不被代理
			//或者当前bean为切面bean
			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return null;
			}
		}

		// Create proxy here if we have a custom TargetSource.
		// Suppresses unnecessary default instantiation of the target bean:
		// The TargetSource will handle target instances in a custom fashion.
		//如果我们自定义了TargetSource（用于包装目标对象的类），那么对自定义的TargetSource进行代理
		if (beanName != null) {
			TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
			if (targetSource != null) {
			    //如果我们确实自己自定义包装了当前bean，那么将当前beanName缓存到targetSourcedBeans集合中
			    //在初始化后置的时候就不会进行代理
				this.targetSourcedBeans.add(beanName);
				//获取Advisor，具体逻辑，在分析初始化后置的时候详细说明
				Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
				//创建代理对象，具体逻辑，在分析初始化后置的时候详细说明
				Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
				//缓存代理类型
				this.proxyTypes.put(cacheKey, proxy.getClass());
				return proxy;
			}
		}

		return null;
	}
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;postProcessAfterInstantiation实例化后置
</p>


```
public boolean postProcessAfterInstantiation(Object bean, String beanName) {
		return true;
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;postProcessPropertyValues属性填充后置
</p>


```
public PropertyValues postProcessPropertyValues(
			PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) {

		return pvs;
	}
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;postProcessBeforeInitialization初始化前置
</p>


```
public Object postProcessBeforeInitialization(Object bean, String beanName) {
		return bean;
	}
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;postProcessAfterInitialization初始化后置
</p>


```
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if (bean != null) {
		    //根据bean的class与beanName生成缓存key
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			//是否被包含在了早期引用集合中，这里我们在上面分析getEarlyBeanReference方法的时候讲到过
			//表示代理早期bean对象
			if (!this.earlyProxyReferences.contains(cacheKey)) {
			    //如果未被代理过，那么重新代理
			    //(*1*)
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
	
	//(*1*)
	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
	    //是否在自定义目标beanName集合中已经存在，如果已经存在，说明在bean的实例化前就进行了代理
	    //不会重新代理
		if (beanName != null && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		//是否是基础aop相关的bean，或者是切面，是那么直接返回
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		//判断当前bean是否为Advice，Advisor，AopInfrastructureBean类型或者是否为切面
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
		    //加入缓存，表示下次不再进行代理
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// Create proxy if we have advice.
		//获取Advisor
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		
		if (specificInterceptors != DO_NOT_PROXY) {
		    //如果找到，那么标识当前bean能够被代理
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			//创建代理
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			//缓存代理类型
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}
        //如果没有找到Advisor，那么缓存当前bean不能被代理
		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
	
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;在对目标bean进行代理之前，需要先将通知拦截逻辑找出来，作用于目标对象上才能够实现aop，那么spring是如果查找这些通知拦截逻辑的呢？
</p>


```
//beanClass 表示当前需要代理的bean的class对象
//beanName 表示当前需要代理的bean的beanName
//targetSource 需要代理的目标对象包装器
protected Object[] org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator.getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, TargetSource targetSource) {
        //查找可应用到当前bean的通知者
        //(*1*)
		List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
		if (advisors.isEmpty()) {
			return DO_NOT_PROXY;
		}
		return advisors.toArray();
	}
	
	//(*1*)
	protected List<Advisor> org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator.findEligibleAdvisors(Class<?> beanClass, String beanName) {
	    //查找候选的通知器
		List<Advisor> candidateAdvisors = findCandidateAdvisors();
		//过滤掉不适用当前bean的通知器
		List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
		//设置扩展的通知器，设置了一个暴露调用信息的通知器
		//(*1*)
		extendAdvisors(eligibleAdvisors);
		if (!eligibleAdvisors.isEmpty()) {
		    //对通知器进行排序，通过优先级，或者定义顺序，或者AnnotationAwareOrderComparator进行排序
		    //AnnotationAwareOrderComparator这个比较器我们在前面已经见到过很多次了，它主要是先通过是否实现了PriorityOrdered接口
		    //或Ordered接口 或 @Order 或 @Priority进行排序
			eligibleAdvisors = sortAdvisors(eligibleAdvisors);
		}
		return eligibleAdvisors;
	}
	
	//(*1*)
	protected void org.springframework.aop.aspectj.autoproxy.AspectJAwareAdvisorAutoProxyCreator.extendAdvisors(List<Advisor> candidateAdvisors) {
	    //（*2*）
		AspectJProxyUtils.makeAdvisorChainAspectJCapableIfNecessary(candidateAdvisors);
	}
	
	//（*2*）
	public static boolean AspectJProxyUtils.makeAdvisorChainAspectJCapableIfNecessary(List<Advisor> advisors) {
		// Don't add advisors to an empty list; may indicate that proxying is just not required
		if (!advisors.isEmpty()) {
			boolean foundAspectJAdvice = false;
			for (Advisor advisor : advisors) {
				// Be careful not to get the Advice without a guard, as
				// this might eagerly instantiate a non-singleton AspectJ aspect
				//查找是否是切面类aop通知
				//判断条件
				//(advisor instanceof InstantiationModelAwarePointcutAdvisor || advisor.getAdvice() instanceof AbstractAspectJAdvice ||
				//(advisor instanceof PointcutAdvisor && ((PointcutAdvisor) advisor).getPointcut() instanceof AspectJExpressionPointcut));
				if (isAspectJAdvice(advisor)) {
					foundAspectJAdvice = true;
				}
			}
			//添加暴露调用信息的通知器，这个通知器内部持有ExposeInvocationInterceptor拦截器
			if (foundAspectJAdvice && !advisors.contains(ExposeInvocationInterceptor.ADVISOR)) {
				advisors.add(0, ExposeInvocationInterceptor.ADVISOR);
				return true;
			}
		}
		return false;
	}
	
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;来看下ExposeInvocationInterceptor的继承接口
</p>

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/838251938a2195311c242b44e7050589.png)

可以看到，这个拦截器实现了Advice接口，这个接口是用来表示某个类是否为通知的标记接口

Interceptor 标记接口，表示通用拦截器

MethodInterceptor 方法拦截器，我们所使用的通知最终都会去实现这么一个接口，即使不实现，也会使用适配器进行适配

ExposeInvocationInterceptor 用于暴露当前将要调用的方法信息

<p>
&nbsp;&nbsp;&nbsp;&nbsp;继续回到获取advisor的代码逻辑
</p>

```
	protected List<Advisor> org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator.findCandidateAdvisors() {
		// Add all the Spring advisors found according to superclass rules.
		//spring虽然使用了注解获取aop信息，但是也不会抛弃传统配置生成的Advisor，所以调用父类的方法从BeanFactory获取Advisor类型的bean
		//(*1*)
		List<Advisor> advisors = super.findCandidateAdvisors();
		// Build Advisors for all AspectJ aspects in the bean factory.
		//循环所有的beanClass，检测是否被@Aspect注解标注，如果被标注解析上面的@Pointcut，@Before等注解
		advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
		return advisors;
	}
	
	//(*1*)
	public List<Advisor> org.springframework.aop.framework.autoproxy.BeanFactoryAdvisorRetrievalHelper.findAdvisorBeans() {
		// Determine list of advisor bean names, if not cached already.
		String[] advisorNames = null;
		synchronized (this) {
		    //先从缓存中获取，如果没有从BeanFactory中获取
			advisorNames = this.cachedAdvisorBeanNames;
			if (advisorNames == null) {
				// Do not initialize FactoryBeans here: We need to leave all regular beans
				// uninitialized to let the auto-proxy creator apply to them!
				//获取Advisor类型的所有bean，并且不允许提前初始化，这个方法逻辑我们在bean的创建章节中详细分析过
				//此处不再赘述
				advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
						this.beanFactory, Advisor.class, true, false);
				//添加到缓存
				this.cachedAdvisorBeanNames = advisorNames;
			}
		}
		if (advisorNames.length == 0) {
			return new LinkedList<Advisor>();
		}

		List<Advisor> advisors = new LinkedList<Advisor>();
		
		for (String name : advisorNames) {
		   //传统方式获取的advisor直接返回true
			if (isEligibleBean(name)) {
			    //如果当前bean正在创建，那么跳过
				if (this.beanFactory.isCurrentlyInCreation(name)) {
					if (logger.isDebugEnabled()) {
						logger.debug("Skipping currently created advisor '" + name + "'");
					}
				}
				else {
					try {
					    //通过beanName获取
						advisors.add(this.beanFactory.getBean(name, Advisor.class));
					}
					catch (BeanCreationException ex) {
						Throwable rootCause = ex.getMostSpecificCause();
						if (rootCause instanceof BeanCurrentlyInCreationException) {
							BeanCreationException bce = (BeanCreationException) rootCause;
							if (this.beanFactory.isCurrentlyInCreation(bce.getBeanName())) {
								if (logger.isDebugEnabled()) {
									logger.debug("Skipping advisor '" + name +
											"' with dependency on currently created bean: " + ex.getMessage());
								}
								// Ignore: indicates a reference back to the bean we're trying to advise.
								// We want to find advisors other than the currently created bean itself.
								continue;
							}
						}
						throw ex;
					}
				}
			}
		}
		return advisors;
	}
	
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;分析完传统获取Advisor的逻辑后，我继续分析一下注解获取的逻辑
</p>


```
public List<Advisor> org.springframework.aop.aspectj.annotation.BeanFactoryAspectJAdvisorsBuilder.buildAspectJAdvisors() {
		List<String> aspectNames = null;

		synchronized (this) {
		    //获取缓存的切面beanna
			aspectNames = this.aspectBeanNames;
			if (aspectNames == null) {
				List<Advisor> advisors = new LinkedList<Advisor>();
				aspectNames = new LinkedList<String>();
				//获取所有beanName
				String[] beanNames =
						BeanFactoryUtils.beanNamesForTypeIncludingAncestors(this.beanFactory, Object.class, true, false);
				for (String beanName : beanNames) {
				    //使用配置的includePatterns进行正则匹配beanName
		            //比如<aop:include name=".*"/>
					if (!isEligibleBean(beanName)) {
						continue;
					}
					// We must be careful not to instantiate beans eagerly as in this
					// case they would be cached by the Spring container but would not
					// have been weaved
					//获取指定beanName的bean类型，用户获取类型的bean如果是工厂bean，那么不会进行完全的初始化
					//把不完全初始化的单例bean进行缓存，以便完全初始化时使用
					Class<?> beanType = this.beanFactory.getType(beanName);
					if (beanType == null) {
						continue;
					}
					//判断这个bean是否存在@Aspect注解并且不是被AspectJ编译的类，被AspectJ编译的类在编译器就进行增强处理
					//spring不会再处理这样的类
					if (this.advisorFactory.isAspect(beanType)) {
					    //保存切面beanName
						aspectNames.add(beanName);
						//构建切面元数据，用于后去切面信息，比如切面类型，切面名，切面实例化模式，切面反射信息
						//具体构造逻辑请查看附录3-AspectMetadata
						AspectMetadata amd = new AspectMetadata(beanType, beanName);
						//单例模式
						if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
						    //创建切面工厂
						    //获取切面也只是调用BeanFactory的getBean，获取到之后会进行通知的解析
							MetadataAwareAspectInstanceFactory factory =
									new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
						    //解析当前切面，包装advice为advisor
							List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
							//如果当前切面是单例的，那么直接缓存已经解析的Advisor
							if (this.beanFactory.isSingleton(beanName)) {
								this.advisorsCache.put(beanName, classAdvisors);
							}
							//如果不是单例的，那么缓存其切面实例工厂
							else {
								this.aspectFactoryCache.put(beanName, factory);
							}
							advisors.addAll(classAdvisors);
						}
						else {
							// Per target or per this.
							//如果是pertarget和perthis模式的切面，那么它必须不能是单例的，否则报错
							if (this.beanFactory.isSingleton(beanName)) {
								throw new IllegalArgumentException("Bean with name '" + beanName +
										"' is a singleton, but aspect instantiation model is not singleton");
							}
							//创建原型切面实例工厂PrototypeAspectInstanceFactory，这个类继承了BeanFactoryAspectInstanceFactory
							//它与其父类不同的就是构造器里额外做了校验当前切面bean是否是原型声明周期，否则报错
							MetadataAwareAspectInstanceFactory factory =
									new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
							this.aspectFactoryCache.put(beanName, factory);
							//和上面解析单例是一样的操作，只是具体的切面工厂实现不同
							advisors.addAll(this.advisorFactory.getAdvisors(factory));
						}
					}
				}
				this.aspectBeanNames = aspectNames;
				return advisors;
			}
		}

		if (aspectNames.isEmpty()) {
			return Collections.emptyList();
		}
		List<Advisor> advisors = new LinkedList<Advisor>();
		//从缓存中获取
		for (String aspectName : aspectNames) {
			List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
			if (cachedAdvisors != null) {
				advisors.addAll(cachedAdvisors);
			}
			else {
			    //使用缓存的切面工厂构建advisor
				MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
				advisors.addAll(this.advisorFactory.getAdvisors(factory));
			}
		}
		return advisors;
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;获取advisor的大致的时序图如下
</p>

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/803a13269abf2c92e4245d295ef36dd9.png)
首先使用传统的方式到BeanFactory中获取advisor，然后通过@Aspect获取

<p>
&nbsp;&nbsp;&nbsp;&nbsp;上面spring仅仅是获取到了被@Aspect注解的bean，要想获取其具体的切点，通知信息，还需要进一步的处理。spring使用ReflectiveAspectJAdvisorFactory这个类去处理它们，那我们就来看看它是怎么解析的
</p>


```
public List<Advisor> ReflectiveAspectJAdvisorFactory.getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
        //获取切面的class对象
		Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
		//获取切面的beanName
		String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
		//校验当前bean是否是切面，并且其实例化模式不能是percflow，percflowbelow
		//因为spring aop目前不支持这两个模式
		validate(aspectClass);

		// We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
		// so that it will only instantiate once.
		//装饰传入进来的切面工厂，用于缓存实例化后的切面，以便减少多次实例化的开销
		MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
				new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

		List<Advisor> advisors = new LinkedList<Advisor>();
		//获取被@Before，@After等通知注解标注的方法，排除被@Pointcut标注的方法
		for (Method method : getAdvisorMethods(aspectClass)) {
		    //解析每个通知方法，构建advisor
		    //(*1*)
			Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}

		// If it's a per target aspect, emit the dummy instantiating aspect.
		//如果这个切面是pertarget或者perthis模式的，那么加入SyntheticInstantiationAdvisor通知器
		//内部设置了一个before通知，这个before通知的方法逻辑就是每次去调用切面工厂的getAspectInstance方法
		if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
			Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
			advisors.add(0, instantiationAdvisor);
		}

		// Find introduction fields.
		//查找引介增强通知，判断字段上是否被@DeclareParents注解标注
		//返回DeclareParentsAdvisor
		for (Field field : aspectClass.getDeclaredFields()) {
			Advisor advisor = getDeclareParentsAdvisor(field);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}

		return advisors;
	}
	
	
	//(*1*)
	public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
			int declarationOrderInAspect, String aspectName) {
        //校验当前bean是否为切面bean
		validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());
        
        //解析Before.class, Around.class, After.class, AfterReturning.class, AfterThrowing.class, Pointcut.class注解
        //获取其切点，然后包装成AspectJExpressionPointcut对象
		AspectJExpressionPointcut expressionPointcut = getPointcut(
				candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
		if (expressionPointcut == null) {
			return null;
		}
        //构建InstantiationModelAwarePointcutAdvisorImpl对象，这个类主要用于提供切面是否懒加载，切面名，通知，通知类型
		return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
				this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
	}
	
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;我们看到，spring把每个被标注了通知注解的方法包装成了InstantiationModelAwarePointcutAdvisorImpl通知器，值得注意的是它并不是简单的包装，在它的构造方法中做了一些特殊的处理
</p>


```
public InstantiationModelAwarePointcutAdvisorImpl(AspectJExpressionPointcut declaredPointcut,
			Method aspectJAdviceMethod, AspectJAdvisorFactory aspectJAdvisorFactory,
			MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {
        //定义的切点，也就是通知注解上面配置的切点
		this.declaredPointcut = declaredPointcut;
		//通知方法对象
		this.aspectJAdviceMethod = aspectJAdviceMethod;
		//用于解析通知的工厂
		this.aspectJAdvisorFactory = aspectJAdvisorFactory;
		//用于构建切面实例的工厂
		this.aspectInstanceFactory = aspectInstanceFactory;
		//通知定义的顺序
		this.declarationOrder = declarationOrder;
		//切面名称
		this.aspectName = aspectName;
        //如果切面实例化模式是perthis和pertarget的，那么可以懒加载
		if (aspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
			// Static part of the pointcut is a lazy type.
			//合并切面模式切点与通知定义切点，所谓切点合并就是构建一个UnionPointcut，然后内部以数组的形式过滤
			Pointcut preInstantiationPointcut = Pointcuts.union(
					aspectInstanceFactory.getAspectMetadata().getPerClausePointcut(), this.declaredPointcut);

			// Make it dynamic: must mutate from pre-instantiation to post-instantiation state.
			// If it's not a dynamic pointcut, it may be optimized out
			// by the Spring AOP infrastructure after the first evaluation.
			//构建pertarget实例模式切点
			this.pointcut = new PerTargetInstantiationModelPointcut(
					this.declaredPointcut, preInstantiationPointcut, aspectInstanceFactory);
			this.lazy = true;
		}
		else {
			// A singleton aspect.
			this.pointcut = this.declaredPointcut;
			this.lazy = false;
			//根据通知类型，定义对应通知实例
			//(*1*)
			this.instantiatedAdvice = instantiateAdvice(this.declaredPointcut);
		}
	}
	
	//(*1*)
	private Advice instantiateAdvice(AspectJExpressionPointcut pcut) {
	    //(*2*)
	    //通过切面通知者工厂获取通知
		return this.aspectJAdvisorFactory.getAdvice(this.aspectJAdviceMethod, pcut,
				this.aspectInstanceFactory, this.declarationOrder, this.aspectName);
	}
	
	 //(*2*)
	 public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
			MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {
        //获取切面的class对象
		Class<?> candidateAspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
		//校验
		validate(candidateAspectClass);
        //获取Before.class, Around.class, After.class, AfterReturning.class, AfterThrowing.class, Pointcut.class注解对象
        //使用AspectJAnnotation进行包装，解析对应通知类型和对应通知的切点
		AspectJAnnotation<?> aspectJAnnotation =
				AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
		if (aspectJAnnotation == null) {
			return null;
		}

		// If we get here, we know we have an AspectJ method.
		// Check that it's an AspectJ-annotated class
		if (!isAspect(candidateAspectClass)) {
			throw new AopConfigException("Advice must be declared inside an aspect type: " +
					"Offending method '" + candidateAdviceMethod + "' in class [" +
					candidateAspectClass.getName() + "]");
		}

		if (logger.isDebugEnabled()) {
			logger.debug("Found AspectJ method: " + candidateAdviceMethod);
		}

		AbstractAspectJAdvice springAdvice;
        //获取注解类型
		switch (aspectJAnnotation.getAnnotationType()) {
			case AtBefore:
			    //构建前置通知
				springAdvice = new AspectJMethodBeforeAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			case AtAfter:
			    //后置通知
				springAdvice = new AspectJAfterAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			case AtAfterReturning:
			    //返回后通知
				springAdvice = new AspectJAfterReturningAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
				if (StringUtils.hasText(afterReturningAnnotation.returning())) {
					springAdvice.setReturningName(afterReturningAnnotation.returning());
				}
				break;
			case AtAfterThrowing:
			    //抛出异常后通知
				springAdvice = new AspectJAfterThrowingAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
				if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
					springAdvice.setThrowingName(afterThrowingAnnotation.throwing());
				}
				break;
			case AtAround:
			    //环绕通知
				springAdvice = new AspectJAroundAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			case AtPointcut:
			    //如果是切点类型返回null
				if (logger.isDebugEnabled()) {
					logger.debug("Processing pointcut '" + candidateAdviceMethod.getName() + "'");
				}
				return null;
			default:
				throw new UnsupportedOperationException(
						"Unsupported advice type on method: " + candidateAdviceMethod);
		}

		// Now to configure the advice...
		//设置切面名称
		springAdvice.setAspectName(aspectName);
		//设置定义顺序
		springAdvice.setDeclarationOrder(declarationOrder);
		//获取参数名
		String[] argNames = this.parameterNameDiscoverer.getParameterNames(candidateAdviceMethod);
		if (argNames != null) {
			springAdvice.setArgumentNamesFromStringArray(argNames);
		}
		//绑定参数
		//(*3*)
		springAdvice.calculateArgumentBindings();
		return springAdvice;
	}
	
	//(*3*)
	public synchronized final void calculateArgumentBindings() {
		// The simple case... nothing to bind.
		if (this.argumentsIntrospected || this.adviceInvocationArgumentCount == 0) {
			return;
		}
        //获取通知方法的参数个数
		int numUnboundArgs = this.adviceInvocationArgumentCount;
		//获取通知方法参数类型数组
		Class<?>[] parameterTypes = this.aspectJAdviceMethod.getParameterTypes();
		//判断第一个参数是否为JoinPoint或者ProceedingJoinPoint，如果是那么未绑定参数减去一个
		if (maybeBindJoinPoint(parameterTypes[0]) || maybeBindProceedingJoinPoint(parameterTypes[0])) {
			numUnboundArgs--;
		}
		//判断第一个参数是否为JoinPoint.StaticPart
		else if (maybeBindJoinPointStaticPart(parameterTypes[0])) {
			numUnboundArgs--;
		}
        //如果还有未绑定的参数
		if (numUnboundArgs > 0) {
			// need to bind arguments by name as returned from the pointcut match
			bindArgumentsByName(numUnboundArgs);
		}
        //argumentsIntrospected 为true表示已经对参数进行了绑定过了
		this.argumentsIntrospected = true;
	}
	
	//(*4*)
	private void bindArgumentsByName(int numArgumentsExpectingToBind) {
		if (this.argumentNames == null) {
		    //获取通知方法参数名
			this.argumentNames = createParameterNameDiscoverer().getParameterNames(this.aspectJAdviceMethod);
		}
		if (this.argumentNames != null) {
			// We have been able to determine the arg names.
			//绑定确定的参数
			//(*5*)
			bindExplicitArguments(numArgumentsExpectingToBind);
		}
		else {
			throw new IllegalStateException("Advice method [" + this.aspectJAdviceMethod.getName() + "] " +
					"requires " + numArgumentsExpectingToBind + " arguments to be bound by name, but " +
					"the argument names were not specified and could not be discovered.");
		}
	}
	
	//(*5*)
	private void bindExplicitArguments(int numArgumentsLeftToBind) {
		this.argumentBindings = new HashMap<String, Integer>();
        //获取通知方法的参数个数
		int numExpectedArgumentNames = this.aspectJAdviceMethod.getParameterTypes().length;
		//如果参数名与通知方法参数长度不一致，报错
		if (this.argumentNames.length != numExpectedArgumentNames) {
			throw new IllegalStateException("Expecting to find " + numExpectedArgumentNames +
					" arguments to bind by name in advice, but actually found " +
					this.argumentNames.length + " arguments.");
		}

		// So we match in number...
		//绑定剩下的参数
		int argumentIndexOffset = this.adviceInvocationArgumentCount - numArgumentsLeftToBind;
		//获取剩下的参数
		for (int i = argumentIndexOffset; i < this.argumentNames.length; i++) {
			this.argumentBindings.put(this.argumentNames[i], i);
		}

		// Check that returning and throwing were in the argument names list if
		// specified, and find the discovered argument types.
		//如果返回参数名不为空，那么匹配
		if (this.returningName != null) {
			if (!this.argumentBindings.containsKey(this.returningName)) {
				throw new IllegalStateException("Returning argument name '" + this.returningName +
						"' was not bound in advice arguments");
			}
			else {
			    //根据返回参数名称返回参数下标
				Integer index = this.argumentBindings.get(this.returningName);
				//获取返回参数类型
				this.discoveredReturningType = this.aspectJAdviceMethod.getParameterTypes()[index];
				//获取返回参数参数化类型
				this.discoveredReturningGenericType = this.aspectJAdviceMethod.getGenericParameterTypes()[index];
			}
		}
		//抛出异常参数名
		if (this.throwingName != null) {
			if (!this.argumentBindings.containsKey(this.throwingName)) {
				throw new IllegalStateException("Throwing argument name '" + this.throwingName +
						"' was not bound in advice arguments");
			}
			else {
			    //获取异常参数名下标
				Integer index = this.argumentBindings.get(this.throwingName);
				//获取抛出异常类型
				this.discoveredThrowingType = this.aspectJAdviceMethod.getParameterTypes()[index];
			}
		}

		// configure the pointcut expression accordingly.
		//(*6*)
		configurePointcutParameters(argumentIndexOffset);
	}
	
	//(*6*)
	private void configurePointcutParameters(int argumentIndexOffset) {
	    //为绑定参数下标偏移
		int numParametersToRemove = argumentIndexOffset;
		//如果有返回值参数，说明前面已经进行了绑定，参数移除项加1
		if (this.returningName != null) {
			numParametersToRemove++;
		}
	    //如果有异常参数名，说明前面已经进行了绑定，参数移除项加1
		if (this.throwingName != null) {
			numParametersToRemove++;
		}
		//绑定切点参数名
		String[] pointcutParameterNames = new String[this.argumentNames.length - numParametersToRemove];
		Class<?>[] pointcutParameterTypes = new Class<?>[pointcutParameterNames.length];
		Class<?>[] methodParameterTypes = this.aspectJAdviceMethod.getParameterTypes();

		int index = 0;
		for (int i = 0; i < this.argumentNames.length; i++) {
			if (i < argumentIndexOffset) {
				continue;
			}
			//跳过返回参数和抛出异常参数
			if (this.argumentNames[i].equals(this.returningName) ||
				this.argumentNames[i].equals(this.throwingName)) {
				continue;
			}
			//绑定剩下的参数名
			pointcutParameterNames[index] = this.argumentNames[i];
			//绑定剩下的参数类型
			pointcutParameterTypes[index] = methodParameterTypes[i];
			index++;
		}
        //缓存切点参数名
		this.pointcut.setParameterNames(pointcutParameterNames);
		//缓存切点参数类型
		this.pointcut.setParameterTypes(pointcutParameterTypes);
	}
	
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;以下是通过注解解析获取advisor的时序图
</p>

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/876c00ef9ada1fa655a93d6913cb03d9.png)





























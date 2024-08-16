
<html>
<p>
&nbsp;&nbsp;&nbsp;&nbsp;通常我们需要开启诸如@Component，@Service之类的注解，会在xml中配置一段这样的标签<context:component-scan base-package="com.zhipin.service,com.zhipin.aop"></context:component-scan>
这样我们就可以在指定的包下面使用注解的方式构建bean了，context是自定义的一个命名空间，需要使用自定的方式进行解析，前面我们分析了自定义标签的解析，我们知道spring要想解析自定标签就必须通过一个命名空间解析器解析才能够处理指定命名空间下定义的标签，以下就是解析context命名空间的解析器。
</p>


```
public class ContextNamespaceHandler extends NamespaceHandlerSupport {

	@Override
	public void init() {
	    //以下就是context命名空间下的标签处理器
		registerBeanDefinitionParser("property-placeholder", new PropertyPlaceholderBeanDefinitionParser());
		registerBeanDefinitionParser("property-override", new PropertyOverrideBeanDefinitionParser());
		registerBeanDefinitionParser("annotation-config", new AnnotationConfigBeanDefinitionParser());
		registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
		registerBeanDefinitionParser("load-time-weaver", new LoadTimeWeaverBeanDefinitionParser());
		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
		registerBeanDefinitionParser("mbean-export", new MBeanExportBeanDefinitionParser());
		registerBeanDefinitionParser("mbean-server", new MBeanServerBeanDefinitionParser());
	}

}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;上面定义了context命名空间下标签处理器，本节主要研究component-scan标签，可以看到处理这个标签创建了ComponentScanBeanDefinitionParser处理器。
</p>


```
public BeanDefinition org.springframework.context.annotation.ComponentScanBeanDefinitionParser.parse(Element element, ParserContext parserContext) {
        //解析base-package
		String basePackage = element.getAttribute(BASE_PACKAGE_ATTRIBUTE);
		//解析占位符
		basePackage = parserContext.getReaderContext().getEnvironment().resolvePlaceholders(basePackage);
		//",; \t\n"分割
		String[] basePackages = StringUtils.tokenizeToStringArray(basePackage,
				ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);

		// Actually scan for bean definitions and register them.
		//创建扫描器，扫描指定包下的class类
		ClassPathBeanDefinitionScanner scanner = configureScanner(parserContext, element);
		//扫描BeanDefinition
		Set<BeanDefinitionHolder> beanDefinitions = scanner.doScan(basePackages);
		registerComponents(parserContext.getReaderContext(), beanDefinitions, element);

		return null;
	}

```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;配置扫描器
</p>

```
protected ClassPathBeanDefinitionScanner configureScanner(ParserContext parserContext, Element element) {
        //是否使用默认的过滤方式
		boolean useDefaultFilters = true;
		if (element.hasAttribute(USE_DEFAULT_FILTERS_ATTRIBUTE)) {
			useDefaultFilters = Boolean.valueOf(element.getAttribute(USE_DEFAULT_FILTERS_ATTRIBUTE));
		}

		// Delegate bean definition registration to scanner class.
		//new ClassPathBeanDefinitionScanner(readerContext.getRegistry(), useDefaultFilters);
		ClassPathBeanDefinitionScanner scanner = createScanner(parserContext.getReaderContext(), useDefaultFilters);
		//设置资源加载器
		scanner.setResourceLoader(parserContext.getReaderContext().getResourceLoader());
		//设置环境
		scanner.setEnvironment(parserContext.getReaderContext().getEnvironment());
		//设置默认的BeanDefinition属性
		scanner.setBeanDefinitionDefaults(parserContext.getDelegate().getBeanDefinitionDefaults());
		//设置自动装配候选模式
		scanner.setAutowireCandidatePatterns(parserContext.getDelegate().getAutowireCandidatePatterns());
        //设置资源加载resource-pattern模式
		if (element.hasAttribute(RESOURCE_PATTERN_ATTRIBUTE)) {
			scanner.setResourcePattern(element.getAttribute(RESOURCE_PATTERN_ATTRIBUTE));
		}

		try {
		    //从元素中解析name-generator属性，给扫描器设置beanName生成器
		    //所以我们可以实现自己的命名解析器
			parseBeanNameGenerator(element, scanner);
		}
		catch (Exception ex) {
			parserContext.getReaderContext().error(ex.getMessage(), parserContext.extractSource(element), ex.getCause());
		}

		try {
		    //解析scope，分为两种，普通的scope，比如singleton这类
		    //还有一种是scopeProxy，分别是接口（JDK动态代理），目标类（cglib动态代理）,NO（不做任何事情）
			parseScope(element, scanner);
		}
		catch (Exception ex) {
			parserContext.getReaderContext().error(ex.getMessage(), parserContext.extractSource(element), ex.getCause());
		}
        //解析include-filter标签，创建类型过滤器
		parseTypeFilters(element, scanner, parserContext);

		return scanner;
	}
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;在继续往下研究之前，我们先来看下ClassPathBeanDefinitionScanner的类图
</p>

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d2dae01ca20ae873334202082ea0909f.png)

1. Aware 标记接口

2. EnvironmentCapable 定义获取Environment的能力

3. ResourceLoaderAware 定义设置ResourceLoader通知

4. ClassPathScanningCandidateComponentProvider 扫描组件提供者，比如定义@Component扫描组件（默认关联PathMatchingResourcePatternResolver，CachingMetadataReaderFactory）

5. ClassPathBeanDefinitionScanner 提供BeanDefinition的扫描能力 （默认关联AnnotationBeanNameGenerator[]，AnnotationScopeMetadataResolver[会解析@Scope注解]的实例）

6. ConditionEvaluator 条件执行器，由于过滤条件，对应使用的注解@Condition

7. ConditionContextImpl 条件上下文，持有注册器，环境，资源加载器，bean工厂

<p>
&nbsp;&nbsp;&nbsp;&nbsp;其实在创建扫描器的时候，在构造器中做了以下的事情
</p>


```

protected void org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider.registerDefaultFilters() {
        //添加@Component注解过滤器
		this.includeFilters.add(new AnnotationTypeFilter(Component.class));
		ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
		try {
		    //添加@ManagedBean注解，这种注解用于jsf，不过国内很少企业还会使用这种框架
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
			logger.debug("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
		}
		try {
		    //添加@Named注解
			this.includeFilters.add(new AnnotationTypeFilter(
					((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
			logger.debug("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
		}
		catch (ClassNotFoundException ex) {
			// JSR-330 API not available - simply skip.
		}
	}
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;创建用户指定过滤类型
</p>

```
protected TypeFilter createTypeFilter(Element element, ClassLoader classLoader, ParserContext parserContext) {
        //解析type属性
		String filterType = element.getAttribute(FILTER_TYPE_ATTRIBUTE);
		//解析expression表达式
		String expression = element.getAttribute(FILTER_EXPRESSION_ATTRIBUTE);
		//解析占位符
		expression = parserContext.getReaderContext().getEnvironment().resolvePlaceholders(expression);
		try {
		    //如果type指定是注解类型，那么创建AnnotationTypeFilter，匹配注解
			if ("annotation".equals(filterType)) {
				return new AnnotationTypeFilter((Class<Annotation>) classLoader.loadClass(expression));
			}
			//如果type指定assignable，创建AssignableTypeFilter，匹配指定子类
			else if ("assignable".equals(filterType)) {
				return new AssignableTypeFilter(classLoader.loadClass(expression));
			}
			//如果type为aspectj，创建AspectJTypeFilter，使用切面表达式匹配模式符匹配
			else if ("aspectj".equals(filterType)) {
				return new AspectJTypeFilter(expression, classLoader);
			}
			//使用正则表达式匹配，我们tvtaoadmin的项目似乎很钟情于这种方式
			else if ("regex".equals(filterType)) {
				return new RegexPatternTypeFilter(Pattern.compile(expression));
			}
			//这种属于自定义类型过滤器，我们只要实现TypeFilter即可
			else if ("custom".equals(filterType)) {
				Class<?> filterClass = classLoader.loadClass(expression);
				if (!TypeFilter.class.isAssignableFrom(filterClass)) {
					throw new IllegalArgumentException(
							"Class is not assignable to [" + TypeFilter.class.getName() + "]: " + expression);
				}
				return (TypeFilter) BeanUtils.instantiateClass(filterClass);
			}
			else {
				throw new IllegalArgumentException("Unsupported filter type: " + filterType);
			}
		}
		catch (ClassNotFoundException ex) {
			throw new FatalBeanException("Type filter class not found: " + expression, ex);
		}
	}
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;既然提到了TypeFilter，那么我们来看下它的类图
</p>

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/91f9c6924efc651b8aa4793405f73cff.png)

1. TypeFilter 定义匹配某资源的能力

2. CustomTypeFilter 自定义类型过滤器，匹配包含Kustom的类名

3. AspectJTypeFilter 使用切面表达式匹配模式

4. AbstractTypeHierarchyTraversingFilter 这是模板类，AnnotationTypeFilter与AssignableTypeFilter都是有向上层匹配的过程，这个类将这些能够通用的代码抽离，并使用犹如逻辑运算中短路的形式进行匹配。

5. AbstractClassTestingTypeFilter 也定义模板方法

6. RegexPatternTypeFilter 正则匹配

<p>
&nbsp;&nbsp;&nbsp;&nbsp;好了，到了这里，一个ClassPathBeanDefinitionScanner扫描器就配置好了，接下来就看看它是怎么扫描的。
</p>


```
protected Set<BeanDefinitionHolder> org.springframework.context.annotation.ClassPathBeanDefinitionScanner.doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<BeanDefinitionHolder>();
		for (String basePackage : basePackages) {
		    //查找候选的bean
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			for (BeanDefinition candidate : candidates) {
			    //使用scope解析器解析scope，默认是注解scope解析器，会解析类上的@Scope注解
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				candidate.setScope(scopeMetadata.getScopeName());
				//使用beanName生成器生成，默认是注解beanName生成器，会读取诸如@Component上的value属性，如果没有指定value属性值，那么就使用类名的，第一个字母小写，不会截取$$后面的字符，如果是一个$的，那么替换为点
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				//如果它是AbstractBeanDefinition类型的，也就是用于表示一个bean的BeanDefinition（因为还有其他的不是表示bean的，比如PropertyValue这个是表示某个属性名，对应某个值）
				if (candidate instanceof AbstractBeanDefinition) {
				    //将默认属性设置进去，并判断它是否用于自动装配候选
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
				//处理注解解析的BeanDefinition
				if (candidate instanceof AnnotatedBeanDefinition) {
			       //处理注解，比如从@Lazy注解获取加载策略，@Primary表示是否为master bean	
    		   	AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
				//检查这个BeanDefinition是否已经存在，如果不存在，那么返回true，存在就判断是否来自同一个文件，如果不是，那么就表示产生了冲突，会报错，如果是，那么返回false，表示已经处理过了
				if (checkCandidate(beanName, candidate)) {
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
					definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}
		}
		return beanDefinitions;
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;上面是对寻找到的Bean进行进一步处理，装饰，以下才是真正查找bean的过程。
</p>

```

public Set<BeanDefinition> org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider.findCandidateComponents(String basePackage) {
		Set<BeanDefinition> candidates = new LinkedHashSet<BeanDefinition>();
		try {
		    //解析占位符，拼接spring自定义的classpath*:协议，后面拼接Ant匹配模式的**/*.class，这个模式我们可以在标签的resource-pattern属性进行手动指定，默认是**/*.class
			String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
					resolveBasePackage(basePackage) + "/" + this.resourcePattern;
			//这里和分析spring查找xml配置文件的时候是一样的逻辑，只不过是把后缀改成.class罢了，不再赘述
			Resource[] resources = this.resourcePatternResolver.getResources(packageSearchPath);
			boolean traceEnabled = logger.isTraceEnabled();
			boolean debugEnabled = logger.isDebugEnabled();
			for (Resource resource : resources) {
				if (traceEnabled) {
					logger.trace("Scanning " + resource);
				}
				if (resource.isReadable()) {
					try {
					    //构造元素数据读取器，内部使用ASM字节码框架实现字节码的读取，不会对类进行加载。
						MetadataReader metadataReader = this.metadataReaderFactory.getMetadataReader(resource);
						//判断是否是我们想要的bean，我们在上面已经准备好了TypeFilter，这些TypeFilter将会在这里使用，除此之外，内部还会@Conditional注解进行处理，也就是按条件按阶段加载判断是否加载bean
						if (isCandidateComponent(metadataReader)) {
						    //创建ScannedGenericBeanDefinition，这个类实现了AnnotationBeanDefinition，继承了GenericBeanDefinition，具有获取类注解的能力
							ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
							//保存资源，告诉自己我从哪里来
							sbd.setResource(resource);
							//保存资源，告诉自己我从哪里来
							sbd.setSource(resource);
							//判断这个类是否为具体类（不是抽象类或接口）和独立的类（可以独立构造，一个单独的类，或者是一个静态嵌套类）
							if (isCandidateComponent(sbd)) {
								if (debugEnabled) {
									logger.debug("Identified candidate component class: " + resource);
								}
								candidates.add(sbd);
							}
						。。。。。。省略部分代码
		return candidates;
	}


```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;上面我们在扫描类的时候，看到了MetadataReader和MetadataReaderFactory类，很明显MetadataReaderFactory是构建MetadataReader的工厂类，目前spring4.2.6中实现了MetadataReaderFactory的工厂是SimpleMetadataReaderFactory及其子类CachingMetadataReaderFactory，实现了MetadataReader接口的只有SimpleMetadataReader，下面看下类图
</p>

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/ef688479b30119ffacbfa2c1b08ef1ea.png)
1.  ClassMetadata 定义获取类名，成员类，获取超类等能力

2. AnnotatedTypeMetadata 定义获取注解的属性，

3. AnnotationMetadata 定义获取注解的类型名，指定注解的元注解的属性，获取被指定注解标注的方法

4. ClassVisitor 定义如何参观一个类

5. MetadataReaderFactory 定义创建MetadataReader，目前只创建SimpleMetadataReader

6. MetadataReader 定义读取类信息ClassMetadata，注解信息AnnotationMetadata的能力

<p>
&nbsp;&nbsp;&nbsp;&nbsp;ClassVisitor是一个参观者，spring在这里使用了参观者模式，它被ClassReader接受。使用ClassVisitor可以操作类的数据，比如解析注解，方法，成员变量等。如果大家对ASM框架感兴趣，可以到https://asm.ow2.io/官网学习，在学习ASM框架前，最后了解一下字节码文件的格式和jvm指令，咱们继续，等会我们会挑选一个TypeFilter来查看它是怎么匹配的，比如AnnotationTypeFilter，这个类主要处理我们非常熟悉的@Component
</p>


```
protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
        //首先查看这个类是否是排除的，如果是将不会加载为BeanDefinition
		for (TypeFilter tf : this.excludeFilters) {
			if (tf.match(metadataReader, this.metadataReaderFactory)) {
				return false;
			}
		}
		//查看这个类是否是我们需要的类
		for (TypeFilter tf : this.includeFilters) {
			if (tf.match(metadataReader, this.metadataReaderFactory)) {
			    //使用条件注解继续判断当前类是否符合条件
				return isConditionMatch(metadataReader);
			}
		}
		return false;
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;AnnotationTypeFilter匹配，这个类继承了AbstractTypeHierarchyTraversingFilter，这个类的match方法综合了AssignableTypeFilter，AnnotationTypeFilter两个类的匹配功能，使用类似逻辑或的短路操作
</p>


```
public boolean org.springframework.core.type.filter.AbstractTypeHierarchyTraversingFilter.match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory)
			throws IOException {

		// This method optimizes avoiding unnecessary creation of ClassReaders
		// as well as visiting over those readers.
		//首先匹配自己的类上的注解，以及注解的元注解是否匹配
		//元注解（比如@Service这个注解，实际上在扫描器的includeFilters这个集合里并不存在对应的TypeFilter）
		//那么它是怎么匹配上的呢？那是因为这个@Service注解上标有@Component注解，而这个注解被称为元注解
		if (matchSelf(metadataReader)) {
			return true;
		}
		//如果这是一个AssignableTypeFilter实例，那么上面直接返回false
		//判断设置的类是否为当前类或者是它的父类
		ClassMetadata metadata = metadataReader.getClassMetadata();
		if (matchClassName(metadata.getClassName())) {
			return true;
		}
        //如果考虑继承，比如注解上存在@Inherited元注解，如果存在，那么这里为true
		if (this.considerInherited) {
		    //扫描超类，如果还是没有找到，就会继续向上
			if (metadata.hasSuperClass()) {
				// Optimization to avoid creating ClassReader for super class.
				Boolean superClassMatch = matchSuperClass(metadata.getSuperClassName());
				if (superClassMatch != null) {
					if (superClassMatch.booleanValue()) {
						return true;
					}
				}
				else {
					// Need to read super class to determine a match...
					try {
						if (match(metadata.getSuperClassName(), metadataReaderFactory)) {
							return true;
						}
					}
					catch (IOException ex) {
						logger.debug("Could not read super class [" + metadata.getSuperClassName() +
								"] of type-filtered class [" + metadata.getClassName() + "]");
					}
 				}
			}
		}
        //如果考虑接口，那么去匹配接口，如果父接口没有匹配到，那么继续向上。
		if (this.considerInterfaces) {
			for (String ifc : metadata.getInterfaceNames()) {
				// Optimization to avoid creating ClassReader for super class
				Boolean interfaceMatch = matchInterface(ifc);
				if (interfaceMatch != null) {
					if (interfaceMatch.booleanValue()) {
						return true;
					}
				}
				else {
					// Need to read interface to determine a match...
					try {
						if (match(ifc, metadataReaderFactory)) {
							return true;
						}
					}
					catch (IOException ex) {
						logger.debug("Could not read interface [" + ifc + "] for type-filtered class [" +
								metadata.getClassName() + "]");
					}
				}
			}
		}

		return false;
	}
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;即使类匹配到了我们设置的TypeFilter，但这还不算完，这个类它自己有权利拒绝被加载，想要拒绝被加载的类上面会有@Conditional注解，这个注解告诉spring我必须满足什么样的条件我才能被你加载。
</p>


```
private boolean isConditionMatch(MetadataReader metadataReader) {
		if (this.conditionEvaluator == null) {
			this.conditionEvaluator = new ConditionEvaluator(getRegistry(), getEnvironment(), getResourceLoader());
		}
		return !this.conditionEvaluator.shouldSkip(metadataReader.getAnnotationMetadata());
	}
	
	
	public boolean shouldSkip(AnnotatedTypeMetadata metadata, ConfigurationPhase phase) {
	   //如果当前类的注解上不包含@Conditional，那么直接返回false
		if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
			return false;
		}

		if (phase == null) {
		    //如果当前类的注解包含@Configuration或者包含@Component，@ComponentScan，@Import，@ImportResource中的一个，或者是对应的方法上有@Bean注解，那么认为当前为配置阶段，否则为注册bean阶段，很显然我们现在是在配置阶段
			if (metadata instanceof AnnotationMetadata &&
					ConfigurationClassUtils.isConfigurationCandidate((AnnotationMetadata) metadata)) {
				return shouldSkip(metadata, ConfigurationPhase.PARSE_CONFIGURATION);
			}
			return shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN);
		}

		List<Condition> conditions = new ArrayList<Condition>();
		//从所有的注解@Conditional注解上获取他们的value属性列表，这个列表是实现了Condition接口的class类
		for (String[] conditionClasses : getConditionClasses(metadata)) {
			for (String conditionClass : conditionClasses) {
			    //通过反射的方式构建这个对象
				Condition condition = getCondition(conditionClass, this.context.getClassLoader());
				conditions.add(condition);
			}
		}
        
        //如果实现了优先级接口，按优先级在前排序，如果不是按实现的order接口的order值，如果也没有实现order，如果传入的是类那么判断是否存在@Order注解，如果没有，再判断@Priority，是方法也会判断是否有@Order
		AnnotationAwareOrderComparator.sort(conditions);
        
		for (Condition condition : conditions) {
			ConfigurationPhase requiredPhase = null;
			if (condition instanceof ConfigurationCondition) {
				requiredPhase = ((ConfigurationCondition) condition).getConfigurationPhase();
			}
			//如果对阶段有要求，那么在对应的阶段（比如配置阶段）才能配置当前bean
			if (requiredPhase == null || requiredPhase == phase) {
				if (!condition.matches(this.context, metadata)) {
					return true;
				}
			}
		}

		return false;
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;通过以上的步骤，我们将我们需要的bean都获取到了，那么接下就是对每个扫描出来的bean进行进一步的配置，比如scope，lazy等属性，对于scope这个属性，如果我们没有进行特殊的配置的话，那么spring默认是AnnotationScopeMetadataResolver这个解析器，我们来大致看下这个类是如果进行解析
</p>


```
public ScopeMetadata resolveScopeMetadata(BeanDefinition definition) {
        //用于寄存解析出来的scope和scope代理模式
		ScopeMetadata metadata = new ScopeMetadata();
		//如果是使用了注解的BeanDefinition，那么就会进行操作
		if (definition instanceof AnnotatedBeanDefinition) {
			AnnotatedBeanDefinition annDef = (AnnotatedBeanDefinition) definition;
			//获取当前注解类型的所有属性值，这里面涉及了spring的注解继承原则
			//spring将元注解理解为当前注解的父注解，也就是说子注解可以重写父注解的方法
			//比如@Service的元注解@Component就是@Service的父注解，出了value属性以外，其他名字的属性都能被@Service重写
			AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(annDef.getMetadata(), this.scopeAnnotationType);
			if (attributes != null) {
			    //从@Scope注解中获取scope，这里需要注意一下别名，别名使用@AliasFor注解指定，所以在@Scope中我们可以设置value属性，也可以设置scopeName属性
				metadata.setScopeName(attributes.getAliasedString("value", this.scopeAnnotationType, definition.getSource()));
				//设置生命周期代理模式，像Web应用中有Request范围
				//如果一个单例的Bean引用了一个Request范围的bean
				//那该怎么办，那么只能通过代理的方式，才能保证每次调用
				//方法获取的值是request范围bean提供的。
				ScopedProxyMode proxyMode = attributes.getEnum("proxyMode");
				if (proxyMode == null || proxyMode == ScopedProxyMode.DEFAULT) {
					proxyMode = this.defaultProxyMode;
				}
				metadata.setScopedProxyMode(proxyMode);
			}
		}
		return metadata;
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;上面提到了@AliasFor这个注解，这个注解有一下几个功能

</p>

- 设置别名
    - 互为别名对的两个属性他们必须要有默认值，且默认值要一样
    - 互为别名的两个属性不能同时指定值
- 用于指定子注解覆盖父注解的某个属性

<p>
&nbsp;&nbsp;&nbsp;&nbsp;设置完scope之后，spring为这个这些BeanDefinition生成beanName，在没有指定的情况下，使用的是AnnotationBeanNameGenerator生成器，这个生成器，首先通过注解上的value来获取值，如果没有就以bean的类名，首个字母小写的作为beanName，如果是连续两个大写的，比如URL...这种，那么原样返回，像RecommendService这种，就会变成recommendService。生成beanName之后对BeanDefinition进行了一些默认属性的设置，@Lazy，@Primary，@DependsOn等通用注解的获取。比较简单，所以这里不对它们进行分析。到了这里还是不能马上将BeanDefinition注册到BeanFactory中，首先spring需要对这个BeanDefinition进行进一步的检查。
</p>


```

protected boolean org.springframework.context.annotation.ClassPathBeanDefinitionScanner.checkCandidate(String beanName, BeanDefinition beanDefinition) throws IllegalStateException {
        //如果这个bean还未被注册过，那么返回，进行装饰，注册
		if (!this.registry.containsBeanDefinition(beanName)) {
			return true;
		}
		//如果已经存在一个同名的BeanDefinition了
		BeanDefinition existingDef = this.registry.getBeanDefinition(beanName);
		BeanDefinition originatingDef = existingDef.getOriginatingBeanDefinition();
		//判断是否被装饰过，如果被装饰过，那么获取最原始的BeanDefinition
		if (originatingDef != null) {
			existingDef = originatingDef;
		}
		//判断是否是兼容的
		if (isCompatible(beanDefinition, existingDef)) {
			return false;
		}
		throw new ConflictingBeanDefinitionException("Annotation-specified bean name '" + beanName +
				"' for bean class [" + beanDefinition.getBeanClassName() + "] conflicts with existing, " +
				"non-compatible bean definition of same name and class [" + existingDef.getBeanClassName() + "]");
	}
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;以上有段代码判断当前BeanDefinition是否和已经注册过的BeanDefinition兼容的代码，spring将其分为三种情况，只要满足一个就视为兼容：

1、已经注册的bean不是扫描bean，也就是在配置文件中明确指定的BeanDefinition

2、扫描的bean是同一个资源

3、两者的BeanDefinition是equals的

&nbsp;&nbsp;&nbsp;&nbsp;如果需要进行注册，那么首先需要对当前的BeanDefinition进行一次判断，如果有设置生命周期代理模式，那么就需要对当前BeanDefinition进行装饰。

</p>


```
public static BeanDefinitionHolder org.springframework.aop.scope.ScopedProxyUtils.createScopedProxy(BeanDefinitionHolder definition,
			BeanDefinitionRegistry registry, boolean proxyTargetClass) {

		String originalBeanName = definition.getBeanName();
		BeanDefinition targetDefinition = definition.getBeanDefinition();
		//给当前的beanName加上scopedTarget.前缀
		String targetBeanName = getTargetBeanName(originalBeanName);

		// Create a scoped proxy definition for the original bean name,
		// "hiding" the target bean in an internal target definition.
		//对当前BeanDefinition进行装饰，ScopedProxyFactoryBean是一个实现了FactoryBean接口的类
		RootBeanDefinition proxyDefinition = new RootBeanDefinition(ScopedProxyFactoryBean.class);
		//设置被装饰的bean定义
		proxyDefinition.setDecoratedDefinition(new BeanDefinitionHolder(targetDefinition, targetBeanName));
		//记录最原始的BeanDefinition
		proxyDefinition.setOriginatingBeanDefinition(targetDefinition);
		proxyDefinition.setSource(definition.getSource());
		proxyDefinition.setRole(targetDefinition.getRole());
        //添加targetBeanName属性
		proxyDefinition.getPropertyValues().add("targetBeanName", targetBeanName);
		//如果使用cglib代理，给原始bean设置cglib代理标识
		if (proxyTargetClass) {
			targetDefinition.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
			// ScopedProxyFactoryBean's "proxyTargetClass" default is TRUE, so we don't need to set it explicitly here.
		}
		else {
			proxyDefinition.getPropertyValues().add("proxyTargetClass", Boolean.FALSE);
		}

		// Copy autowire settings from original bean definition.
		//将原始bean的值设置到装饰bean中去
		proxyDefinition.setAutowireCandidate(targetDefinition.isAutowireCandidate());
		//设置primary属性
		proxyDefinition.setPrimary(targetDefinition.isPrimary());
		//设置qualified属性
		if (targetDefinition instanceof AbstractBeanDefinition) {
			proxyDefinition.copyQualifiersFrom((AbstractBeanDefinition) targetDefinition);
		}

		// The target bean should be ignored in favor of the scoped proxy.
		//如果这个bean定义被代理了，那么原始bean将不会再作为候选的bean提供给其他的bean注入
		targetDefinition.setAutowireCandidate(false);
		targetDefinition.setPrimary(false);

		// Register the target bean as separate bean in the factory.
		//使用带有代理前缀的beanName进行注册
		registry.registerBeanDefinition(targetBeanName, targetDefinition);

		// Return the scoped proxy definition as primary bean definition
		// (potentially an inner bean).
		//将装饰BeanDefinition作为主bean返回
		return new BeanDefinitionHolder(proxyDefinition, originalBeanName, definition.getAliases());
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;如果不需要装饰就原样返回，如果要装饰就进行装饰，然后一切都已经准备就绪，可以进行注册了。
</p>


```

public void org.springframework.beans.factory.support.DefaultListableBeanFactory.registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

		Assert.hasText(beanName, "Bean name must not be empty");
		Assert.notNull(beanDefinition, "BeanDefinition must not be null");

		if (beanDefinition instanceof AbstractBeanDefinition) {
			try {
			    //进行校验，不能同时出现工厂方法和方法覆写代理方法
				((AbstractBeanDefinition) beanDefinition).validate();
			}
			catch (BeanDefinitionValidationException ex) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Validation of bean definition failed", ex);
			}
		}

		BeanDefinition oldBeanDefinition;

		oldBeanDefinition = this.beanDefinitionMap.get(beanName);
		if (oldBeanDefinition != null) {
		    //如果不允许BeanDefinition的覆盖，那么会报错，默认是运行的
			if (!isAllowBeanDefinitionOverriding()) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
						"': There is already [" + oldBeanDefinition + "] bound.");
			}
			。。。。。。省略部分代码
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
		else {
		    //如果bean已经开始创建，说明此时已经是bean注册阶段，判断的依据为alreadyCreated不为空
			if (hasBeanCreationStarted()) {
				// Cannot modify startup-time collection elements anymore (for stable iteration)
				//加锁，bean的创建注册阶段会有并发（tomcat在启动context的时候是以一个线程池处理每个context的，此时不会有并发问题，但是如果tomcat已经完全启动，那么将会有并发问题）
				synchronized (this.beanDefinitionMap) {
					this.beanDefinitionMap.put(beanName, beanDefinition);
					List<String> updatedDefinitions = new ArrayList<String>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					//更新beanDefinitionNames集合
					this.beanDefinitionNames = updatedDefinitions;
					//如果记录已注册单例bean的beanName的集合已经包含这个bean的beanName，那么移除它，这个manualSingletonNames目前我据我了解，它专门用于保存那么spring自己注册的单例bean，他们没有对应的BeanDefinition
					if (this.manualSingletonNames.contains(beanName)) {
						Set<String> updatedSingletons = new LinkedHashSet<String>(this.manualSingletonNames);
						updatedSingletons.remove(beanName);
						this.manualSingletonNames = updatedSingletons;
					}
				}
			}
			else {
				// Still in startup registration phase
				this.beanDefinitionMap.put(beanName, beanDefinition);
				this.beanDefinitionNames.add(beanName);
				this.manualSingletonNames.remove(beanName);
			}
			this.frozenBeanDefinitionNames = null;
		}
        //移除与老的BeanDefinition相关联的数据，比如合并过的BeanDefinition，注册的bean，依赖，最后还会调用销毁方法销毁这些对象
		if (oldBeanDefinition != null || containsSingleton(beanName)) {
			resetBeanDefinition(beanName);
		}
	}

```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;以上就是我们配置的注解扫描过程，但这里已经结束了，但是spring额外还做了一些事情。调用AnnotationConfigUtils.registerAnnotationConfigProcessors注册注解配置
</p>


```
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, Object source) {
        
        //获取BeanFactory，如果是applicationContext，那么getBeanFactory
		DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
		if (beanFactory != null) {
			if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
			    //设置依赖比较器，在获取到集合依赖时进行排序
				beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
			}
			//设置自动装配解析器，用于检测是否是候选bean，构造懒加载代理bean（比如某个eager的bean依赖了一个懒加载的bean），解析@Value注解
			if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
				beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
			}
		}

		Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<BeanDefinitionHolder>(4);

		if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
		    //注册ConfigurationClassPostProcessor //BeanDefinition后置处理器,实现了BeanDefinitionRegistryPostProcessor
		    //在加载完配置文件后会调用，此时我们可以继续配置BeanDefinition
		    //这里用于解析@Configuration注解
			RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
		    //注册AutowiredAnnotationBeanPostProcessor后置处理器
		    //这个后置处理器用于处理@Autowire和@Value以及@Inject
			RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		if (!registry.containsBeanDefinition(REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
		    //处理@Required
			RootBeanDefinition def = new RootBeanDefinition(RequiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		// Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
		//处理@WebServiceRef, @EJB, @Resource，jsr250Present这个变量是通过检测@Resource注解是否存在
		if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		// Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
		//添加JPA支持，判断是否存在@EntityManagerFactory和PersistenceAnnotationBeanPostProcessor
		if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition();
			try {
				def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
						AnnotationConfigUtils.class.getClassLoader()));
			}
			catch (ClassNotFoundException ex) {
				throw new IllegalStateException(
						"Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
			}
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
		}
        //用于@EventListener的解析，使用ApplicationListenerMethodAdapter适配器进行包装
		if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
		}
		//定义创建ApplicationListenerMethodAdapter监听器
		if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
		}

		return beanDefs;
	}
```
<p>
&nbsp;&nbsp;&nbsp;&nbsp;好了，@Component的扫描先告一段落了。
</p>

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/a2535768faef9ff7dff2b717c67deed8.png)

总结：首先通过自定义标签的命名空间获取到处理这个命名空间的处理器，然后通过命名空间找到对应标签元素的解析器进行解析，获取元素标签上设置的基路径，加载对应目录下的类资源，使用ASM框架去读取类的元数据信息，用默认的和自定义的TypeFilter去过滤这个类资源，最后注册到BeanFactory中。


</html>


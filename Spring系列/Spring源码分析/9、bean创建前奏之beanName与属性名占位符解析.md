<p>
&nbsp;&nbsp;&nbsp;&nbsp;我们在项目中或多或少都会使用properties配置文件，比如配置数据源的时候，可能会在classpath目录下配置一个jdbc.properties文件，用于配置数据库相关的url地址，数据库密码等相关的信息。当我们想要引入这部分数据的时候，我们会在xml配置文件中配置一个标签，如：
<context:property-placeholder location="classpath:res/*.properties" />
这个标签表示从类加载路径下的res目录下加载所有properties文件，这个标签是一个自定义标签，其对应的标签解析器会向BeanFactory注入一个叫PropertySourcesPlaceholderConfigurer的类，这个类实现了BeanFactoryPostProcessor接口，下面是它的类图
</p>

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/0cb8f24f203113a54c127ec41d20a832.png)

1、PropertiesLoaderSupport 用于支持properties文件的加载，持有PropertiesPersister实例，这个接口主要负责properties的加载和存储工作

2、PropertyResourceConfigurer 抽象类，定义处理占位符能力，实现了BeanFactoryPostProcessor接口

3、PlaceholderConfigurerSupport  定义占位符符号与提供占位符解析样板代码

4、PropertySourcesPlaceholderConfigurer 实现类，提供占位符资源值


<p>
&nbsp;&nbsp;&nbsp;&nbsp;设置环境变量以及指定配置的properties数据
</p>


```
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		if (this.propertySources == null) {
		    //创建多数据源集合
			this.propertySources = new MutablePropertySources();
			if (this.environment != null) {
			    //添加环境变量
				this.propertySources.addLast(
					new PropertySource<Environment>(ENVIRONMENT_PROPERTIES_PROPERTY_SOURCE_NAME, this.environment) {
						@Override
						public String getProperty(String key) {
							return this.source.getProperty(key);
						}
					}
				);
			}
			try {
			    //添加xml配置指定的properties
				PropertySource<?> localPropertySource =
						new PropertiesPropertySource(LOCAL_PROPERTIES_PROPERTY_SOURCE_NAME, mergeProperties());
				if (this.localOverride) {
					this.propertySources.addFirst(localPropertySource);
				}
				else {
					this.propertySources.addLast(localPropertySource);
				}
			}
			catch (IOException ex) {
				throw new BeanInitializationException("Could not load properties", ex);
			}
		}
        //处理属性，替换占位符
		processProperties(beanFactory, new PropertySourcesPropertyResolver(this.propertySources));
		this.appliedPropertySources = this.propertySources;
	}
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;上面主要是属性数据的准备，真正对${}占位的解析还没有开始
</p>


```

protected void org.springframework.context.support.PropertySourcesPlaceholderConfigurer.processProperties(ConfigurableListableBeanFactory beanFactoryToProcess,
			final ConfigurablePropertyResolver propertyResolver) throws BeansException {
        //设置占位符${
		propertyResolver.setPlaceholderPrefix(this.placeholderPrefix);
		//设置占位符}
		propertyResolver.setPlaceholderSuffix(this.placeholderSuffix);
		//设置默认值分隔符:
		propertyResolver.setValueSeparator(this.valueSeparator);
        
        //构建字符串值解析器，一个闭包，使用当前范围中的变量判断使用严格
        //或非严格的字符串解析方法，避免参数显示传递
		StringValueResolver valueResolver = new StringValueResolver() {
			@Override
			public String resolveStringValue(String strVal) {
				String resolved = ignoreUnresolvablePlaceholders ?
						propertyResolver.resolvePlaceholders(strVal) :
						propertyResolver.resolveRequiredPlaceholders(strVal);
				return (resolved.equals(nullValue) ? null : resolved);
			}
		};
        
		doProcessProperties(beanFactoryToProcess, valueResolver);
	}



protected void doProcessProperties(ConfigurableListableBeanFactory beanFactoryToProcess,
			StringValueResolver valueResolver) {
        //创建参观者
		BeanDefinitionVisitor visitor = new BeanDefinitionVisitor(valueResolver);

        //获取所有已经注册了的BeanDefinition名
		String[] beanNames = beanFactoryToProcess.getBeanDefinitionNames();
		for (String curName : beanNames) {
			// Check that we're not parsing our own bean definition,
			// to avoid failing on unresolvable placeholders in properties file locations.
			//当前将要解析占位符的bean不能是当前这个解析类
			//并且只能解析当前解析类同BeanFactory的bean
			if (!(curName.equals(this.beanName) && beanFactoryToProcess.equals(this.beanFactory))) {
				BeanDefinition bd = beanFactoryToProcess.getBeanDefinition(curName);
				try {
				    //开始占位符的解析
					visitor.visitBeanDefinition(bd);
				}
				catch (Exception ex) {
					throw new BeanDefinitionStoreException(bd.getResourceDescription(), curName, ex.getMessage(), ex);
				}
			}
		}

		// New in Spring 2.5: resolve placeholders in alias target names and aliases as well.
		//解析别名的占位符
		beanFactoryToProcess.resolveAliases(valueResolver);

		// New in Spring 3.0: resolve placeholders in embedded values such as annotation attributes.
		//给BeanFactory添加解析器，比如解析@Value
		beanFactoryToProcess.addEmbeddedValueResolver(valueResolver);
	}
	
	
	public void org.springframework.beans.factory.config.BeanDefinitionVisitor.visitBeanDefinition(BeanDefinition beanDefinition) {
	    //解析父beanName
		visitParentName(beanDefinition);
		//解析beanClass
		visitBeanClassName(beanDefinition);
		//解析工厂bean名
		visitFactoryBeanName(beanDefinition);
		//解析工厂方法名
		visitFactoryMethodName(beanDefinition);
		//解析scope
		visitScope(beanDefinition);
		//解析xml配置的属性
		visitPropertyValues(beanDefinition.getPropertyValues());
		//解析xml配置的构造器参数
		ConstructorArgumentValues cas = beanDefinition.getConstructorArgumentValues();
		visitIndexedArgumentValues(cas.getIndexedArgumentValues());
		visitGenericArgumentValues(cas.getGenericArgumentValues());
	}

	
```

<p>
&nbsp;&nbsp;&nbsp;&nbsp;具体的解析动作和创建容器解析配置文件路径时是一样的，只不过这里增加了自己定义的properties属性数据，所以解析占位符的具体代码不再赘述。下面是本次占位符解析的流程
</p>

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/234df6c486495206593a96128a9d589d.png)



这节的MapperScannerConfigurer与spring耦合性比较强，所以如果遗忘了知识点的，请移步到[spring源码分析](https://blog.csdn.net/qq_27785239/article/category/9194901)系列文章中复习下

虽然可以不配置扫描器，但是通常我们在项目中还是会去配置扫描器，配置如下：

```
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"></property>
        <property name="basePackage" value="com.booway.mapper"></property>
    </bean>
```
以下是MapperScannerConfigurer的类图

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/19c08b034aff17857b248ce444b4fcf0.png)


以下是MapperScannerConfigurer的声明

```
public class MapperScannerConfigurer implements BeanDefinitionRegistryPostProcessor, InitializingBean, ApplicationContextAware, BeanNameAware {

  //扫描路径
  private String basePackage;
    
  //是否需要把扫描到的接口配置到Configuration对象中
  private boolean addToConfig = true;
  
  //第一节创建的SqlSessionFactory
  private SqlSessionFactory sqlSessionFactory;

  //SqlSession的模板类
  private SqlSessionTemplate sqlSessionTemplate;
  //SqlSessionFactory的beanName
  private String sqlSessionFactoryBeanName;
  //SqlSessionTemplate的beanNam
  private String sqlSessionTemplateBeanName;
  //用于过滤的注解类型
  private Class<? extends Annotation> annotationClass;
  //标记接口
  private Class<?> markerInterface;
  //spring容器
  private ApplicationContext applicationContext;
  //当前bean在spring容器中的beanName
  private String beanName;
  //是否处理占位符
  private boolean processPropertyPlaceHolders;
  //beanName生成器
  private BeanNameGenerator nameGenerator;
  
  。。。。。。
    
}
```
BeanNameAware与ApplicationContextAware接口并未做除设置对应beanName和applicationContext之外的事情，所以不再分析。InitializingBean接口的实现方法afterPropertiesSet也没有做什么逻辑，只是做了basePackage是否为空的判断。
剩下只剩下实现的BeanDefinitionRegistryPostProcessor这个接口了，这个接口的postProcessBeanFactory方法也没有做什么逻辑，所以再看到它的另一个接口方法postProcessBeanDefinitionRegistry，这个方法就是mybatis实现扫描mapper接口的地方了。这个方法在容器解析完所有xml配置文件后，spring提供的最后添加BeanDefinition的地方了

```
public void org.mybatis.spring.mapper.MapperScannerConfigurer.postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    //是否处理占位符
    if (this.processPropertyPlaceHolders) {
      //(*1*)
      processPropertyPlaceHolders();
    }
    //创建classpath路径扫描器
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    //设置属性
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
    //注册过滤器，如果
    //(*2*)
    scanner.registerFilters();
    scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
  }
  
  //(*1*)
  private void processPropertyPlaceHolders() {
    //从容器中获取实现了BeanFactoryPostProcessor接口的实现资源配置
    Map<String, PropertyResourceConfigurer> prcs = applicationContext.getBeansOfType(PropertyResourceConfigurer.class);

    //如果容器是通用容器实现，才会进行属性的替换，其实吧，即使是GenericApplicationContext，它也是实现了AbstractApplicationContext的
    //一般来说，spring容器会自动替换掉，所以这里的步骤不一定是需要的，但是，毕竟这个接口属于通用接口，一般提供给用户自定义容器的
    //所以可能会覆盖掉refresh方法或者是invokeBeanFactoryPostProcessors方法
    if (!prcs.isEmpty() && applicationContext instanceof GenericApplicationContext) {
      //获取mapperScannerBean
      BeanDefinition mapperScannerBean = ((GenericApplicationContext) applicationContext)
          .getBeanFactory().getBeanDefinition(beanName);

      // PropertyResourceConfigurer does not expose any methods to explicitly perform
      // property placeholder substitution. Instead, create a BeanFactory that just
      // contains this mapper scanner and post process the factory.
      //构建一个空的DefaultListableBeanFactory
      DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
      //然后把mapperScannerBean注入到这个BeanFactory中
      factory.registerBeanDefinition(beanName, mapperScannerBean);

      for (PropertyResourceConfigurer prc : prcs.values()) {
        //调用beanFactory后置处理，替换占位符
        prc.postProcessBeanFactory(factory);
      }

      PropertyValues values = mapperScannerBean.getPropertyValues();
      //更新替换过占位符的属性
      this.basePackage = updatePropertyValue("basePackage", values);
      this.sqlSessionFactoryBeanName = updatePropertyValue("sqlSessionFactoryBeanName", values);
      this.sqlSessionTemplateBeanName = updatePropertyValue("sqlSessionTemplateBeanName", values);
    }
  }
  
  //(*2*)
  public void registerFilters() {
    boolean acceptAllInterfaces = true;

    // if specified, use the given annotation and / or marker interface
    //如果指定了注解
    if (this.annotationClass != null) {
      //添加注解过滤器
      addIncludeFilter(new AnnotationTypeFilter(this.annotationClass));
      //如果添加了注解过滤器，那么acceptAllInterfaces为false，因为有了过滤条件那么不是所有接口都符合条件了
      acceptAllInterfaces = false;
    }

    // override AssignableTypeFilter to ignore matches on the actual marker interface
    //如果添加了标记接口，那么会过滤出这个接口的继承接口
    if (this.markerInterface != null) {
      addIncludeFilter(new AssignableTypeFilter(this.markerInterface) {
        @Override
        protected boolean matchClassName(String className) {
          return false;
        }
      });
      acceptAllInterfaces = false;
    }
    //是否添加了其他的过滤器，如果没有，那么一股脑的全部通过
    if (acceptAllInterfaces) {
      // default include filter that accepts all classes
      addIncludeFilter(new TypeFilter() {
        @Override
        public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
          return true;
        }
      });
    }

    // exclude package-info.java
    //排除package-info.java
    addExcludeFilter(new TypeFilter() {
      @Override
      public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
        String className = metadataReader.getClassMetadata().getClassName();
        return className.endsWith("package-info");
      }
    });
  }
```

开始扫描指定package的类路径下的接口

```
public int org.springframework.context.annotation.ClassPathBeanDefinitionScanner.scan(String... basePackages) {
        //获取容器中已经注册BeanDefinition数量
		int beanCountAtScanStart = this.registry.getBeanDefinitionCount();
        //扫描
        //(*1*)
		doScan(basePackages);

		// Register annotation config processors, if necessary.
		//是否需要注册注解后置处理，比如注册ConfigurationClassPostProcessor，AutowiredAnnotationBeanPostProcessor等
		if (this.includeAnnotationConfig) {
			AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
		}
        /注册了多少mapper
		return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
	}
	
	//(*1*)
public Set<BeanDefinitionHolder> doScan(String... basePackages) {
    //调用父类的扫描方法
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
      logger.warn("No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
    } else {
      //对扫描出来的BeanDefinition做属性处理，比如添加SqlSessionFactory等
      processBeanDefinitions(beanDefinitions);
    }

    return beanDefinitions;

}
```
> ClassPathBeanDefinitionScanner.doScan

```
 protected Set<BeanDefinitionHolder> org.springframework.context.annotation.ClassPathBeanDefinitionScanner.doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<BeanDefinitionHolder>();
		for (String basePackage : basePackages) {
		    //寻找符合要求的BeanDefinition
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			for (BeanDefinition candidate : candidates) {
			    //如果是AnnotatedBeanDefinition，那么将会读取@Scope注解获取scope
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				//设置scope
				candidate.setScope(scopeMetadata.getScopeName());
				//生成beanName，默认AnnotationBeanNameGenerator，这个beanName生成器，会尝试从@Component（即使是元数据）注解中获取beanName
				//如果没有注解，那么就使用默认的beanName生成方法（也就是类的短命名）
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				//应用默认的全局定义，比如是否懒加载，自动装配啊之类
				if (candidate instanceof AbstractBeanDefinition) {
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
				if (candidate instanceof AnnotatedBeanDefinition) {
				    //读取一些基本设置，比如@Lazy,@Primary,@Role等
					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
				//检查是否已经存在，不存在或者可以覆盖就返回true
				if (checkCandidate(beanName, candidate)) {
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
					//设置scope代理模式
					definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);
					//注册
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}
		}
		return beanDefinitions;
	}
```
> ClassPathScanningCandidateComponentProvider.findCandidateComponents(String)

```
public Set<BeanDefinition> org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider.findCandidateComponents(String basePackage) {
		Set<BeanDefinition> candidates = new LinkedHashSet<BeanDefinition>();
		try {
		    //classpath*: + 点号替换成正斜杠(占位符处理(basePackage)) + "/" + "**/*.class"
			String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
					resolveBasePackage(basePackage) + "/" + this.resourcePattern;
			//PathMatchingResourcePatternResolver扫描器，以AntPathMatcher做为匹配器，扫描所有类加载器路径下的class
			//具体的分析过程请翻阅spring源码分析
			Resource[] resources = this.resourcePatternResolver.getResources(packageSearchPath);
			boolean traceEnabled = logger.isTraceEnabled();
			boolean debugEnabled = logger.isDebugEnabled();
			for (Resource resource : resources) {
				if (traceEnabled) {
					logger.trace("Scanning " + resource);
				}
				if (resource.isReadable()) {
					try {
					    //使用asm解码工具读取字节码
						MetadataReader metadataReader = this.metadataReaderFactory.getMetadataReader(resource);
						//使用前面设置的过滤器和排除过滤器过滤class
						if (isCandidateComponent(metadataReader)) {
						    //包装成ScannedGenericBeanDefinition
							ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
							//记录数据来源
							sbd.setResource(resource);
							sbd.setSource(resource);
							//mybatis实现，用于判断当前BeanDefinition是否是接口并且是独立的类（比如成员内部类就不是独立的，它依赖外部类的创建）
							if (isCandidateComponent(sbd)) {
								if (debugEnabled) {
									logger.debug("Identified candidate component class: " + resource);
								}
								//保存到集合中
								candidates.add(sbd);
							}
							else {
								if (debugEnabled) {
									logger.debug("Ignored because not a concrete top-level class: " + resource);
								}
							}
						}
						else {
							if (traceEnabled) {
								logger.trace("Ignored because not matching any filter: " + resource);
							}
						}
					}
					catch (Throwable ex) {
						throw new BeanDefinitionStoreException(
								"Failed to read candidate component class: " + resource, ex);
					}
				}
				else {
					if (traceEnabled) {
						logger.trace("Ignored because not readable: " + resource);
					}
				}
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
		}
		return candidates;
	}
```

好了，这样mybatis就找到了所有需要的mapper类，但是还没完，获取到到这些BeanDefinition后还没有为它们设置属性呢。


```
private void org.mybatis.spring.mapper.ClassPathMapperScanner.processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    for (BeanDefinitionHolder holder : beanDefinitions) {
      definition = (GenericBeanDefinition) holder.getBeanDefinition();

      if (logger.isDebugEnabled()) {
        logger.debug("Creating MapperFactoryBean with name '" + holder.getBeanName() 
          + "' and '" + definition.getBeanClassName() + "' mapperInterface");
      }

      // the mapper interface is the original class of the bean
      // but, the actual class of the bean is MapperFactoryBean
      //添加泛化的构造参数，这个参数是接口的类名
      definition.getConstructorArgumentValues().addGenericArgumentValue(definition.getBeanClassName()); // issue #59
      //将这个BeanDefinition的className设置为mapperFactoryBean，这个mapperFactoryBean实现了spring的FactoryBean
      definition.setBeanClass(this.mapperFactoryBean.getClass());
      //添加addToConfig属性
      definition.getPropertyValues().add("addToConfig", this.addToConfig);

      boolean explicitFactoryUsed = false;
      if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
        //添加sqlSessionFactory的beanName
        definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionFactory != null) {
        //直接添加sqlSessionFactory实例
        definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
        explicitFactoryUsed = true;
      }

      if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
        if (explicitFactoryUsed) {
          //sqlSessionFactory与sqlSessionTemplate不能同时设置，否则sqlSessionFactory会被忽略，sqlSessionTemplate内部就已经注册了一个
          //sqlSessionFactory，sqlSessionTemplate具有更高的优先级
          logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionTemplate != null) {
        if (explicitFactoryUsed) {
          logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        //直接设置sqlSessionTemplate的实例
        definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
        explicitFactoryUsed = true;
      }

      if (!explicitFactoryUsed) {
        if (logger.isDebugEnabled()) {
          logger.debug("Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
        }
        //设置自动装配模式为更具类型注册
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
      }
    }
  }
```

从上面的逻辑可以看到，mybatis最终把BeanDefinition改造成了MapperFactoryBean的BeanDefinition，而这个MapperFactoryBean是实现了spring的FactoryBean的

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/292a77b7c48e380085f40d7edbbf8ea9.png)

我们可以看到它实现了InitializingBean接口，首先我们先来看下我们在配置文件中给它配置的属性是怎么设置的，然后就来看下它的初始化方法

首先我们通常会设置SqlSessionFactory

```
public void org.mybatis.spring.support.SqlSessionDaoSupport.setSqlSessionFactory(SqlSessionFactory sqlSessionFactory) {
    if (!this.externalSqlSession) {
      //可以看到，它构建了一个SqlSessionTemplate
      this.sqlSession = new SqlSessionTemplate(sqlSessionFactory);
    }
  }
```
SqlSessionTemplate的构建

```
public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
    //sqlSessionFactory.getConfiguration().getDefaultExecutorType() 默认值是ExecutorType.SIMPLE
    this(sqlSessionFactory, sqlSessionFactory.getConfiguration().getDefaultExecutorType());
  }
  
public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType) {
    //MyBatisExceptionTranslator 异常转换，发生数据库异常时，可以将异常进行封装，以自己的方式抛出
    this(sqlSessionFactory, executorType,
        new MyBatisExceptionTranslator(
            sqlSessionFactory.getConfiguration().getEnvironment().getDataSource(), true));
  }
  
public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
    notNull(executorType, "Property 'executorType' is required");

    this.sqlSessionFactory = sqlSessionFactory;
    this.executorType = executorType;
    this.exceptionTranslator = exceptionTranslator;
    //构建SqlSession代理
    this.sqlSessionProxy = (SqlSession) newProxyInstance(
        SqlSessionFactory.class.getClassLoader(),
        new Class[] { SqlSession.class },
        new SqlSessionInterceptor());
  }
```
从上面的代码可以看到mybatis的SqlSessionTemplate除了自己实现了SqlSession接口外，还持有一个代理的SqlSession，这是一种装饰器模式
现在我们来看下SqlSessionInterceptor做了什么事情

```
public Object org.mybatis.spring.SqlSessionTemplate.SqlSessionInterceptor.invoke(Object proxy, Method method, Object[] args) throws Throwable {
        //获取SqlSession
        //(*1*)
      SqlSession sqlSession = getSqlSession(
          SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType,
          SqlSessionTemplate.this.exceptionTranslator);
      try {
        //调用sqlSession的方法
        Object result = method.invoke(sqlSession, args);
        //判断当前事务中的sqlSession是不是自己，不是自己才会进行提交，确保在关闭前能够进行提交操作
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          // force commit even on non-dirty sessions because some databases require
          // a commit/rollback before calling close()
          sqlSession.commit(true);
        }
        return result;
      } catch (Throwable t) {
        Throwable unwrapped = unwrapThrowable(t);
        if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
          // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
          //关闭session
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
          sqlSession = null;
          //处理异常，重新抛出
          Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
          if (translated != null) {
            unwrapped = translated;
          }
        }
        throw unwrapped;
      } finally {
        if (sqlSession != null) {
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
  }
  
   //(*1*)
   public static SqlSession org.mybatis.spring.SqlSessionUtils.getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);
    notNull(executorType, NO_EXECUTOR_TYPE_SPECIFIED);
    //从spring事务管理器中获取本地线程中保存的SqlSessionHolder，这个事务管理器的知识在spring源码分析文中分析过，此处不再详细阐述
    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);
    //校验executorType是否相等，已在事务中的executorType，不能发生更改，如果校验成功，从SqlSessionHolder获取SqlSession
    SqlSession session = sessionHolder(executorType, holder);
    if (session != null) {
      return session;
    }

    if (LOGGER.isDebugEnabled()) {
      LOGGER.debug("Creating a new SqlSession");
    }
    //如果没有开启这个sqlSession事务，那么调用SqlSessionFactory打开一个
    //这里将从spring的事务管理器中获取
    //(*2*)
    session = sessionFactory.openSession(executorType);
    //注册sessionHodler
    //(*3*)
    registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);

    return session;
  }
  
  //(*2*)
  private SqlSession org.apache.ibatis.session.defaults.DefaultSqlSessionFactory.openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
        //获取mybatis的Environment，一个维护数据源和事务工厂，数据库id的对象
      final Environment environment = configuration.getEnvironment();
      //获取事务工厂，在spring环境中，默认是SpringManagedTransactionFactory
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      //构建事务，这个事务是从spring的事务管理器TransactionSynchronizationManager中获取ConnectionHolder，最后获取connection
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      //创建执行器
      //(*4*)
      final Executor executor = configuration.newExecutor(tx, execType);
      //构建sqlSession
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
  
  //(*3*)
  private static void registerSessionHolder(SqlSessionFactory sessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator, SqlSession session) {
    SqlSessionHolder holder;
    //事务同步通知是否已经激活
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
      Environment environment = sessionFactory.getConfiguration().getEnvironment();
      //获取事务工厂，在前面的章节中我们分析到，在没有明确指定事务工厂的时候，mybatis默认设置的就是这个SpringManagedTransactionFactory
      if (environment.getTransactionFactory() instanceof SpringManagedTransactionFactory) {
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Registering transaction synchronization for SqlSession [" + session + "]");
        }
        //构建SqlSessionHolder
        holder = new SqlSessionHolder(session, executorType, exceptionTranslator);
        //然后将这个SqlSessionHolder绑定到本地线程中
        TransactionSynchronizationManager.bindResource(sessionFactory, holder);
        //注册mybatis自己的SqlSessionSynchronization事务通知
        //在事务发生挂起，提交，回滚时会被通知，可以用于移除SqlSessionHolder
        TransactionSynchronizationManager.registerSynchronization(new SqlSessionSynchronization(holder, sessionFactory));
        //标记为事务通知
        holder.setSynchronizedWithTransaction(true);
        //自增引用
        holder.requested();
      } else {
        if (TransactionSynchronizationManager.getResource(environment.getDataSource()) == null) {
          if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("SqlSession [" + session + "] was not registered for synchronization because DataSource is not transactional");
          }
        } else {
          throw new TransientDataAccessResourceException(
              "SqlSessionFactory must be using a SpringManagedTransactionFactory in order to use Spring transaction synchronization");
        }
      }
    } else {
      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("SqlSession [" + session + "] was not registered for synchronization because synchronization is not active");
      }
    }
}

//(*4*)
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      //默认我们创建的executorType就是这个
      executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {
      //缓存装饰
      executor = new CachingExecutor(executor);
    }
    //插件代理
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
  
```

初始化方法

```
public final void org.springframework.dao.support.DaoSupport.afterPropertiesSet() throws IllegalArgumentException, BeanInitializationException {
		// Let abstract subclasses check their configuration.
		//(*1*)
		checkDaoConfig();

		// Let concrete implementations initialize themselves.
		try {
		    //还未实现
			initDao();
		}
		catch (Exception ex) {
			throw new BeanInitializationException("Initialization of DAO failed", ex);
		}
	}
	
//(*1*)
protected void checkDaoConfig() {
    //父类只是做了一些判断，判断SqlSession不能为空
    super.checkDaoConfig();
    //mapper接口类名不为空
    notNull(this.mapperInterface, "Property 'mapperInterface' is required");
    //获取配置对象
    Configuration configuration = getSqlSession().getConfiguration();
    //如果addToConfig为true并且配置对象中还没有设置这个mapper接口
    if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
      try {
        //那么添加这个接口的配置，这个方法我们在前面分析过，内部在添加的时候会进行相同路径下mapper配置文件的获取并加载
        //然后就是注解配置方式的解析。
        configuration.addMapper(this.mapperInterface);
      } catch (Exception e) {
        logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", e);
        throw new IllegalArgumentException(e);
      } finally {
        ErrorContext.instance().reset();
      }
    }
  }
```

从接口继承上来看，MapperScannerConfigurer还实现了FactoryBean接口，我们来看下它的getObject方法

```
public T getObject() throws Exception {
    //getSqlSession我们从上面的分析可以知道它其实就是SqlSessionTemplate，内部维护了一个代理的SqlSessionProxy
    //每次调用它的方法的时候都会从spring的事务管理器中获取数据库连接，构建一个新的DefaultSqlSession，然后反射调用其方法
    //(*1*)
    return getSqlSession().getMapper(this.mapperInterface);
  }
  
  //(*1*)
  public <T> T getMapper(Class<T> type) {
    //(*2*)
    return configuration.<T>getMapper(type, this);
  }
  
   //(*2*)
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }
```
创建mapper接口代理对象

```
public <T> T org.apache.ibatis.binding.MapperRegistry.getMapper(Class<T> type, SqlSession sqlSession) {
    //根据接口类型获取MapperProxyFactory，这个mapper接口的注册，我们在前面解析Mapper配置文件的时候分析过
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      //(*1*)
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
  
  //(*1*)
  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    //(*2*)
    return newInstance(mapperProxy);
  }
  
  //(*2*)
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }
```
MapperProxy就是实现了InvocationHandler接口的类

```
public Object org.apache.ibatis.binding.MapperProxy.invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if (Object.class.equals(method.getDeclaringClass())) {
      try {
        //如果是Object的方法，那么直接调用
        return method.invoke(this, args);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    }
    //(*1*)
    //获取MapperMethod
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
  
  //(*1*)
  private MapperMethod cachedMapperMethod(Method method) {
    //缓存中获取
    MapperMethod mapperMethod = methodCache.get(method);
    if (mapperMethod == null) {
      //如果没有，那么构建一个，然后加入缓存
      mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
      methodCache.put(method, mapperMethod);
    }
    return mapperMethod;
  }
```
MapperMethod的方法执行，我们将在后面分析
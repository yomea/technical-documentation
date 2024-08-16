本篇文章是建立在[spring源码分析](https://blog.csdn.net/qq_27785239/article/category/9194901)系列文章之上的，只会对springBoot的启动过程进行大致分析，内部的什么bean的创建，占位符的解析不会再赘述

首先有下面一个例子

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringDemo010JpaApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringDemo010JpaApplication.class, args);
	}
}
```
然后我们只要启动这个main方法就行了

下面大致浏览下这个SpringApplication的一些属性定和方法（方法这里就不贴出来了）

```java
public class SpringApplication {

	/**
	 * The class name of application context that will be used by default for non-web
	 * environments.
	 */
	//如果不是web项目，启动的时候，使用的容器就是这个
	public static final String DEFAULT_CONTEXT_CLASS = "org.springframework.context."
			+ "annotation.AnnotationConfigApplicationContext";

	/**
	 * The class name of application context that will be used by default for web
	 * environments.
	 */
	 //用于web项目
	public static final String DEFAULT_WEB_CONTEXT_CLASS = "org.springframework.boot."
			+ "web.servlet.context.AnnotationConfigServletWebServerApplicationContext";

    //用于判定是否是web项目的依据，这个后面可以看到
	private static final String[] WEB_ENVIRONMENT_CLASSES = { "javax.servlet.Servlet",
			"org.springframework.web.context.ConfigurableWebApplicationContext" };

	/**
	 * The class name of application context that will be used by default for reactive web
	 * environments.
	 */
	 //响应式框架
	public static final String DEFAULT_REACTIVE_WEB_CONTEXT_CLASS = "org.springframework."
			+ "boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext";

    //用于响应式处理的分发处理器
	private static final String REACTIVE_WEB_ENVIRONMENT_CLASS = "org.springframework."
			+ "web.reactive.DispatcherHandler";

    //用于web项目的分发，我们在SpringMVC中看过
	private static final String MVC_WEB_ENVIRONMENT_CLASS = "org.springframework."
			+ "web.servlet.DispatcherServlet";

	/**
	 * Default banner location.
	 */
	 //banner的路径
	public static final String BANNER_LOCATION_PROPERTY_VALUE = SpringApplicationBannerPrinter.DEFAULT_BANNER_LOCATION;

	/**
	 * Banner location property key.
	 */
	 //指定banner地址的属性key名
	public static final String BANNER_LOCATION_PROPERTY = SpringApplicationBannerPrinter.BANNER_LOCATION_PROPERTY;

    //这个我还不晓得是干啥的
	private static final String SYSTEM_PROPERTY_JAVA_AWT_HEADLESS = "java.awt.headless";
    
    //日志对象
	private static final Log logger = LogFactory.getLog(SpringApplication.class);

    //我们设置的主配置源
	private Set<Class<?>> primarySources;

	private Set<String> sources = new LinkedHashSet<>();
	
    //main方法所在的那个类，这里是SpringDemo010JpaApplication
	private Class<?> mainApplicationClass;

    //banner输出模式
	private Banner.Mode bannerMode = Banner.Mode.CONSOLE;

    //是否打印启动日志
	private boolean logStartupInfo = true;

    //是否添加命令行属性
	private boolean addCommandLineProperties = true;

    //banner对象，用于打印banner
	private Banner banner;

    //资源加载器
	private ResourceLoader resourceLoader;

    //beanName生成器
	private BeanNameGenerator beanNameGenerator;

    //可配置的环境，提供属性资源的获取
	private ConfigurableEnvironment environment;
    
    //被应用的容器类名
	private Class<? extends ConfigurableApplicationContext> applicationContextClass;

    //WebApplicationType，用于指定web应用的类型
	private WebApplicationType webApplicationType;

	private boolean headless = true;
    
    //注册钩子函数
	private boolean registerShutdownHook = true;

    //应用初始化器
	private List<ApplicationContextInitializer<?>> initializers;

    //应用监听器
	private List<ApplicationListener<?>> listeners;
    
    //一些默认的属性
	private Map<String, Object> defaultProperties;

    //偏好配置
	private Set<String> additionalProfiles = new HashSet<>();
    
    。。。。。。
}
```
run方法

```java
public static ConfigurableApplicationContext org.springframework.boot.SpringApplication.run(Class<?> primarySource,
			String... args) {
		return run(new Class<?>[] { primarySource }, args);
}
                            |
                            V
public static ConfigurableApplicationContext run(Class<?>[] primarySources,
			String[] args) {
		return new SpringApplication(primarySources).run(args);
}
```

创建SpringApplication对象

```java
public SpringApplication(Class<?>... primarySources) {
		this(null, primarySources);
}
                            |
                            V
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
        //null
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		//设置主class资源
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
		//决定web应用类型
		//(*1*)
		this.webApplicationType = deduceWebApplicationType();
		//设置初始化器
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
		//设置监听器
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		//设置main方法的那个类
		this.mainApplicationClass = deduceMainApplicationClass();
}

//(*1*)
private WebApplicationType deduceWebApplicationType() {
		if (ClassUtils.isPresent(REACTIVE_WEB_ENVIRONMENT_CLASS, null)
				&& !ClassUtils.isPresent(MVC_WEB_ENVIRONMENT_CLASS, null)) {
			return WebApplicationType.REACTIVE;
		}
		for (String className : WEB_ENVIRONMENT_CLASSES) {
			if (!ClassUtils.isPresent(className, null)) {
				return WebApplicationType.NONE;
			}
		}
		return WebApplicationType.SERVLET;
}
```
我们从上面可以看到，有这么一个方法getSpringFactoriesInstances
这个方法是干啥的呢？这个方法会加载类路径下的META-INF/spring.factories

```java
private <T> Collection<T> org.springframework.boot.SpringApplication.getSpringFactoriesInstances(Class<T> type) {
		return getSpringFactoriesInstances(type, new Class<?>[] {});
}
                                            |
                                            V
private <T> Collection<T> org.springframework.boot.SpringApplication.getSpringFactoriesInstances(Class<T> type,
			Class<?>[] parameterTypes, Object... args) {
		ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
		// Use names and ensure unique to protect against duplicates
		Set<String> names = new LinkedHashSet<>(
				SpringFactoriesLoader.loadFactoryNames(type, classLoader));
		//根据类型和构造参数类型数组创建实例，然后按照order接口，优先级，order注解排序
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
				classLoader, args, names);
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
}
                                            |
                                            V
public static List<String> org.springframework.core.io.support.SpringFactoriesLoader.loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
		String factoryClassName = factoryClass.getName();
		return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
}
                                            |
                                            V
private static Map<String, List<String>> org.springframework.core.io.support.SpringFactoriesLoader.loadSpringFactories(@Nullable ClassLoader classLoader) {
	MultiValueMap<String, String> result = cache.get(classLoader);
	if (result != null)
		return result;
	try {
	    //加载类路径META-INF下的spring.factories文件
		Enumeration<URL> urls = (classLoader != null ?
				classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
				ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
		result = new LinkedMultiValueMap<>();
		while (urls.hasMoreElements()) {
			URL url = urls.nextElement();
			UrlResource resource = new UrlResource(url);
			Properties properties = PropertiesLoaderUtils.loadProperties(resource);
			for (Map.Entry<?, ?> entry : properties.entrySet()) {
				List<String> factoryClassNames = Arrays.asList(
						StringUtils.commaDelimitedListToStringArray((String) entry.getValue()));
				result.addAll((String) entry.getKey(), factoryClassNames);
			}
		}
		//缓存
		cache.put(classLoader, result);
		return result;
	}
	catch (IOException ex) {
		throw new IllegalArgumentException("Unable to load factories from location [" +
				FACTORIES_RESOURCE_LOCATION + "]", ex);
	}
}

```
我们来观望下spring.factories文件，key：org.springframework.context.ApplicationContextInitializer表示容器初始化器，在创建了容器，并准备调用refresh方法前被调用，
org.springframework.boot.autoconfigure.EnableAutoConfiguration：这个指定某个模块的自动配置类

```
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer

# Auto Configuration Import Listeners
org.springframework.boot.autoconfigure.AutoConfigurationImportListener=\
org.springframework.boot.autoconfigure.condition.ConditionEvaluationReportAutoConfigurationImportListener

# Auto Configuration Import Filters
org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
org.springframework.boot.autoconfigure.condition.OnClassCondition

# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.cloud.CloudAutoConfiguration,\
org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration,\
org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration,\
org.springframework.boot.autoconfigure.couchbase.CouchbaseAutoConfiguration,\
org.springframework.boot.autoconfigure.dao.PersistenceExceptionTranslationAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraReactiveDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraReactiveRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.cassandra.CassandraRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseReactiveDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseReactiveRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.couchbase.CouchbaseRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.jpa.JpaRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.ldap.LdapDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.ldap.LdapRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.mongo.MongoDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.mongo.MongoReactiveDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.mongo.MongoReactiveRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.mongo.MongoRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.neo4j.Neo4jDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.neo4j.Neo4jRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.solr.SolrRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisReactiveAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.rest.RepositoryRestMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.data.web.SpringDataWebAutoConfiguration,\
org.springframework.boot.autoconfigure.elasticsearch.jest.JestAutoConfiguration,\
org.springframework.boot.autoconfigure.flyway.FlywayAutoConfiguration,\
org.springframework.boot.autoconfigure.freemarker.FreeMarkerAutoConfiguration,\
org.springframework.boot.autoconfigure.gson.GsonAutoConfiguration,\
org.springframework.boot.autoconfigure.h2.H2ConsoleAutoConfiguration,\
org.springframework.boot.autoconfigure.hateoas.HypermediaAutoConfiguration,\
org.springframework.boot.autoconfigure.hazelcast.HazelcastAutoConfiguration,\
org.springframework.boot.autoconfigure.hazelcast.HazelcastJpaDependencyAutoConfiguration,\
org.springframework.boot.autoconfigure.http.HttpMessageConvertersAutoConfiguration,\
org.springframework.boot.autoconfigure.http.codec.CodecsAutoConfiguration,\
org.springframework.boot.autoconfigure.influx.InfluxDbAutoConfiguration,\
org.springframework.boot.autoconfigure.info.ProjectInfoAutoConfiguration,\
org.springframework.boot.autoconfigure.integration.IntegrationAutoConfiguration,\
org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.JdbcTemplateAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.JndiDataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.XADataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.DataSourceTransactionManagerAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.JmsAutoConfiguration,\
org.springframework.boot.autoconfigure.jmx.JmxAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.JndiConnectionFactoryAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.activemq.ActiveMQAutoConfiguration,\
org.springframework.boot.autoconfigure.jms.artemis.ArtemisAutoConfiguration,\
org.springframework.boot.autoconfigure.groovy.template.GroovyTemplateAutoConfiguration,\
org.springframework.boot.autoconfigure.jersey.JerseyAutoConfiguration,\
org.springframework.boot.autoconfigure.jooq.JooqAutoConfiguration,\
org.springframework.boot.autoconfigure.jsonb.JsonbAutoConfiguration,\
org.springframework.boot.autoconfigure.kafka.KafkaAutoConfiguration,\
org.springframework.boot.autoconfigure.ldap.embedded.EmbeddedLdapAutoConfiguration,\
org.springframework.boot.autoconfigure.ldap.LdapAutoConfiguration,\
org.springframework.boot.autoconfigure.liquibase.LiquibaseAutoConfiguration,\
org.springframework.boot.autoconfigure.mail.MailSenderAutoConfiguration,\
org.springframework.boot.autoconfigure.mail.MailSenderValidatorAutoConfiguration,\
org.springframework.boot.autoconfigure.mongo.embedded.EmbeddedMongoAutoConfiguration,\
org.springframework.boot.autoconfigure.mongo.MongoAutoConfiguration,\
org.springframework.boot.autoconfigure.mongo.MongoReactiveAutoConfiguration,\
org.springframework.boot.autoconfigure.mustache.MustacheAutoConfiguration,\
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration,\
org.springframework.boot.autoconfigure.quartz.QuartzAutoConfiguration,\
org.springframework.boot.autoconfigure.reactor.core.ReactorCoreAutoConfiguration,\
org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration,\
org.springframework.boot.autoconfigure.security.servlet.UserDetailsServiceAutoConfiguration,\
org.springframework.boot.autoconfigure.security.servlet.SecurityFilterAutoConfiguration,\
org.springframework.boot.autoconfigure.security.reactive.ReactiveSecurityAutoConfiguration,\
org.springframework.boot.autoconfigure.security.reactive.ReactiveUserDetailsServiceAutoConfiguration,\
org.springframework.boot.autoconfigure.sendgrid.SendGridAutoConfiguration,\
org.springframework.boot.autoconfigure.session.SessionAutoConfiguration,\
org.springframework.boot.autoconfigure.security.oauth2.client.OAuth2ClientAutoConfiguration,\
org.springframework.boot.autoconfigure.solr.SolrAutoConfiguration,\
org.springframework.boot.autoconfigure.thymeleaf.ThymeleafAutoConfiguration,\
org.springframework.boot.autoconfigure.transaction.TransactionAutoConfiguration,\
org.springframework.boot.autoconfigure.transaction.jta.JtaAutoConfiguration,\
org.springframework.boot.autoconfigure.validation.ValidationAutoConfiguration,\
org.springframework.boot.autoconfigure.web.client.RestTemplateAutoConfiguration,\
org.springframework.boot.autoconfigure.web.embedded.EmbeddedWebServerFactoryCustomizerAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.HttpHandlerAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.ReactiveWebServerFactoryAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.WebFluxAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.error.ErrorWebFluxAutoConfiguration,\
org.springframework.boot.autoconfigure.web.reactive.function.client.WebClientAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.HttpEncodingAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.MultipartAutoConfiguration,\
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration,\
org.springframework.boot.autoconfigure.websocket.reactive.WebSocketReactiveAutoConfiguration,\
org.springframework.boot.autoconfigure.websocket.servlet.WebSocketServletAutoConfiguration,\
org.springframework.boot.autoconfigure.websocket.servlet.WebSocketMessagingAutoConfiguration,\
org.springframework.boot.autoconfigure.webservices.WebServicesAutoConfiguration

# Failure analyzers
org.springframework.boot.diagnostics.FailureAnalyzer=\
org.springframework.boot.autoconfigure.diagnostics.analyzer.NoSuchBeanDefinitionFailureAnalyzer,\
org.springframework.boot.autoconfigure.jdbc.DataSourceBeanCreationFailureAnalyzer,\
org.springframework.boot.autoconfigure.jdbc.HikariDriverConfigurationFailureAnalyzer,\
org.springframework.boot.autoconfigure.session.NonUniqueSessionRepositoryFailureAnalyzer

# Template availability providers
org.springframework.boot.autoconfigure.template.TemplateAvailabilityProvider=\
org.springframework.boot.autoconfigure.freemarker.FreeMarkerTemplateAvailabilityProvider,\
org.springframework.boot.autoconfigure.mustache.MustacheTemplateAvailabilityProvider,\
org.springframework.boot.autoconfigure.groovy.template.GroovyTemplateAvailabilityProvider,\
org.springframework.boot.autoconfigure.thymeleaf.ThymeleafTemplateAvailabilityProvider,\
org.springframework.boot.autoconfigure.web.servlet.JspTemplateAvailabilityProvider

```
现在我们回到SpringApplication的run方法

```java
public ConfigurableApplicationContext org.springframework.boot.SpringApplication.run(String... args) {
        //构建一个停止监控，这个主要是用于记录任务的执行时间，上次的执行的任务和执行过的任务
        //内部关联TaskInfo，TaskInfo记录任务的任务名，花费时间
		StopWatch stopWatch = new StopWatch();
		//还是记录当前任务时间，当前任务名，默认是空字符串
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		//配置headless属性（设置到系统属性中），默认为true
		configureHeadlessProperty();
		//获取SpringApplicationRunListener，这个运行监控也是从spring.factories中获取
		//将获取到的SpringApplicationRunListener构建成SpringApplicationRunListeners
		SpringApplicationRunListeners listeners = getRunListeners(args);
		//触发starting事件，内部会调用每一个SpringApplicationRunListener的starting方法
		listeners.starting();
		try {
		    //构建一个默认的应用参数持有者
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			//准备环境
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
		    。。。。。。
		}
		catch (Throwable ex) {
			handleRunFailure(context, listeners, exceptionReporters, ex);
			throw new IllegalStateException(ex);
		}
		listeners.running(context);
		return context;
}
```

上面我们分析到环境的准备，我们看下它是怎么准备的

```java
private ConfigurableEnvironment org.springframework.boot.SpringApplication.prepareEnvironment(
			SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
		// Create and configure the environment
		//创建环境，如果是应用是web的，那么创建是StandardServletEnvironment，这个我们在spring源码分析系列中分析过
		ConfigurableEnvironment environment = getOrCreateEnvironment();
		//配置环境
		configureEnvironment(environment, applicationArguments.getSourceArgs());
		//触发springApplicationRunListener
		listeners.environmentPrepared(environment);
		//从environment绑定属性值到SpringApplication中
		bindToSpringApplication(environment);
		if (this.webApplicationType == WebApplicationType.NONE) {
			environment = new EnvironmentConverter(getClassLoader())
					.convertToStandardEnvironmentIfNecessary(environment);
		}
		ConfigurationPropertySources.attach(environment);
		return environment;
}
```

```java
public ConfigurableApplicationContext run(String... args) {
		。。。。。。
		    
		    //配置忽略banner的属性，默认为true
			configureIgnoreBeanInfo(environment);
			//打印banner
			Banner printedBanner = printBanner(environment);
			//创建容器，这里创建的是org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext
			context = createApplicationContext();
			//从spring.factories中获取异常报告对象，用于处理异常信息
			exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			//准备上下文
			//(*1*)
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
			//刷新上下文，也就是我们在spring源码分析系列比较中的refresh方法，此处不再赘述
			refreshContext(context);
			//刷新后的处理，空方法
			afterRefresh(context, applicationArguments);
			//任务已结束
			stopWatch.stop();
			//打印日志
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			//触发已经启动事件
			listeners.started(context);
			//调用执行器，也就是ApplicationRunner或者CommandLineRunner
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, listeners, exceptionReporters, ex);
			throw new IllegalStateException(ex);
		}
		listeners.running(context);
		return context;
}

//(*1*)
private void org.springframework.boot.SpringApplication.prepareContext(ConfigurableApplicationContext context,
			ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments, Banner printedBanner) {
		//设置环境
		context.setEnvironment(environment);
		//设置beanName生成器，设置资源加载器
		postProcessApplicationContext(context);
		//调用应用上下文初始化器
		applyInitializers(context);
		//触发上下文准备事件
		listeners.contextPrepared(context);
		//打印日志
		if (this.logStartupInfo) {
			logStartupInfo(context.getParent() == null);
			logStartupProfileInfo(context);
		}

		// Add boot specific singleton beans
		//注册单例bean，这个就是包装了传给main方法的参数
		context.getBeanFactory().registerSingleton("springApplicationArguments",
				applicationArguments);
		if (printedBanner != null) {
		    //注册打印banner的单例bean
			context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
		}

		// Load the sources
		//获取配置类资源
		Set<Object> sources = getAllSources();
		Assert.notEmpty(sources, "Sources must not be empty");
		//加载配置类到容器中
		load(context, sources.toArray(new Object[0]));
		//触发容器已加载事件
		listeners.contextLoaded(context);
}
```

看了以上代码后，还是有疑问，这个springboot到底是怎么做到自动配置的呢?
我们来看下标注在SpringDemo010JpaApplication上的@SpringBootApplication注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {

    。。。。。。
}
```
其他的没啥特别的，我们就看下@EnableAutoConfiguration注解

```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    。。。。。。
}
```
我们看到这个注解上有个@Import元注解

AutoConfigurationImportSelector类实现了DeferredImportSelector接口，DeferredImportSelector接口又继承自ImportSelector，对于@Import注解的解析和实现这些接口的解析，我们在[spirng源码分析](https://blog.csdn.net/qq_27785239/article/category/9194901)系列文章中可以了解到，他们是通过一个叫做ConfigurationClassPostProcessor的后置处理器解析的，这个类实现了BeanDefinitionRegistryPostProcessor接口


```java
//annotationMetadata 是引入当前类的引荐者的字节码信息
@Override
public String[] org.springframework.boot.autoconfigure.AutoConfigurationImportSelector.selectImports(AnnotationMetadata annotationMetadata) {
	if (!isEnabled(annotationMetadata)) {
		return NO_IMPORTS;
	}
	try {
	    //加载自动配置元数据，它加载类路劲下的META-INF/spring-autoconfigure-metadata.properties
		AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
				.loadMetadata(this.beanClassLoader);
		//获取引荐者上所有注解的信息
		AnnotationAttributes attributes = getAttributes(annotationMetadata);
		//从spring.factories中获取所有的org.springframework.boot.autoconfigure.EnableAutoConfiguration
		//这些都是java配置类，用于自动配置
		List<String> configurations = getCandidateConfigurations(annotationMetadata,
				attributes);
		//移除重复的配置类名
		configurations = removeDuplicates(configurations);
		//根据自动配置元数据中的AutoConfigureAfter/AutoConfigureBefore或者是类上的@AutoConfigureAfter/@AutoConfigureBefore注解去排序，基本排序思想就是找出某个自动配置类的所有依赖配置类，然后再遍历这些依赖配置类，递归寻找依赖配置类的依赖配置类，然后继续循环，继续递归，直到找到没有依赖其他配置类的配置类。那么这个配置类排在前面。
		configurations = sort(configurations, autoConfigurationMetadata);
		//获取排除的自动配置类
		Set<String> exclusions = getExclusions(annotationMetadata, attributes);
		//检查排除类是否都在configurations，如果存在指定的排除类在configurations不存在，就抛出错误
		checkExcludedClasses(configurations, exclusions);
		//移除被排除的
		configurations.removeAll(exclusions);
		//通过自动配置元数据中的ConditionalOnClass排除一些条件不满足的配置类
		configurations = filter(configurations, autoConfigurationMetadata);
		//触发自动配置引入事件
		fireAutoConfigurationImportEvents(configurations, exclusions);
		return StringUtils.toStringArray(configurations);
	}
	catch (IOException ex) {
		throw new IllegalStateException(ex);
	}
}
```

上面的排序和过滤的具体逻辑请自行分析，比较简单，此处不过多的分析，顺便提下，ConfigurationClassPostProcessor这个后置处理器是在构造容器是在构造器中构建AnnotatedBeanDefinitionReader时加入的

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		Assert.notNull(environment, "Environment must not be null");
		this.registry = registry;
		this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
		//在此处注册的ConfigurationClassPostProcessor
		AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
}
```
> 下面是spring-autoconfigure-metadata.properties文件，用于定义自动配置类的一些规则，比如在某某配置类之后解析，在某某类之前解析，存在某个bean时解析。。。。与spring.factories配置文件配合使用

```java
。。。。。。
org.springframework.boot.autoconfigure.mail.MailSenderValidatorAutoConfiguration=

。。。。。。

org.springframework.boot.autoconfigure.mail.MailSenderValidatorAutoConfiguration.AutoConfigureAfter=org.springframework.boot.autoconfigure.mail.MailSenderAutoConfiguration

。。。。。。

```

我们有时候会在META-INF下面看到一个spring.provides文件，这个文件只是用来告诉用户这个包（可以认为是一个模块）由哪些模块提供。

比如：

```
provides: spring-orm,hibernate-entity-manager,spring-data-jpa
```




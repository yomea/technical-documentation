前提：spring版本 -》1.2.6.RELEASE

## 一、基础组件


我们启动客户端时会引入一个包

```xml

<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-netflix-eureka-client</artifactId>
</dependency>

```
打开这个包，在META-INF目录下我们可以看到一个配置文件spring.factories


```properties

org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.netflix.eureka.config.EurekaClientConfigServerAutoConfiguration,\
org.springframework.cloud.netflix.eureka.config.EurekaDiscoveryClientConfigServiceAutoConfiguration,\
org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration,\
org.springframework.cloud.netflix.ribbon.eureka.RibbonEurekaAutoConfiguration

org.springframework.cloud.bootstrap.BootstrapConfiguration=\
org.springframework.cloud.netflix.eureka.config.EurekaDiscoveryClientConfigServiceBootstrapConfiguration

org.springframework.cloud.client.discovery.EnableDiscoveryClient=\
org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration

```

我们着重看下 EurekaClientAutoConfiguration 这个类，其他的什么 EurekaClientConfigServerAutoConfiguration 和 EurekaDiscoveryClientConfigServiceAutoConfiguration 是用于配置中心的，这里我们先不研究


```java
@Configuration
@EnableConfigurationProperties
@ConditionalOnClass(EurekaClientConfig.class)
@ConditionalOnProperty(value = "eureka.client.enabled", matchIfMissing = true)
@AutoConfigureBefore({ NoopDiscoveryClientAutoConfiguration.class,
		CommonsClientAutoConfiguration.class })
@AutoConfigureAfter(name = "org.springframework.cloud.autoconfigure.RefreshAutoConfiguration")
public class EurekaClientAutoConfiguration {

	。。。。。。

    @Bean
	@ConditionalOnMissingBean(value = EurekaClientConfig.class, search = SearchStrategy.CURRENT)
	public EurekaClientConfigBean eurekaClientConfigBean() {
		EurekaClientConfigBean client = new EurekaClientConfigBean();
		if ("bootstrap".equals(this.env.getProperty("spring.config.name"))) {
			// We don't register during bootstrap by default, but there will be another
			// chance later.
			client.setRegisterWithEureka(false);
		}
		return client;
	}

	@Bean
	@ConditionalOnMissingBean(value = EurekaInstanceConfig.class, search = SearchStrategy.CURRENT)
	public EurekaInstanceConfigBean eurekaInstanceConfigBean(InetUtils inetUtils) {
		EurekaInstanceConfigBean instance = new EurekaInstanceConfigBean(inetUtils);
		instance.setNonSecurePort(this.nonSecurePort);
		instance.setInstanceId(getDefaultInstanceId(this.env));

		if (this.managementPort != this.nonSecurePort && this.managementPort != 0) {
			if (StringUtils.hasText(this.hostname)) {
				instance.setHostname(this.hostname);
			}
			RelaxedPropertyResolver relaxedPropertyResolver = new RelaxedPropertyResolver(env, "eureka.instance.");
			String statusPageUrlPath = relaxedPropertyResolver.getProperty("statusPageUrlPath");
			String healthCheckUrlPath = relaxedPropertyResolver.getProperty("healthCheckUrlPath");
			if (StringUtils.hasText(statusPageUrlPath)) {
				instance.setStatusPageUrlPath(statusPageUrlPath);
			}
			if (StringUtils.hasText(healthCheckUrlPath)) {
				instance.setHealthCheckUrlPath(healthCheckUrlPath);
			}
			String scheme = instance.getSecurePortEnabled() ? "https" : "http";
			instance.setStatusPageUrl(scheme + "://" + instance.getHostname() + ":"
					+ this.managementPort + instance.getStatusPageUrlPath());
			instance.setHealthCheckUrl(scheme + "://" + instance.getHostname() + ":"
					+ this.managementPort + instance.getHealthCheckUrlPath());
		}
		return instance;
	}

	@Bean
	public DiscoveryClient discoveryClient(EurekaInstanceConfig config,
			EurekaClient client) {
		return new EurekaDiscoveryClient(config, client);
	}

	@Bean
	@ConditionalOnMissingBean(value = DiscoveryClientOptionalArgs.class, search = SearchStrategy.CURRENT)
	public MutableDiscoveryClientOptionalArgs discoveryClientOptionalArgs() {
		return new MutableDiscoveryClientOptionalArgs();
	}

	@Configuration
	@ConditionalOnMissingRefreshScope
	protected static class EurekaClientConfiguration {

		@Autowired
		private ApplicationContext context;

		@Autowired(required = false)
		private DiscoveryClientOptionalArgs optionalArgs;

		@Bean(destroyMethod = "shutdown")
		@ConditionalOnMissingBean(value = EurekaClient.class, search = SearchStrategy.CURRENT)
		public EurekaClient eurekaClient(ApplicationInfoManager manager,
				EurekaClientConfig config) {
			return new CloudEurekaClient(manager, config, this.optionalArgs,
					this.context);
		}

		@Bean
		@ConditionalOnMissingBean(value = ApplicationInfoManager.class, search = SearchStrategy.CURRENT)
		public ApplicationInfoManager eurekaApplicationInfoManager(
				EurekaInstanceConfig config) {
			InstanceInfo instanceInfo = new InstanceInfoFactory().create(config);
			return new ApplicationInfoManager(config, instanceInfo);
		}
	}

	。。。。。。

}

```

上面的代码我们只关注下一些比较重要的类

- EurekaClientConfigBean：eureka客户端（所谓客户端就是指当前服务）配置，配置属性以 eureka.client 开头，用于配置链接注册中心的超时时间，拉去服务列表的频率，向其他节点同步的频率，链接eurekaServer的地址等等，比如我们在application.yml配置的属性
    - EurekaTransportConfig transport：httpClient传输层相关配置
    - int registryFetchIntervalSeconds = 30：指定拉取远程列表信息的间隔
    - int instanceInfoReplicationIntervalSeconds = 30：更新实例信息到注册中心的间隔
    - int initialInstanceInfoReplicationIntervalSeconds = 40：初始注册实例信息到注册中心的延时时间
    - int eurekaServiceUrlPollIntervalSeconds = 5 * MINUTES：定时拉取注册中心地址的间隔，具体启动任务 AsyncResolver
    - String proxyPort：注册中心代理端口
    - String proxyHost：注册中心代理机器
    - String proxyUserName：代理登陆用户名
    - String proxyPassword：代理登陆密码
    - int eurekaServerReadTimeoutSeconds = 8：读取注册中心数据超时时间
    - int eurekaServerConnectTimeoutSeconds = 5：链接注册中心超时时间
    - String backupRegistryImpl：备份的注册中心实现，当所有的注册中心无法链接的时候，会通过这个实现获取注册列表，这个实现可以从配置文件中读取或者其他的方式
    - int eurekaServerTotalConnections = 200：与所有注册中心建立的总链接数
    - int eurekaServerTotalConnectionsPerHost = 50：链接到每个注册中心的总连接数
    - String eurekaServerURLContext：注册中心的相对uri，用于通过DNS解析域名时
    - String eurekaServerPort：注册中心的端口，用于DNS解析域名时
    - String eurekaServerDNSName：DNS名字
    - String region = "us-east-1"：数据中心区域名，如果使用了亚马逊云，华为云的，可以指定区域名，默认 us-east-1（美国东海岸）
    - int eurekaConnectionIdleTimeoutSeconds = 30：与注册中心链接空闲超时时间
    - String registryRefreshSingleVipAddress：如果指定了具体的虚拟ip地址，那么仅仅通过此ip获取或者刷新注册列表
    - int heartbeatExecutorThreadPoolSize = 2：心跳线程池最大线程数
    - int heartbeatExecutorExponentialBackOffBound = 10：允许的最大心跳定时间隔，由于网络或则其他的问题，可能在原来固定的定时间隔时间内没有返回结果，为了避免过多的请求阻塞，超时，在小于heartbeatExecutorExponentialBackOffBound的范围内延时时间为正常心跳间隔时间的2倍
    - int cacheRefreshExecutorThreadPoolSize = 2：刷新注册服务列表最大线程数
    - int cacheRefreshExecutorExponentialBackOffBound = 10：与 heartbeatExecutorExponentialBackOffBound 一样的意思，只不过它是刷新本地注册列表的最大延时
    - Map<String, String> serviceUrl：注册中心地址，key：可用域，value：注册中心地址
    - boolean gZipContent = true：压缩传输的数据
    - boolean useDnsForFetchingServiceUrls = false：使用dns服务解析注册中心地址
    - boolean registerWithEureka = true：是否将自身注册到注册中心
    - boolean preferSameZoneEureka = true：偏好可用域，如果为true，那么这个可用域会排在第一被优先使用
    - boolean logDeltaDiff：是否打印增量日志
    - boolean disableDelta：是否禁用增量更新拉取注册列表
    - String fetchRemoteRegionsRegistry：指定获取可用域的域，逗号分割，指定后将会从指定的域获取可用域，获取可用域再获取注册中心服务地址
    - Map<String, String> availabilityZones = new HashMap<>()：指定可用域
    - boolean filterOnlyUpInstances = true：为true过滤掉非UP状态的服务实例
    - boolean fetchRegistry = true：是否从注册中心拉取注册列表
    - String dollarReplacement = "_-"：在与注册中心通信过程中会将美元$用这个 dollarReplacement 值替换
    - String escapeCharReplacement = "__"：在与注册中心通信过程中会将下划线用这个 escapeCharReplacement 值替换
    - boolean allowRedirects = false：在请求注册中心的过程中是否允许重定向
    - boolean onDemandUpdateStatusChange = true：是否监听实例状态更改消息
    - String encoderName：与注册中心通信时的字符编码
    - String decoderName：与注册中心通信时的字符解码
    - String clientDataAccept = EurekaAccept.full.name()：是否压缩数据


- EurekaInstanceConfigBean：eureka客户端实例配置，比如当前服务启动的时绑定的本机器ip，端口，实例id，hostname，启动状态，健康检查等信息
  其中它有几个比较重要的属性
    - appname：应用名，通过 spring.application.name 指定
    - hostname：当前服务所在主机hostname，可以通过 eureka.instance.hostname 制定，如果没有指定，那么通过InetUtils工具类获取
    - nonSecurePort：普通不加密的http端口，server.port 指定
    - securePort：安全端口，https的443
    - leaseRenewalIntervalInSeconds：心跳间隔
    - leaseExpirationDurationInSeconds：心跳超时时间
    - virtualHostName：虚拟的hostname，默认是 appname
    - instanceId：实例id，默认时ip地址 + appname + 端口
    - secureVirtualHostName：https链接时的虚拟hostname
    - aSGName：亚马逊服务组名，一般不用
    - metadataMap：自定义的一些额外属性
    - dataCenterInfo：返回部署此实例的数据中心信息，数据中心是全球协作的特定设备网络，用来在internet网络基础设施上传递、加速、展示、计算、存储数据信息。
    - ipAddress：当前启动实例绑定的ip地址
    - statusPageUrlPath：状态路径，这个用于获取服务状态，默认“/info”，我没有看到在哪里使用了，应该是需要用户自己实现对应controller，当然你如果你是注册中心，那么在开启了 management.info.env.enabled=true 或者没有配置这个属性，那么/info路径将返回贡献者信息
    - homePageUrlPath：定义用户home页面的路径，默认“/”，注意这个根路径对于注册中心来说是注册中心服务的状态，当前注册的提供者信息
    - initialStatus：实例状态，com.netflix.appinfo.InstanceInfo.InstanceStatus

- ApplicationInfoManager：持有实例配置与实例对象信息
    - com.netflix.appinfo.InstanceInfo instanceInfo：通过 EurekaInstanceConfigBean 属性创建的一个代表当前启动服务实例的对象
    - com.netflix.appinfo.EurekaInstanceConfig config：EurekaInstanceConfigBean 实例

- CloudEurekaClient：继承了 DiscoveryClient ，相比DiscoveryClient它提供了心跳事件的发布，这个client是客户端操作实例，可用于获取注册服务，发现服务，注销，关闭等操作

- MutableDiscoveryClientOptionalArgs：用于用户自定义定制 DiscoveryClient 的操作功能，比如注册，发现等请求时添加过滤器，发布eureka事件监听器，定制健康检查方式，定制 EurekaJerseyClient 请求客户端，一般使用springCloud默认的即可

## 二、DiscoveryClient

客户端操作实例，可用于获取注册服务，发现服务，注销，关闭等操作

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/511be5fc3c3c7e397fc7e8a254101a55.png#pic_center)


- LookupService：查找服务接口

```java
public interface LookupService<T> {

    /**
     * 通过应用服务名获取所以注册到注册中心的服务，这个Application可以看作 appname -> List<InstanceInfo> 的容器
     */
    Application getApplication(String appName);

    /**
     * Application集合，获取所有注册到注册中心的应用
     * 
     */
    Applications getApplications();

    /**
     * 通过实例id获取注册的实例信息
     */
    List<InstanceInfo> getInstancesById(String id);

    /**
     * 通过轮询的方式获取下一个实例
     *
     * @param virtualHostname
     *            虚拟hostName，可以指定，在不指定的情况下默认是 appname
     * @param secure
     *            true表示获取https的链接方式，false表示用http的链接方式.
     * @return 返回注册的提供者实例
     */
    InstanceInfo getNextServerFromEureka(String virtualHostname, boolean secure);
}

```

- EurekaClient

```java

public interface EurekaClient extends LookupService {

    /**
     * 获取某域注册的服务实例，region 是数据中心的一个概念，比如亚马逊云，华为云，阿里云等
     */
    public Applications getApplicationsForARegion(@Nullable String region);

    /**
     * 获取指定注册中心注册的服务
     */
    public Applications getApplications(String serviceUrl);

    /**
     * 获取指定虚拟ip地址的服务，vipAddress默认与appname一样
     */
    public List<InstanceInfo> getInstancesByVipAddress(String vipAddress, boolean secure);

    /**
     * 获取指定region与vipAddress的服务
     */
    public List<InstanceInfo> getInstancesByVipAddress(String vipAddress, boolean secure, @Nullable String region);

    /**
     * 获取指定vipAddress与appName的服务列表
     */
    public List<InstanceInfo> getInstancesByVipAddressAndAppName(String vipAddress, String appName, boolean secure);

    // ==========================
    // getters for local metadata
    // ==========================

    /**
     * 获取所以可以访问的 region 名（从注册中心拉取的），包括本地的
     */
    public Set<String> getAllKnownRegions();

    /**
     * 获取自身实例的状态
     */
    public InstanceInfo.InstanceStatus getInstanceRemoteStatus();

    /**
     * 根据可用域获取注册中心地址
     */
    @Deprecated
    public List<String> getDiscoveryServiceUrls(String zone);

    /**
     * 通过指定可用区域从配置中获取注册中心
     */
    @Deprecated
    public List<String> getServiceUrlsFromConfig(String instanceZone, boolean preferSameZone);

    /**
     * 通过指定可用区域从DNS中获取注册中心
     */
    @Deprecated
    public List<String> getServiceUrlsFromDNS(String instanceZone, boolean preferSameZone);

    // ===========================
    // healthcheck related methods
    // ===========================

    /**
     * 注册健康检查回调，用于刷新状态
     */
    @Deprecated
    public void registerHealthCheckCallback(HealthCheckCallback callback);

    /**
     * 和上面 registerHealthCheckCallback 是一样的
     */
    public void registerHealthCheck(HealthCheckHandler healthCheckHandler);

    /**
     * 添加事件监听器，在状态发生改变，刷新注册列表时都会收到消息
     * 
     * @param eventListener
     */
    public void registerEventListener(EurekaEventListener eventListener);
    
    /**
     * 移除事件监听器
     */
    public boolean unregisterEventListener(EurekaEventListener eventListener);
    
    /**
     * 获取健康检查处理器
     */
    public HealthCheckHandler getHealthCheckHandler();

    // =============
    // other methods
    // =============

    /**
     * 关闭客户端，注销注册到注册中心的信息
     */
    public void shutdown();
}

```

- DiscoveryClient
    - ScheduledExecutorService scheduler：定时任务调度线程池，用于定时拉取注册列表，更新注册中心地址，更新注册信息等任务
    - ThreadPoolExecutor heartbeatExecutor：心跳线程池，用于执行心跳处理
    - ThreadPoolExecutor cacheRefreshExecutor：缓存刷新线程池，用于刷新注册列表
    - Provider<HealthCheckHandler> healthCheckHandlerProvider：健康检查处理
    - Provider<HealthCheckCallback> healthCheckCallbackProvider：健康检查回掉
    - AtomicReference<Applications> localRegionApps：注册列表本地缓存信息
    - Lock fetchRegistryUpdateLock：拉取注册列表更新锁
    - AtomicLong fetchRegistryGeneration：记录每次拉取更新注册列表的版本
    - ApplicationInfoManager applicationInfoManager：持有当前服务实例对象与配置信息
    - InstanceInfo instanceInfo：当前服务实例
    - AtomicReference<String> remoteRegionsToFetch：用于拉取注册中心服务地址的数据中心域，多个区域按逗号分割
    - AtomicReference<String[]> remoteRegionsRef：缓存 remoteRegionsToFetch 按逗号分割后的数组
    - InstanceRegionChecker instanceRegionChecker：用于解析当前实例所在的区域，没有使用到亚马逊云一般返回null
    - EndpointUtils.ServiceUrlRandomizer urlRandomizer：用于随机获取一个注册中心地址，实现方式是通过当前hostname进行hash求余，将余数前面的地址往后挪，下次重新hash重复这样的动作
    - Provider<BackupRegistry> backupRegistryProvider：当前所有的注册中心不可用时，将使用这个类获取备份的注册中心地址
    - EurekaTransport eurekaTransport：客户端与注册中心的传输client的集合，持有注册client，查询client，为何要区分注册与查询呢？主要时注册和查询client的传输层配置不一样
    - HealthCheckHandler healthCheckHandler：健康检查处理器
    - Map<String, Applications> remoteRegionVsApps：key -》区域，value-》该区域注册的所有服务
    - InstanceInfo.InstanceStatus lastRemoteInstanceStatus：上次更新到注册中心的实例状态
    - CopyOnWriteArraySet<EurekaEventListener> eventListeners：事件监听器
    - String appPathIdentifier：服务路径鉴别id，appname + instanceId
    - ApplicationInfoManager.StatusChangeListener statusChangeListener：状态监听器，在实例发生状态改变时需要通知的观察者
    - InstanceInfoReplicator instanceInfoReplicator：用于刷新实例状态，更新注册到注册中心的信息
    - int registrySize = 0：从注册中心拉取的服务列表的个数
    - long lastSuccessfulRegistryFetchTimestamp = -1：记录最近一次拉取注册信息的时间戳，可以根据此字段获取与下次拉取成功时间的间隔周期
    - long lastSuccessfulHeartbeatTimestamp = -1：记录上次心跳发生的时间戳
    - ThresholdLevelsMetric heartbeatStalenessMonitor：心跳过时监视器，每次心跳后将获取当前心跳时间与上次心跳时间的差值，与监视器里设置的各个级别的时间比较，我们可以通过监控器监控当前心跳健康状态，springCloud默认使用 jmx 注册到 MBeanServer，那么我们可以通过界面去监控
    - ThresholdLevelsMetric registryStalenessMonitor：注册超时监视器，与 heartbeatStalenessMonitor 一样的道理
    - EurekaClientConfig clientConfig：客户端配置
    - EurekaTransportConfig transportConfig：httpclient配置
    - long initTimestampMs：客户端初始化时间戳

下面我们看下 DiscoveryClient 的构造过程

```java

DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, DiscoveryClientOptionalArgs args,
                    Provider<BackupRegistry> backupRegistryProvider) {
        //如果用户定制了可选参数，那么将优先使用用户定制的，健康检查，事件监听器
        if (args != null) {
            this.healthCheckHandlerProvider = args.healthCheckHandlerProvider;
            this.healthCheckCallbackProvider = args.healthCheckCallbackProvider;
            this.eventListeners.addAll(args.getEventListeners());
        } else {
            this.healthCheckCallbackProvider = null;
            this.healthCheckHandlerProvider = null;
        }
        
        //持有实例与配置，这些配置与实例的作用我们在前文做了比较详细的叙述
        this.applicationInfoManager = applicationInfoManager;
        InstanceInfo myInfo = applicationInfoManager.getInfo();

        clientConfig = config;
        staticClientConfig = clientConfig;
        transportConfig = config.getTransportConfig();
        instanceInfo = myInfo;
        if (myInfo != null) {
            appPathIdentifier = instanceInfo.getAppName() + "/" + instanceInfo.getId();
        } else {
            logger.warn("Setting instanceInfo to a passed in null value");
        }

        this.backupRegistryProvider = backupRegistryProvider;

        this.urlRandomizer = new EndpointUtils.InstanceInfoBasedUrlRandomizer(instanceInfo);
        //初始化本地注册列表
        localRegionApps.set(new Applications());

        fetchRegistryGeneration = new AtomicLong(0);
        //设置获取可用区域
        remoteRegionsToFetch = new AtomicReference<String>(clientConfig.fetchRegistryForRemoteRegions());
        remoteRegionsRef = new AtomicReference<>(remoteRegionsToFetch.get() == null ? null : remoteRegionsToFetch.get().split(","));

        //如果需要拉取注册列表，那么设置注册超时监控器，用于监控拉取注册状态
        if (config.shouldFetchRegistry()) {
            this.registryStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRY_PREFIX + "lastUpdateSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
        } else {
            this.registryStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
        }
        
        //如果需要将自身注册到注册中心，那么需要监控心跳状态
        if (config.shouldRegisterWithEureka()) {
            this.heartbeatStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRATION_PREFIX + "lastHeartbeatSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
        } else {
            this.heartbeatStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
        }
        
        //如果不需要暴露自己并且拉取注册列表，那么什么任务也不设置
        if (!config.shouldRegisterWithEureka() && !config.shouldFetchRegistry()) {
            logger.info("Client configured to neither register nor query for data.");
            scheduler = null;
            heartbeatExecutor = null;
            cacheRefreshExecutor = null;
            eurekaTransport = null;
            instanceRegionChecker = new InstanceRegionChecker(new PropertyBasedAzToRegionMapper(config), clientConfig.getRegion());

            // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
            // to work with DI'd DiscoveryClient
            DiscoveryManager.getInstance().setDiscoveryClient(this);
            DiscoveryManager.getInstance().setEurekaClientConfig(config);

            initTimestampMs = System.currentTimeMillis();

            logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
                    initTimestampMs, this.getApplications().size());
            return;  // no need to setup up an network tasks and we are done
        }

        try {
            //构建调度线程池，用于调度
            //1、用户拉取和刷新注册列表任务
            //2、心跳任务
            //更新服务注册中心地址的任务是在 com.netflix.discovery.shared.resolver.AsyncResolver#executorService 中调度的
            scheduler = Executors.newScheduledThreadPool(3,
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-%d")
                            .setDaemon(true)
                            .build());

            //scheduler 调度任务，heartbeatExecutor 实际执行心跳逻辑
            heartbeatExecutor = new ThreadPoolExecutor(
                    1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                    new SynchronousQueue<Runnable>(),
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
                            .setDaemon(true)
                            .build()
            );  // use direct handoff

            //scheduler 调度任务，cacheRefreshExecutor 实际执行刷新注册列表的逻辑
            cacheRefreshExecutor = new ThreadPoolExecutor(
                    1, clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                    new SynchronousQueue<Runnable>(),
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d")
                            .setDaemon(true)
                            .build()
            );  // use direct handoff
            //初始化传输层client，比如用于拉取注册列表的client，用于查询信息的client，用于解析注册中心地址的解析器
            eurekaTransport = new EurekaTransport();
            scheduleServerEndpointTask(eurekaTransport, args);

            //用于构建通过region获取可用区域的映射器
            AzToRegionMapper azToRegionMapper;
            if (clientConfig.shouldUseDnsForFetchingServiceUrls()) {
                azToRegionMapper = new DNSBasedAzToRegionMapper(clientConfig);
            } else {
                azToRegionMapper = new PropertyBasedAzToRegionMapper(clientConfig);
            }
            if (null != remoteRegionsToFetch.get()) {
                azToRegionMapper.setRegionsToFetch(remoteRegionsToFetch.get().split(","));
            }
            //用于获取实例所在的区域
            instanceRegionChecker = new InstanceRegionChecker(azToRegionMapper, clientConfig.getRegion());
        } catch (Throwable e) {
            throw new RuntimeException("Failed to initialize DiscoveryClient!", e);
        }
        //fetchRegistry 方法用于获取注册中心列表， 如果获取失败，那么获取备份注册中心列表
        if (clientConfig.shouldFetchRegistry() && !fetchRegistry(false)) {
            fetchRegistryFromBackup();
        }

        //初始化调度任务，比如定时获取注册列表，心跳
        initScheduledTasks();
        try {
            //注册到jmx中
            Monitors.registerObject(this);
        } catch (Throwable e) {
            logger.warn("Cannot register timers", e);
        }

        // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
        // to work with DI'd DiscoveryClient
        DiscoveryManager.getInstance().setDiscoveryClient(this);
        DiscoveryManager.getInstance().setEurekaClientConfig(config);

        initTimestampMs = System.currentTimeMillis();
        logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
                initTimestampMs, this.getApplications().size());
    }

```

以上对客户端进行了初始化，设置了注册，拉取，心跳任务，另外还初始化了传输client，用于注册，查询


## 三、EurekaTransport


```java
private void scheduleServerEndpointTask(EurekaTransport eurekaTransport,
                                            DiscoveryClientOptionalArgs args) {

        //请求过滤器，用于客户端与注册中心请求时的过滤
        Collection<ClientFilter> additionalFilters = args == null
                ? Collections.<ClientFilter>emptyList()
                : args.additionalFilters;

        EurekaJerseyClient providedJerseyClient = args == null
                ? null
                : args.eurekaJerseyClient;
        //传输层client工厂，用于创建httpClient
        eurekaTransport.transportClientFactory = providedJerseyClient == null
                ? TransportClientFactories.newTransportClientFactory(clientConfig, additionalFilters, applicationInfoManager.getInfo())
                : TransportClientFactories.newTransportClientFactory(additionalFilters, providedJerseyClient);
        
        ApplicationsResolver.ApplicationsSource applicationsSource = new ApplicationsResolver.ApplicationsSource() {
            @Override
            public Applications getApplications(int stalenessThreshold, TimeUnit timeUnit) {
                long thresholdInMs = TimeUnit.MILLISECONDS.convert(stalenessThreshold, timeUnit);
                long delay = getLastSuccessfulRegistryFetchTimePeriod();
                if (delay > thresholdInMs) {
                    logger.info("Local registry is too stale for local lookup. Threshold:{}, actual:{}",
                            thresholdInMs, delay);
                    return null;
                } else {
                    return localRegionApps.get();
                }
            }
        };
        //用于解析注册中心地址
        eurekaTransport.bootstrapResolver = EurekaHttpClients.newBootstrapResolver(
                clientConfig,
                transportConfig,
                eurekaTransport.transportClientFactory,
                applicationInfoManager.getInfo(),
                applicationsSource
        );
        
        //构建用于注册的client
        if (clientConfig.shouldRegisterWithEureka()) {
            EurekaHttpClientFactory newRegistrationClientFactory = null;
            EurekaHttpClient newRegistrationClient = null;
            try {
                newRegistrationClientFactory = EurekaHttpClients.registrationClientFactory(
                        eurekaTransport.bootstrapResolver,
                        eurekaTransport.transportClientFactory,
                        transportConfig
                );
                newRegistrationClient = newRegistrationClientFactory.newClient();
            } catch (Exception e) {
                logger.warn("Transport initialization failure", e);
            }
            eurekaTransport.registrationClientFactory = newRegistrationClientFactory;
            eurekaTransport.registrationClient = newRegistrationClient;
        }

        // new method (resolve from primary servers for read)
        // Configure new transport layer (candidate for injecting in the future)
        //用于拉取列表的client
        if (clientConfig.shouldFetchRegistry()) {
            EurekaHttpClientFactory newQueryClientFactory = null;
            EurekaHttpClient newQueryClient = null;
            try {
                newQueryClientFactory = EurekaHttpClients.queryClientFactory(
                        eurekaTransport.bootstrapResolver,
                        eurekaTransport.transportClientFactory,
                        clientConfig,
                        transportConfig,
                        applicationInfoManager.getInfo(),
                        applicationsSource
                );
                newQueryClient = newQueryClientFactory.newClient();
            } catch (Exception e) {
                logger.warn("Transport initialization failure", e);
            }
            eurekaTransport.queryClientFactory = newQueryClientFactory;
            eurekaTransport.queryClient = newQueryClient;
        }
    }

```

以上便是初始化注册，查询client的代码

## 四、初始化调度任务


```java

private void initScheduledTasks() {
        if (clientConfig.shouldFetchRegistry()) {
            // registry cache refresh timer
            int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
            int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
            //提交刷新注册列表的任务
            scheduler.schedule(
                    new TimedSupervisorTask(
                            "cacheRefresh",
                            scheduler,
                            cacheRefreshExecutor,
                            registryFetchIntervalSeconds,
                            TimeUnit.SECONDS,
                            expBackOffBound,
                            new CacheRefreshThread()
                    ),
                    registryFetchIntervalSeconds, TimeUnit.SECONDS);
        }

        if (clientConfig.shouldRegisterWithEureka()) {
            int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
            int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
            logger.info("Starting heartbeat executor: " + "renew interval is: " + renewalIntervalInSecs);
            
            //提交心跳的任务
            // Heartbeat timer
            scheduler.schedule(
                    new TimedSupervisorTask(
                            "heartbeat",
                            scheduler,
                            heartbeatExecutor,
                            renewalIntervalInSecs,
                            TimeUnit.SECONDS,
                            expBackOffBound,
                            new HeartbeatThread()
                    ),
                    renewalIntervalInSecs, TimeUnit.SECONDS);

            // InstanceInfo replicator
            //用于刷新实例状态和更新注册信息
            instanceInfoReplicator = new InstanceInfoReplicator(
                    this,
                    instanceInfo,
                    clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                    2); // burstSize
            
            //状态更新监听器，用于刷新注册到注册中心的实例信息
            statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
                @Override
                public String getId() {
                    return "statusChangeListener";
                }

                @Override
                public void notify(StatusChangeEvent statusChangeEvent) {
                    if (InstanceStatus.DOWN == statusChangeEvent.getStatus() ||
                            InstanceStatus.DOWN == statusChangeEvent.getPreviousStatus()) {
                        // log at warn level if DOWN was involved
                        logger.warn("Saw local status change event {}", statusChangeEvent);
                    } else {
                        logger.info("Saw local status change event {}", statusChangeEvent);
                    }
                    //状态更新时，刷新注册到注册中心的信息
                    instanceInfoReplicator.onDemandUpdate();
                }
            };

            if (clientConfig.shouldOnDemandUpdateStatusChange()) {
                applicationInfoManager.registerStatusChangeListener(statusChangeListener);
            }
            //设置定时刷新注册信息的任务
            instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
        } else {
            logger.info("Not registering with Eureka server per configuration");
        }
    }

```

以上代码初始化定时刷新，心跳，注册任务






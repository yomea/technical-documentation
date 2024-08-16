前提：spring版本 -》1.2.6.RELEASE

## 一、注册中心的搭建


### 1.1 启动类

```java
@EnableEurekaServer
@SpringBootApplication
public class EurekaServer {

	public static void main(String[] args) {
		new SpringApplicationBuilder(EurekaServer.class).web(true).run(args);
	}

}
```

### 1.2 配置

```yaml
# 应用服务名
spring.application.name=eureka-server
server.port=7070
# 指定注册中心域名，如果是其他的名字，需要修改hosts文件
#eureka.instance.hostname=localhost

# 是否向注册中心注册自己
eureka.client.register-with-eureka=false
# 是否从注册中心获取注册列表
eureka.client.fetch-registry=false
# 设置注册中心地址
eureka.client.serviceUrl.defaultZone=http://localhost:${server.port}/eureka/,http://localhost2:${server.port}/eureka/


```

可以看见启动一个注册中心很简单，只要在启动类上加上 EnableEurekaServer 注解即可，那么这个注解做了什么呢？

## 二、EnableEurekaServer


```java
/**
 * @author Dave Syer
 *
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(EurekaServerConfiguration.class)
public @interface EnableEurekaServer {

}
```

可以看到这个注解引入了一个配置类，EurekaServerConfiguration，我们点击去看看

```java
@Configuration
@Import(EurekaServerInitializerConfiguration.class)
@EnableDiscoveryClient
@EnableConfigurationProperties({ EurekaDashboardProperties.class, InstanceRegistryProperties.class })
@PropertySource("classpath:/eureka/server.properties")
public class EurekaServerConfiguration extends WebMvcConfigurerAdapter {
	
	。。。。。。
	
	@Configuration
	protected static class EurekaServerConfigBeanConfiguration {
		@Bean
		@ConditionalOnMissingBean
		public EurekaServerConfig eurekaServerConfig(EurekaClientConfig clientConfig) {
			EurekaServerConfigBean server = new EurekaServerConfigBean();
			if (clientConfig.shouldRegisterWithEureka()) {
				// Set a sensible default if we are supposed to replicate
				server.setRegistrySyncRetries(5);
			}
			return server;
		}
	}

	@Bean
	@ConditionalOnProperty(prefix = "eureka.dashboard", name = "enabled", matchIfMissing = true)
	public EurekaController eurekaController() {
		return new EurekaController(this.applicationInfoManager);
	}

	static {
		CodecWrappers.registerWrapper(JACKSON_JSON);
		EurekaJacksonCodec.setInstance(JACKSON_JSON.getCodec());
	}

	@Bean
	public ServerCodecs serverCodecs() {
		return new CloudServerCodecs(this.eurekaServerConfig);
	}

	private static CodecWrapper getFullJson(EurekaServerConfig serverConfig) {
		CodecWrapper codec = CodecWrappers.getCodec(serverConfig.getJsonCodecName());
		return codec == null ? CodecWrappers.getCodec(JACKSON_JSON.codecName()) : codec;
	}

	private static CodecWrapper getFullXml(EurekaServerConfig serverConfig) {
		CodecWrapper codec = CodecWrappers.getCodec(serverConfig.getXmlCodecName());
		return codec == null ? CodecWrappers.getCodec(CodecWrappers.XStreamXml.class)
				: codec;
	}

	class CloudServerCodecs extends DefaultServerCodecs {

		public CloudServerCodecs(EurekaServerConfig serverConfig) {
			super(getFullJson(serverConfig),
					CodecWrappers.getCodec(CodecWrappers.JacksonJsonMini.class),
					getFullXml(serverConfig),
					CodecWrappers.getCodec(CodecWrappers.JacksonXmlMini.class));
		}
	}

	@Bean
	public PeerAwareInstanceRegistry peerAwareInstanceRegistry(
			ServerCodecs serverCodecs) {
		this.eurekaClient.getApplications(); // force initialization
		return new InstanceRegistry(this.eurekaServerConfig, this.eurekaClientConfig,
				serverCodecs, this.eurekaClient,
				this.instanceRegistryProperties.getExpectedNumberOfRenewsPerMin(),
				this.instanceRegistryProperties.getDefaultOpenForTrafficCount());
	}

	@Bean
	public PeerEurekaNodes peerEurekaNodes(PeerAwareInstanceRegistry registry,
			ServerCodecs serverCodecs) {
		return new PeerEurekaNodes(registry, this.eurekaServerConfig,
				this.eurekaClientConfig, serverCodecs, this.applicationInfoManager);
	}

	@Bean
	public EurekaServerContext eurekaServerContext(ServerCodecs serverCodecs,
			PeerAwareInstanceRegistry registry, PeerEurekaNodes peerEurekaNodes) {
		return new DefaultEurekaServerContext(this.eurekaServerConfig, serverCodecs,
				registry, peerEurekaNodes, this.applicationInfoManager);
	}

	@Bean
	public EurekaServerBootstrap eurekaServerBootstrap(PeerAwareInstanceRegistry registry,
			EurekaServerContext serverContext) {
		return new EurekaServerBootstrap(this.applicationInfoManager,
				this.eurekaClientConfig, this.eurekaServerConfig, registry,
				serverContext);
	}

	/**
	 * Register the Jersey filter
	 */
	@Bean
	public FilterRegistrationBean jerseyFilterRegistration(
			javax.ws.rs.core.Application eurekaJerseyApp) {
		FilterRegistrationBean bean = new FilterRegistrationBean();
		bean.setFilter(new ServletContainer(eurekaJerseyApp));
		bean.setOrder(Ordered.LOWEST_PRECEDENCE);
		bean.setUrlPatterns(
				Collections.singletonList(EurekaConstants.DEFAULT_PREFIX + "/*"));

		return bean;
	}

	/**
	 * Construct a Jersey {@link javax.ws.rs.core.Application} with all the resources
	 * required by the Eureka server.
	 */
	@Bean
	public javax.ws.rs.core.Application jerseyApplication(Environment environment,
			ResourceLoader resourceLoader) {

		ClassPathScanningCandidateComponentProvider provider = new ClassPathScanningCandidateComponentProvider(
				false, environment);

		// Filter to include only classes that have a particular annotation.
		//
		provider.addIncludeFilter(new AnnotationTypeFilter(Path.class));
		provider.addIncludeFilter(new AnnotationTypeFilter(Provider.class));

		// Find classes in Eureka packages (or subpackages)
		//
		Set<Class<?>> classes = new HashSet<Class<?>>();
		for (String basePackage : EUREKA_PACKAGES) {
			Set<BeanDefinition> beans = provider.findCandidateComponents(basePackage);
			for (BeanDefinition bd : beans) {
				Class<?> cls = ClassUtils.resolveClassName(bd.getBeanClassName(),
						resourceLoader.getClassLoader());
				classes.add(cls);
			}
		}

		// Construct the Jersey ResourceConfig
		//
		Map<String, Object> propsAndFeatures = new HashMap<String, Object>();
		propsAndFeatures.put(
				// Skip static content used by the webapp
				ServletContainer.PROPERTY_WEB_PAGE_CONTENT_REGEX,
				EurekaConstants.DEFAULT_PREFIX + "/(fonts|images|css|js)/.*");

		DefaultResourceConfig rc = new DefaultResourceConfig(classes);
		rc.setPropertiesAndFeatures(propsAndFeatures);

		return rc;
	}
}

```

现在我们分析下一些主要的类

- ServerCodecs：编码，包含json，压缩json，xml，压缩json

- EurekaController：eureka.dashboard.enabled 属性控制是否开启服务监控，通过 http://ip:port/ 查看服务注册信息面板

- EurekaServerConfigBean：服务配置，eureka.serve前缀
    - String aWSAccessId：亚马逊服务 AccessId
    - String aWSSecretKey：亚马逊服务密钥
    - int eIPBindRebindRetries = 3：亚马逊云ip绑定到实例的重试次数
    - int eIPBindingRetryIntervalMs = 5 * MINUTES：亚马逊云ip绑定到实例的定时间隔
    - int eIPBindingRetryIntervalMsWhenUnbound = 1 * MINUTES：亚马逊云ip未绑定成功定时间隔时间
    - boolean enableSelfPreservation = true：是否开启自我保护机制，这是为了避免清理掉因网络分区使得心跳过期的有效服务。
    - double renewalPercentThreshold = 0.85：心跳阈值百分比
    - int renewalThresholdUpdateIntervalMs = 15 * MINUTES：心跳阈值更新间隔
    - int peerEurekaNodesUpdateIntervalMs = 10 * MINUTES：定时更新同步注册节点信息
    - int numberOfReplicationRetries = 5：复制重试次数
    - int peerEurekaStatusRefreshTimeIntervalMs = 30 * 1000：定时刷新每个节点的状态
    - int waitTimeInMsWhenSyncEmpty = 5 * MINUTES：同步到其他节点的等待时间
    - int peerNodeConnectTimeoutMs = 200：链接其他注册节点的超时时间
    - int peerNodeReadTimeoutMs = 200：读取其他注册节点的超时时间
    - int peerNodeTotalConnections = 1000：链接到其他注册节点的总连接数
    - int peerNodeTotalConnectionsPerHost = 500：链接到每个注册节点的最大链接数
    - int peerNodeConnectionIdleTimeoutSeconds = 30：链接空闲超时时间
    - long retentionTimeInMSInDeltaQueue = 3 * MINUTES：保留在最近更新队列（ConcurrentLinkedQueue<RecentlyChangedItem> recentlyChangedQueue）缓存的时间
    - long deltaRetentionTimerIntervalInMs = 30 * 1000：定期清理最近更新队列
    - long evictionIntervalTimerInMs = 60 * 1000：定期清理心跳过期的注册服务
    - int aSGQueryTimeoutMs = 300：查询 AWS 亚马逊云服务的超时时间
    - long responseCacheAutoExpirationInSeconds = 180：缓存响应的过期时间，客户端请求注册中心时需要将数据进行编码，为了避免每次都要编码，此处做了缓存处理
    - long responseCacheUpdateIntervalMs = 30 * 1000：将缓存响应的缓存内容保存到只读缓存的定时任务时间
    - boolean useReadOnlyResponseCache = true：是否启用只读缓存，如果启用需要靠 responseCacheUpdateIntervalMs 定时刷新内容，会有一定的滞后性
    - boolean disableDelta：是否禁止获取增量信息
    - long maxIdleThreadInMinutesAgeForStatusReplication = 10：节点同步最大空闲线程数
    - int minThreadsForStatusReplication = 1：节点同步最小线程数
    - int maxThreadsForStatusReplication = 1：节点同步的最大线程数
    - int maxElementsInStatusReplicationPool = 10000：节点同步任务队列最大任务数
    - boolean syncWhenTimestampDiffers = true：当客户端实例上次更新时间与在本服务缓存的实例上次更新时间不同时，是否也向其他节点进行同步
    - int registrySyncRetries = 0：如果当前注册中心也注册到了其他注册中心，那么本机将同步这些实例信息，这是同步失败的重试次数
    - long registrySyncRetryWaitMs = 30 * 1000：重试时的等待时间
    - int maxElementsInPeerReplicationPool = 10000：节点同步任务队列最大任务数
    - long maxIdleThreadAgeInMinutesForPeerReplication = 15：每个节点处理任务的最大空闲线程
    - int minThreadsForPeerReplication = 5：每个节点的最小处理任务线程数
    - int maxThreadsForPeerReplication = 20：每个节点的最大处理任务线程数
    - int maxTimeForReplication = 30000：节点同步任务最大处理时间
    - boolean disableDeltaForRemoteRegions：禁用从其他区域注册中心服务（可以认为是从其他注册中心集群获取注册列表，在本地注册中心未获取到注册实例的情况下从其他区域集群获取）读取增量信息
    - int remoteRegionConnectTimeoutMs = 1000：其他区域注册中心服务链接超时时间，remoteRegionUrlsWithName设置之后有效
    - int remoteRegionReadTimeoutMs = 1000：其他区域注册中心服务读取超时时间，remoteRegionUrlsWithName设置之后有效
    - int remoteRegionTotalConnections = 1000：其他区域注册中心服务总链接数，remoteRegionUrlsWithName设置之后有效
    - int remoteRegionTotalConnectionsPerHost = 500：每台服务的总连接数，remoteRegionUrlsWithName设置之后有效
    - int remoteRegionConnectionIdleTimeoutSeconds = 30：其他区域注册中心服务链接空闲超时时间，remoteRegionUrlsWithName设置之后有效
    - Map<String, String> remoteRegionUrlsWithName = new HashMap<>()：key-》区域名，value-》区域url，remoteRegionUrlsWithName设置之后有效
    - String[] remoteRegionUrls：区域地址，remoteRegionUrlsWithName设置之后有效
    - Map<String, Set<String>> remoteRegionAppWhitelist：其他区域注册中心服务白名单列表
    - int remoteRegionRegistryFetchInterval = 30：定时从指定区域拉取注册信息，remoteRegionUrlsWithName设置之后有效
    - int remoteRegionFetchThreadPoolSize = 20：处理指定区域拉取注册信息逻辑的线程数
    - boolean disableTransparentFallbackToOtherRegion：如果从本地注册列表中未获取到实例那么从远程区域中获取
    - boolean batchReplication：指示节点之间是否批量复制
    - boolean rateLimiterEnabled = false：是否开启限流
    - boolean rateLimiterThrottleStandardClients = false：是否开启限流白名单
    - Set<String> rateLimiterPrivilegedClients = Collections.emptySet()：限流白名单
    - int rateLimiterBurstSize = 10：令牌桶大小
    - int rateLimiterRegistryFetchAverageRate = 500：平均拉取数据速率
    - int rateLimiterFullFetchAverageRate = 100：平均拉取全部数据的速率
    - String jsonCodecName：json编码（编码的java类名，非全限定名）
    - String xmlCodecName：xml编码（编码的java类名，非全限定名）


- PeerAwareInstanceRegistry：用于注册，拉取远程区域注册列表

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/05dedf45095882a5183c0bbead73c6b2.png#pic_center)



```java
//定义查找服务的接口
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


//租约管理
public interface LeaseManager<T> {

    /**
     * 注册实例到本机
     */
    void register(T r, int leaseDuration, boolean isReplication);

    /**
     * 注销租约
     */
    boolean cancel(String appName, String id, boolean isReplication);

    /**
     * 续约
     */
    boolean renew(String appName, String id, boolean isReplication);

    /**
     * 清理过期实例
     */
    void evict();
}

//
public interface InstanceRegistry extends LeaseManager<InstanceInfo>, LookupService<String> {
    
    //设置每分钟期望的租约数，设置每分钟租约阈值
    //启动清理过期实例任务
    void openForTraffic(ApplicationInfoManager applicationInfoManager, int count);
    //关闭其他节点任务，取消更新任务
    void shutdown();
    
    //保存重载状态
    @Deprecated
    void storeOverriddenStatusIfRequired(String id, InstanceStatus overriddenStatus);

    void storeOverriddenStatusIfRequired(String appName, String id, InstanceStatus overriddenStatus);
    
    //状态更新
    boolean statusUpdate(String appName, String id, InstanceStatus newStatus,
                         String lastDirtyTimestamp, boolean isReplication);
    //清楚状态重载
    boolean deleteStatusOverride(String appName, String id, InstanceStatus newStatus,
                                 String lastDirtyTimestamp, boolean isReplication);
    //获取重载快照
    Map<String, InstanceStatus> overriddenInstanceStatusesSnapshot();

    //只从本地缓存获取注册列表
    Applications getApplicationsFromLocalRegionOnly();

    //获取注册列表
    List<Application> getSortedApplications();

    /**
     * 获取注册列表，includeRemoteRegion 为true时，如果本地缓存未获取到注册信息，那么就从指定远程区域读取
     */
    Application getApplication(String appName, boolean includeRemoteRegion);

    /**
     * 通过服务名和实例id获取，默认允许从指定远程区域获取注册信息
     */
    InstanceInfo getInstanceByAppAndId(String appName, String id);

    /**
     * 通过服务名和实例id获取，includeRemoteRegions 指定是否允许从远程区域获取注册信息
     */
    InstanceInfo getInstanceByAppAndId(String appName, String id, boolean includeRemoteRegions);

    //清理注册信息
    void clearRegistry();

    //初始化响应缓存，用于缓存每次请求编码后的响应信息
    void initializedResponseCache();

    //获取响应缓存
    ResponseCache getResponseCache();
    
    //获取上次一分钟内统计的续约次数
    long getNumOfRenewsInLastMin();

    //获取每分钟续约的阈值
    int getNumOfRenewsPerMinThreshold();
    
    //检查是否小于续约阈值
    int isBelowRenewThresold();
    //最近注册实例
    List<Pair<Long, String>> getLastNRegisteredInstances();
    //最近取消的实例
    List<Pair<Long, String>> getLastNCanceledInstances();

    /**
     * 是否能够超时，如果允许超时那么将允许清理过期的实例
     */
    boolean isLeaseExpirationEnabled();
    //是否开启自我保护
    boolean isSelfPreservationModeEnabled();

}


public interface PeerAwareInstanceRegistry extends InstanceRegistry {
    
    //初始化响应缓存
    //定时更新续约阈值
    //如果指定了远程区域，那么会构建RemoteRegionRegistry用于获取远程区域注册列表
    void init(PeerEurekaNodes peerEurekaNodes) throws Exception;

    /**
     * 从其他节点同步注册信息
     */
    int syncUp();

    /**
     * 是否允许从注册列表中获取信息，只有在注册列表准别好之后才允许访问，否则可能返回空列表
     */
     boolean shouldAllowAccess(boolean remoteRegionRequired);
     //注册
     void register(InstanceInfo info, boolean isReplication);
     //状态更新
     void statusUpdate(final String asgName, final ASGResource.ASGStatus newStatus, final boolean isReplication);
}

```

- PeerEurekaNodes：其他节点实例，用于同步信息
- EurekaServerContext：服务上下文，持有编码方式，服务配置，注册实例，其他节点对象，另外可以进行初始化，设置定时更新节点信息的任务，初始化相应缓存，更新续约阈值

```java
@PostConstruct
@Override
public void com.netflix.eureka.DefaultEurekaServerContext#initialize() throws Exception {
    logger.info("Initializing ...");
    //开启定时更新节点的定时任务
    peerEurekaNodes.start();
    //
    registry.init(peerEurekaNodes);
    logger.info("Initialized");
}

@Override
public void com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#init(PeerEurekaNodes peerEurekaNodes) throws Exception {
    this.numberOfReplicationsLastMin.start();
    this.peerEurekaNodes = peerEurekaNodes;
    //初始化响应缓存，用于缓存编码响应
    initializedResponseCache();
    //设置续约阈值调度任务
    scheduleRenewalThresholdUpdateTask();
    //如果设置了从远程区域获取注册列表的区域，那么创建一个 RemoteRegionRegistry 用于获取
    initRemoteRegionRegistry();

    try {
        Monitors.registerObject(this);
    } catch (Throwable e) {
        logger.warn("Cannot register the JMX monitor for the InstanceRegistry :", e);
    }
}
```

- EurekaServerBootstrap：初始化注册中心环境和上下文，另外设置了过期实例清理任务，同步其他节点实例节点
- javax.ws.rs.core.Application：扫描注解Provider与Path，Jersey是一个 webservice 框架，springcloud通过它接受请求


## 二、注册中心服务操作

### 2.1 容器信息查询

```java
@Path("/{version}/apps")
@Produces({"application/xml", "application/json"})
public class ApplicationsResource {
    
    。。。。。。省略部分代码
    
    /**
     * 获取所有应用信息
     */
    @GET
    public Response getContainers(@PathParam("version") String version,
                                  @HeaderParam(HEADER_ACCEPT) String acceptHeader,
                                  @HeaderParam(HEADER_ACCEPT_ENCODING) String acceptEncoding,
                                  @HeaderParam(EurekaAccept.HTTP_X_EUREKA_ACCEPT) String eurekaAccept,
                                  @Context UriInfo uriInfo,
                                  @Nullable @QueryParam("regions") String regionsStr) {

        。。。。。。
    }

    /**
     * 获取注册信息到增量信息
     */
    @Path("delta")
    @GET
    public Response getContainerDifferential(
            @PathParam("version") String version,
            @HeaderParam(HEADER_ACCEPT) String acceptHeader,
            @HeaderParam(HEADER_ACCEPT_ENCODING) String acceptEncoding,
            @HeaderParam(EurekaAccept.HTTP_X_EUREKA_ACCEPT) String eurekaAccept,
            @Context UriInfo uriInfo, @Nullable @QueryParam("regions") String regionsStr) {

        。。。。。。
    }
}

```

用于查询所有注册信息与增量信息

### 2.2 实例查询与注册


```java
@Produces({"application/xml", "application/json"})
public class ApplicationResource {

    。。。。。。

    /**
     * 获取注册实例信息
     */
    @GET
    public Response getApplication(@PathParam("version") String version,
                                   @HeaderParam("Accept") final String acceptHeader,
                                   @HeaderParam(EurekaAccept.HTTP_X_EUREKA_ACCEPT) String eurekaAccept) {
        。。。。。。
    }

    /**
     * 通过实例id获取注册实例信息
     */
    @Path("{id}")
    public InstanceResource getInstanceInfo(@PathParam("id") String id) {
        return new InstanceResource(this, id, serverConfig, registry);
    }

    /**
     * 注册实例
     */
    @POST
    @Consumes({"application/json", "application/xml"})
    public Response addInstance(InstanceInfo info,
                                @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
        。。。。。。
    }

    /**
     * Returns the application name of a particular application.
     *
     * @return the application name of a particular application.
     */
    String getName() {
        return appName;
    }

    private boolean isBlank(String str) {
        return str == null || str.isEmpty();
    }
}
```

用于注册与查询实例信息

### 2.3 实例查询与注册


```java

@Path("/{version}/peerreplication")
@Produces({"application/xml", "application/json"})
public class PeerReplicationResource {

    。。。。。。

    /**
     * 批量复制节点
     */
    @Path("batch")
    @POST
    public Response batchReplication(ReplicationList replicationList) {
        。。。。。。
    }

    。。。。。。
}

```








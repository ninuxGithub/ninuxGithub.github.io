---
title: SpringCloud Eureka Server 原理分析
author: ninuxGithub
layout: post
date: 2019-8-1 08:35:47
description: "Eureka Server"
tag: spring-cloud
---


## 目的
    分析Eureka Server 运行的原理，源代码分析思路如下
     
    
    简单介绍一下spring boot 的原理；
    spring boot auto configuration 的原理建立xxxx-start, 然后在MATE-INF会配置一个spring.factories, 启动spring boot 的时候
    这些spring.factories里面配置的自动启动的类都会被spring加载，进行初始化，创建实例对象为后续的业务代码做准备工作；
    
    spring cloud Eureka Server 启动的时候在我们的启动main类需要启用@EnableEurekaServer定位到类的位置，定位到包，
    和这个注解相关的业务代码应该是在同一个包；或者可以直接查看对于的jar包的MATE-INF/spring.factories里面配置的自动启动加载的类
    我们可以和容易的找到xxxAutoConfiguration


## 原理分析
    EurekaServerAutoConfiguration 这个类承担了spring cloud 微服务Eureka Server 启动的时候的主要的类的加载工作；
    下面通过源代码来查看细节；


```java
class EurekaServerAutoConfiguration{
    @Autowired
	private ApplicationInfoManager applicationInfoManager;

	@Autowired
	private EurekaServerConfig eurekaServerConfig;

	@Autowired
	private EurekaClientConfig eurekaClientConfig;

	@Autowired
	private EurekaClient eurekaClient;
	
	@Autowired
    private InstanceRegistryProperties instanceRegistryProperties;
}
```
    
    @Autowired
    private ApplicationInfoManager applicationInfoManager;
    通过EurekaClientAutoConfiguration进行了初始化
    

    @Autowired
    private EurekaServerConfig eurekaServerConfig;
    通过读取配置文件进行bean的配置创建实例对象EurekaServerConfigBean 获取配置文件里面的eureka.server.**

    @Autowired
    private EurekaClientConfig eurekaClientConfig;
    通过读取配置文件eureka.client.** 属性类初始化EurekaClientConfigBean
    

    @Autowired
    private EurekaClient eurekaClient;
    通过EurekaClientAutoConfiguration来进行配置的
    有2个范围的bean, 一个是ConditionalOnMissingRefreshScope ，另外一个是ConditionalOnRefreshScope 个人理解是一个是有条件刷新
    环境的bean , 另外一个是没有条件刷新的环境的bean;
    class CloudEurekaClient extends DiscoveryClient可见CloudEurekaClient继承了DisCoveryClient对象
    CloudEurekaClient 是spring cloud 自己曾经的一个服务注册的客户端
    DiscoveryClient 是netflix提供的eureka服务发现组件；
    可以理解为spring cloud Eureka 借助了netflix 来实现微服务的注册和发现；
    所以服务发现和注册的逻辑还要去关注netflix底层做了哪些动作， spring cloud 只是对其进行了一层包装，并且加入了一下spring cloud
    自己实现的东西；


```java
public class CloudEurekaClient extends DiscoveryClient {
    
    public CloudEurekaClient(ApplicationInfoManager applicationInfoManager,EurekaClientConfig config,
                             AbstractDiscoveryClientOptionalArgs<?> args,
                             ApplicationEventPublisher publisher) {
        super(applicationInfoManager, config, args);
        this.applicationInfoManager = applicationInfoManager;
        this.publisher = publisher;
        this.eurekaTransportField = ReflectionUtils.findField(DiscoveryClient.class, "eurekaTransport");
        ReflectionUtils.makeAccessible(this.eurekaTransportField);
    }
}
```    
    
    这是CloudEurekaClient的在自动配置的时候创建对象的构造器， 那么我们可以看到调用了super(applicationInfoManager, config, args)
    来进行父类的初始化的动作；
    进一步的debug进入到DiscoveryClient查看细节

```java
class DiscoveryClient {
    
    DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                        Provider<BackupRegistry> backupRegistryProvider) {
        if (args != null) {
            this.healthCheckHandlerProvider = args.healthCheckHandlerProvider;
            this.healthCheckCallbackProvider = args.healthCheckCallbackProvider;
            this.eventListeners.addAll(args.getEventListeners());
            this.preRegistrationHandler = args.preRegistrationHandler;
        } else {
            this.healthCheckCallbackProvider = null;
            this.healthCheckHandlerProvider = null;
            this.preRegistrationHandler = null;
        }
        
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
        
        //设置applications 进入可以考到Applications 内部维护了一个map有一系列的Application对象
        //我们经常说eureka为我们提供服务的注册和发现；
        // 那么这个Application不就是我们Eureka维护的服务的实例对象吗； 需要进一步的debug验证
        //localRegionApps 是一个AtomicReference对象，为这些服务的增加减少提供原子的操作
        localRegionApps.set(new Applications());

        fetchRegistryGeneration = new AtomicLong(0);

        remoteRegionsToFetch = new AtomicReference<String>(clientConfig.fetchRegistryForRemoteRegions());
        remoteRegionsRef = new AtomicReference<>(remoteRegionsToFetch.get() == null ? null : remoteRegionsToFetch.get().split(","));

        //设置guage monitor 一个衡量的列表的标准，如果服务达到了延迟达到了某个阈值的时候会进行调节请求的间隔时间
        if (config.shouldFetchRegistry()) {
            this.registryStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRY_PREFIX + "lastUpdateSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
        } else {
            this.registryStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
        }

        if (config.shouldRegisterWithEureka()) {
            this.heartbeatStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRATION_PREFIX + "lastHeartbeatSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
        } else {
            this.heartbeatStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
        }

        logger.info("Initializing Eureka in region {}", clientConfig.getRegion());

        //register-with-eureka: false 
        //fetch-registry: false 
        //当这个2个配置为false的时候不会将自己注册为client服务
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
        
        
        //否则就是要注册为服务了， 那么注册为服务后需要维护服务是可用的就需要应用进行更新
        //要确定应用是在线的就需要发送心跳
        //所以先创建了一个scheduler 来执行一个定时器的任务，每隔x秒来执行某个任务来执行服务的更新，和服务心跳发送
        //执行什么任务呢？ 需要进一步的往下看
        try {
            // default size of 2 - 1 each for heartbeat and cacheRefresh
            //创建一个schedule 来执行定时器任务
            scheduler = Executors.newScheduledThreadPool(2,
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-%d")
                            .setDaemon(true)
                            .build());

            //心跳的线程池执行器
            heartbeatExecutor = new ThreadPoolExecutor(
                    1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                    new SynchronousQueue<Runnable>(),
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
                            .setDaemon(true)
                            .build()
            );  // use direct handoff

            //服务缓存刷新的线程池执行器
            cacheRefreshExecutor = new ThreadPoolExecutor(
                    1, clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                    new SynchronousQueue<Runnable>(),
                    new ThreadFactoryBuilder()
                            .setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d")
                            .setDaemon(true)
                            .build()
            );  // use direct handoff

            //创建eurekaTransport对象
            eurekaTransport = new EurekaTransport();
            
            //判断服务是否需要fetchRegistry or registerWithEureka
            scheduleServerEndpointTask(eurekaTransport, args);

            AzToRegionMapper azToRegionMapper;
            if (clientConfig.shouldUseDnsForFetchingServiceUrls()) {
                azToRegionMapper = new DNSBasedAzToRegionMapper(clientConfig);
            } else {
                azToRegionMapper = new PropertyBasedAzToRegionMapper(clientConfig);
            }
            if (null != remoteRegionsToFetch.get()) {
                azToRegionMapper.setRegionsToFetch(remoteRegionsToFetch.get().split(","));
            }
            //从名称来看是服务实例区域的检查 ， 例如阿里云服务器是华南的还是东北那嘎达的
            instanceRegionChecker = new InstanceRegionChecker(azToRegionMapper, clientConfig.getRegion());
        } catch (Throwable e) {
            throw new RuntimeException("Failed to initialize DiscoveryClient!", e);
        }

        //fetchRegistry(false) 开始获取注册的信息
        //fire event 开发布事件消息 去invoke 对应的事件
        if (clientConfig.shouldFetchRegistry() && !fetchRegistry(false)) {
            //java doc : 如果所有的eureka server url都不可用的时候从backup获取注册
            fetchRegistryFromBackup();
        }

        // call and execute the pre registration handler before all background tasks (inc registration) is started
        if (this.preRegistrationHandler != null) {
            this.preRegistrationHandler.beforeRegistration();
        }

        //should-enforce-registration-at-init: true
        //register-with-eureka: true
        //如果2个变量设置为true的时候进行启动服务的时候强制的进行访问的注册， 如果有一个或2个为false呢？
        //为false的时候服务什么时候进行注册呢？ 往下继续看
        if (clientConfig.shouldRegisterWithEureka() && clientConfig.shouldEnforceRegistrationAtInit()) {
            try {
               //注册实例到中心
                if (!register() ) {
                    throw new IllegalStateException("Registration error at startup. Invalid server response.");
                }
            } catch (Throwable th) {
                logger.error("Registration error at startup: {}", th.getMessage());
                throw new IllegalStateException(th);
            }
        }

        // finally, init the schedule tasks (e.g. cluster resolvers, heartbeat, instanceInfo replicator, fetch
        //java doc : 开始我们的心跳服务，服务缓存更新的任务开始调度了任务线程为CacheRefreshThread ， HeartbeatThread 这个不在看了
        //内部是通过多线程，使用eurekaTransport获取到的EurekaHttpClient发起请求进行访问的续租和服务心跳的发送
        initScheduledTasks();

        try {
            //向监视器注册自己
            Monitors.registerObject(this);
        } catch (Throwable e) {
            logger.warn("Cannot register timers", e);
        }

        // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
        // to work with DI'd DiscoveryClient
        //DiscoveryManager 是一个单例对象， 注册自己  和设置客户端的配置
        DiscoveryManager.getInstance().setDiscoveryClient(this);
        DiscoveryManager.getInstance().setEurekaClientConfig(config);

        initTimestampMs = System.currentTimeMillis();
        logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
                initTimestampMs, this.getApplications().size());
    }
    
    //初始化schedule任务
    private void initScheduledTasks() {
        //fetch-registry: true 进行服务的注册进行获取
        if (clientConfig.shouldFetchRegistry()) {
            // registry cache refresh timer
            //registryFetchIntervalSeconds = 30; 默认为30秒
            int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
            int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
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

        //register-with-eureka: true
        if (clientConfig.shouldRegisterWithEureka()) {
            //DEFAULT_LEASE_RENEWAL_INTERVAL = 30; 服务续租的默认时间为30秒
            int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
            int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
            logger.info("Starting heartbeat executor: " + "renew interval is: {}", renewalIntervalInSecs);

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
            //instanceInfoReplicationIntervalSeconds = 30;
            //创建一个服务的复制对象
            instanceInfoReplicator = new InstanceInfoReplicator(
                    this,
                    instanceInfo,
                    clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                    2); // burstSize

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
                    //onDemandUpdate 内部怎么调用的了， 往下看 进行分析
                    instanceInfoReplicator.onDemandUpdate();
                }
            };

            //on-demand-update-status-change: true
            if (clientConfig.shouldOnDemandUpdateStatusChange()) {
                //注册服务状态改变的监听器
                //这个监听器就是如果在服务状态发生了变化的时候进行访问的状态的更新
                //监听器的notify会被谁去调用呢？ 继续debug
                applicationInfoManager.registerStatusChangeListener(statusChangeListener);
            }

            instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
        } else {
            logger.info("Not registering with Eureka server per configuration");
        }
    }
    
    
    public boolean onDemandUpdate() {
        if (rateLimiter.acquire(burstSize, allowedRatePerMinute)) {
            if (!scheduler.isShutdown()) {
                scheduler.submit(new Runnable() {
                    @Override
                    public void run() {
                        logger.debug("Executing on-demand update of local InstanceInfo");
    
                        Future latestPeriodic = scheduledPeriodicRef.get();
                        if (latestPeriodic != null && !latestPeriodic.isDone()) {
                            logger.debug("Canceling the latest scheduled update, it will be rescheduled at the end of on demand update");
                            latestPeriodic.cancel(false);
                        }
                        //调用了run方法 ，可以debug进入看看， 发现调用了discoveryClient.register();进行了服务的注册
                        InstanceInfoReplicator.this.run();
                    }
                });
                return true;
            } else {
                logger.warn("Ignoring onDemand update due to stopped scheduler");
                return false;
            }
        } else {
            logger.warn("Ignoring onDemand update due to rate limiter");
            return false;
        }
    }
    
}



/*
    如果DiscoveryClient在初始化的时候没有强制启动服务的注册， 那么会通过EurekaAutoServiceRegistration进行访问的注册，
    改类实现了SmartLifecycle : 该类有stop start会在spring 容器启动的是被调用
    所以可以通过该类在启动的是进行访问的注册



 */
public class EurekaAutoServiceRegistration implements AutoServiceRegistration, SmartLifecycle, Ordered {

	@Override
	public void start() {
		// only set the port if the nonSecurePort or securePort is 0 and this.port != 0
		if (this.port.get() != 0) {
			if (this.registration.getNonSecurePort() == 0) {
				this.registration.setNonSecurePort(this.port.get());
			}

			if (this.registration.getSecurePort() == 0 && this.registration.isSecure()) {
				this.registration.setSecurePort(this.port.get());
			}
		}

		// only initialize if nonSecurePort is greater than 0 and it isn't already running
		// because of containerPortInitializer below
		if (!this.running.get() && this.registration.getNonSecurePort() > 0) {

            //调用EurekaServiceRegistry进行服务的注册
			this.serviceRegistry.register(this.registration);

			this.context.publishEvent(
					new InstanceRegisteredEvent<>(this, this.registration.getInstanceConfig()));
			this.running.set(true);
		}
	}

}



public class EurekaServiceRegistry implements ServiceRegistry<EurekaRegistration> {

	private static final Log log = LogFactory.getLog(EurekaServiceRegistry.class);

	@Override
	public void register(EurekaRegistration reg) {
		maybeInitializeClient(reg);

		if (log.isInfoEnabled()) {
			log.info("Registering application " + reg.getInstanceConfig().getAppname()
					+ " with eureka with status "
					+ reg.getInstanceConfig().getInitialStatus());
		}

        //*** setInstanceStatus查看内部的逻辑， 不就是调用了DiscoveryClient注册监听到了ApplicationInfoManager里面去了吗
        // 然后就是进行监听器的notify调用了，从而进行了instanceInfoReplicator.onDemandUpdate();的调用
		reg.getApplicationInfoManager()
				.setInstanceStatus(reg.getInstanceConfig().getInitialStatus());

		reg.getHealthCheckHandler().ifAvailable(healthCheckHandler ->
				reg.getEurekaClient().registerHealthCheck(healthCheckHandler));
	}

}

```

    
    然后是获取到父类里面的eurekaTransport，设置为可以暴力访问属性；
    eurekaTransport可用来干什么呢？  
    -->就是为了或EurekaHttpClient对象来发起http的请求；
    所以服务的注册， 服务的心跳，服务的缓存更新都需要这个对象发起http请求来完成的；
    
    
    
    
    
    @Autowired
    private InstanceRegistryProperties instanceRegistryProperties;
    读取配置文件进行配置eureka.instance.registry.**
    
    
  
  
```java

/**
 * @author Dave Syer
 */
@Configuration
public class EurekaServerInitializerConfiguration
		implements ServletContextAware, SmartLifecycle, Ordered {

	private static final Log log = LogFactory.getLog(EurekaServerInitializerConfiguration.class);

	@Autowired
	private EurekaServerConfig eurekaServerConfig;

	private ServletContext servletContext;

	@Autowired
	private ApplicationContext applicationContext;

	@Autowired
	private EurekaServerBootstrap eurekaServerBootstrap;

	private boolean running;

	private int order = 1;

	@Override
	public void setServletContext(ServletContext servletContext) {
		this.servletContext = servletContext;
	}

	@Override
	public void start() {
		new Thread(new Runnable() {
			@Override
			public void run() {
				try {
					//TODO: is this class even needed now?
					//这里才是真正开始启动eureka sever 的地方
					eurekaServerBootstrap.contextInitialized(EurekaServerInitializerConfiguration.this.servletContext);
					log.info("Started Eureka Server");

                    //发布事件
					publish(new EurekaRegistryAvailableEvent(getEurekaServerConfig()));
					EurekaServerInitializerConfiguration.this.running = true;
					publish(new EurekaServerStartedEvent(getEurekaServerConfig()));
				}
				catch (Exception ex) {
					// Help!
					log.error("Could not initialize Eureka servlet context", ex);
				}
			}
		}).start();
	}
}

```  



## 总结
    2019-8-1 10:08:30 更新
    Eureka 组件提供了服务的注册和发现机制， 让我们的服务接收eureka的管理，通过定时器来执行多线程任务来进行访问缓存的更新，还有
    就是进行服务心跳的发送来检查我们的服务是否是存活的；
    EurekaAutoServiceRegistration实现了SmartLifecycle 被spring 容器启动的是进行了初始化调用start进行访问的注册动作
    那么DiscoveryClient是通spring boot 自动配置来完成初始化加载的，提供了Eureka服务的发现机制，当然访问注册的逻辑也通过参数
    来强制控制服务的注册在DiscoveryClient初始化的时候就进行一次调用；
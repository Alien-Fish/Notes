---
layout: post
#标题配置
title:  Eureka源码分析3-客户端初始化
#时间配置
date:   2019-11-10 12:00:00 +0800
#大类配置
categories: 服务注册
#小类配置
tag: Eureka
---

* content
{:toc}

## 说明
![/styles/images/eureka/eureka-client_discovery.png]({{ '/styles/images/eureka/eureka-client_discovery.png' | prepend: site.baseurl  }})
DiscoveryClient是核心客户端服务发现实现类，实现了 EurekaClient 定义的规范接口。  
本文只讲客户端模块发起注册过程，不涉及服务端模块接口处理逻辑。  

## 1.初始化构造器
```bash
public DiscoveryClient(ApplicationInfoManager applicationInfoManager, final EurekaClientConfig config,     
    AbstractDiscoveryClientOptionalArgs args) {
    this(applicationInfoManager, config, args, ResolverUtils::randomize);
}
```
```bash
public DiscoveryClient(ApplicationInfoManager applicationInfoManager, final EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args, EndpointRandomizer randomizer) {
      // 构造方法新增备份eureka server获取信息类
      this(applicationInfoManager, config, args, new Provider<BackupRegistry>() {
          private volatile BackupRegistry backupRegistryInstance;

          @Override
          public synchronized BackupRegistry get() {
              if (backupRegistryInstance == null) {
                  String backupRegistryClassName = config.getBackupRegistryImpl();
                  if (null != backupRegistryClassName) {
                      try {
                          backupRegistryInstance = (BackupRegistry) Class.forName(backupRegistryClassName).newInstance();
                          logger.info("Enabled backup registry of type {}", backupRegistryInstance.getClass());
                      } catch (InstantiationException e) {
                          logger.error("Error instantiating BackupRegistry.", e);
                      } catch (IllegalAccessException e) {
                          logger.error("Error instantiating BackupRegistry.", e);
                      } catch (ClassNotFoundException e) {
                          logger.error("Error instantiating BackupRegistry.", e);
                      }
                  }

                  if (backupRegistryInstance == null) {
                      logger.warn("Using default backup registry implementation which does not do anything.");
                      backupRegistryInstance = new NotImplementedRegistryImpl();
                  }
              }

              return backupRegistryInstance;
          }
      }, randomizer);
  }
  ```
  ```bash
  @Inject
  DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                  Provider<BackupRegistry> backupRegistryProvider, EndpointRandomizer endpointRandomizer) {
      // 初始化健康检查处理和启动事件监听器              
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

      // 初始化配置实例
      clientConfig = config;
      staticClientConfig = clientConfig;
      transportConfig = config.getTransportConfig();
      instanceInfo = myInfo;
      if (myInfo != null) {
          appPathIdentifier = instanceInfo.getAppName() + "/" + instanceInfo.getId();
      } else {
          logger.warn("Setting instanceInfo to a passed in null value");
      }

      // 初始化备份获取Eureka server获取信息类
      this.backupRegistryProvider = backupRegistryProvider;
      this.endpointRandomizer = endpointRandomizer;
      this.urlRandomizer = new EndpointUtils.InstanceInfoBasedUrlRandomizer(instanceInfo);
      localRegionApps.set(new Applications());

      fetchRegistryGeneration = new AtomicLong(0);

      remoteRegionsToFetch = new AtomicReference<String>(clientConfig.fetchRegistryForRemoteRegions());
      remoteRegionsRef = new AtomicReference<>(remoteRegionsToFetch.get() == null ? null : remoteRegionsToFetch.get().split(","));

      // 判断是否从Eureka服务器拉取注册信息
      if (config.shouldFetchRegistry()) {
          // 注册表超时监控
          this.registryStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRY_PREFIX + "lastUpdateSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
      } else {
          this.registryStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
      }

      // 判断该实例是否注册到Eureka服务器，参与其他服务发现
      if (config.shouldRegisterWithEureka()) {
        // 心跳超时监控
          this.heartbeatStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRATION_PREFIX + "lastHeartbeatSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
      } else {
          this.heartbeatStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
      }

      logger.info("Initializing Eureka in region {}", clientConfig.getRegion());

      // 客户端配置为既不注册也不查询数据
      if (!config.shouldRegisterWithEureka() && !config.shouldFetchRegistry()) {
          logger.info("Client configured to neither register nor query for data.");
          scheduler = null;
          heartbeatExecutor = null;
          cacheRefreshExecutor = null;
          eurekaTransport = null;
          instanceRegionChecker = new InstanceRegionChecker(new PropertyBasedAzToRegionMapper(config), clientConfig.getRegion());

          // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
          // to work with DI"'"d DiscoveryClient
          DiscoveryManager.getInstance().setDiscoveryClient(this);
          DiscoveryManager.getInstance().setEurekaClientConfig(config);

          initTimestampMs = System.currentTimeMillis();
          logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
                  initTimestampMs, this.getApplications().size());

          return;  // no need to setup up an network tasks and we are done
      }

      /**
       + 构造器核心代码，主要做了三件事：
       + 1.创建线程池调度计划scheduler
       + 2.创建服务心跳检测线程池
       + 3.创建服务缓存刷新线程池
       */
      try {
          // default size of 2 - 1 each for heartbeat and cacheRefresh
          scheduler = Executors.newScheduledThreadPool(2,
                  new ThreadFactoryBuilder()
                          .setNameFormat("DiscoveryClient-%d")
                          .setDaemon(true)
                          .build());

          heartbeatExecutor = new ThreadPoolExecutor(
                  1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                  new SynchronousQueue<Runnable>(),
                  new ThreadFactoryBuilder()
                          .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
                          .setDaemon(true)
                          .build()
          );  // use direct handoff

          cacheRefreshExecutor = new ThreadPoolExecutor(
                  1, clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                  new SynchronousQueue<Runnable>(),
                  new ThreadFactoryBuilder()
                          .setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d")
                          .setDaemon(true)
                          .build()
          );  // use direct handoff

          eurekaTransport = new EurekaTransport();
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
          instanceRegionChecker = new InstanceRegionChecker(azToRegionMapper, clientConfig.getRegion());
      } catch (Throwable e) {
          throw new RuntimeException("Failed to initialize DiscoveryClient!", e);
      }

      // 如何从Eureka服务端获取实例失败，则从备份服务获取
      if (clientConfig.shouldFetchRegistry() && !fetchRegistry(false)) {
          fetchRegistryFromBackup();
      }

      // call and execute the pre registration handler before all background tasks (inc registration) is started
      if (this.preRegistrationHandler != null) {
          this.preRegistrationHandler.beforeRegistration();
      }

      if (clientConfig.shouldRegisterWithEureka() && clientConfig.shouldEnforceRegistrationAtInit()) {
          try {
              if (!register() ) {
                  throw new IllegalStateException("Registration error at startup. Invalid server response.");
              }
          } catch (Throwable th) {
              logger.error("Registration error at startup: {}", th.getMessage());
              throw new IllegalStateException(th);
          }
      }

      // finally, init the schedule tasks (e.g. cluster resolvers, heartbeat, instanceInfo replicator, fetch
      initScheduledTasks();

      try {
          Monitors.registerObject(this);
      } catch (Throwable e) {
          logger.warn("Cannot register timers", e);
      }

      // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
      // to work with DI"'"d DiscoveryClient
      DiscoveryManager.getInstance().setDiscoveryClient(this);
      DiscoveryManager.getInstance().setEurekaClientConfig(config);

      initTimestampMs = System.currentTimeMillis();
      logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
              initTimestampMs, this.getApplications().size());
  }
  ```

## 2.初始化定时器 
```bash
/**
   + Initializes all scheduled tasks.
   */
  private void initScheduledTasks() {
      if (clientConfig.shouldFetchRegistry()) {
          // registry cache refresh timer
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

      if (clientConfig.shouldRegisterWithEureka()) {
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
                  instanceInfoReplicator.onDemandUpdate();
              }
          };

          if (clientConfig.shouldOnDemandUpdateStatusChange()) {
              applicationInfoManager.registerStatusChangeListener(statusChangeListener);
          }

          instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
      } else {
          logger.info("Not registering with Eureka server per configuration");
      }
  }
```

## 3.总结
构造方法里面主要做了三件事：<br/>
* 创建线程池调度执行计划scheduler <br/>
* 创建服务获取线程池cacheRefreshExecutor <br/>
* 创建心跳检测线程池heartbeatExecutor <br/>
初始化定时任务为两个线程池指定任务，最后调用instanceInfoReplicator.start执行 <br/>
至此，客户端注册过程就完成了。
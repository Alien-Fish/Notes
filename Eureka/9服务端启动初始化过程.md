---
layout: post
#标题配置
title:  Eureka源码分析9-服务端启动初始化
#时间配置
date:   2020-06-22 21:00:00 +0800
#大类配置
categories: 服务注册
#小类配置
tag: Eureka
---

* content
{:toc}

## 说明
Eureka服务端其实普通的是一个webapp Servlet项目，早期做过web项目的都知道启动会初始化加载WEB-INF下web.xml，也就是当启动一个WEB项目时，容器包括（Undertow、JBoss、Tomcat等）首先会读取项目web.xml配置文件里的配置。

## eureka-server模块结构
![/styles/images/eureka/eureka-server.png]({{ '/styles/images/eureka/eureka-server.png' | prepend: site.baseurl  }})
这个模块里面没有包含代码目录，主要存放配置和启动web.xml文件，因为依赖包含了eureka-client、eureka-core、eureka-resources，其中eureka-client、eureka-core为功能代码实现模块，eureka-resources主要存在静态资源文件和jsp文件。

### 启动入口
```bash
<listener>
    <listener-class>com.netflix.eureka.EurekaBootStrap</listener-class>
</listener>
```
WEB-INF包下的web.xml是启动配置，其中<listener>为web应用程序定义监听器，执行EurekaBootStrap的contextInitialized方法执行初始化服务端。

### EurekaBootStrap初始化入口
```bash
/**
     * Initializes Eureka, including syncing up with other Eureka peers and publishing the registry.
     *
     * @see
     * javax.servlet.ServletContextListener#contextInitialized(javax.servlet.ServletContextEvent)
     */
    @Override
    public void contextInitialized(ServletContextEvent event) {
        try {
            // 初始化环境
            initEurekaEnvironment();
            // 初始化上下文
            initEurekaServerContext();

            // servlet上下文设置为当前eureka server上下文
            ServletContext sc = event.getServletContext();
            sc.setAttribute(EurekaServerContext.class.getName(), serverContext);
        } catch (Throwable e) {
            logger.error("Cannot bootstrap eureka server :", e);
            throw new RuntimeException("Cannot bootstrap eureka server :", e);
        }
    }
```
初始化Eureka，包括与其他Eureka对等点同步和发布注册表。

### 初始化环境
```bash
/**
  * Users can override to initialize the environment themselves.
  */
protected void initEurekaEnvironment() throws Exception {
    logger.info("Setting the eureka configuration..");

    // 获取eureka数据中心eureka.datacenter
    String dataCenter = ConfigurationManager.getConfigInstance().getString(EUREKA_DATACENTER);
    if (dataCenter == null) {
        logger.info("Eureka data center value eureka.datacenter is not set, defaulting to default");
        ConfigurationManager.getConfigInstance().setProperty(ARCHAIUS_DEPLOYMENT_DATACENTER, DEFAULT);
    } else {
        ConfigurationManager.getConfigInstance().setProperty(ARCHAIUS_DEPLOYMENT_DATACENTER, dataCenter);
    }
    // 获取eureka环境，开发、测试、生产等
    String environment = ConfigurationManager.getConfigInstance().getString(EUREKA_ENVIRONMENT);
    if (environment == null) {
        ConfigurationManager.getConfigInstance().setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, TEST);
        logger.info("Eureka environment value eureka.environment is not set, defaulting to test");
    }
}
```
用户可以通过重写来初始化环境。

### 初始化上下文
```bash
/**
  * init hook for server context. Override for custom logic.
  */
protected void initEurekaServerContext() throws Exception {
    EurekaServerConfig eurekaServerConfig = new DefaultEurekaServerConfig();

    // For backward compatibility
    JsonXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(), XStream.PRIORITY_VERY_HIGH);
    XmlXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(), XStream.PRIORITY_VERY_HIGH);

    logger.info("Initializing the eureka client...");
    logger.info(eurekaServerConfig.getJsonCodecName());
    ServerCodecs serverCodecs = new DefaultServerCodecs(eurekaServerConfig);

    ApplicationInfoManager applicationInfoManager = null;

    if (eurekaClient == null) {
        EurekaInstanceConfig instanceConfig = isCloud(ConfigurationManager.getDeploymentContext())
                ? new CloudInstanceConfig()
                : new MyDataCenterInstanceConfig();

        applicationInfoManager = new ApplicationInfoManager(
                instanceConfig, new EurekaConfigBasedInstanceInfoProvider(instanceConfig).get());

        EurekaClientConfig eurekaClientConfig = new DefaultEurekaClientConfig();
        eurekaClient = new DiscoveryClient(applicationInfoManager, eurekaClientConfig);
    } else {
        applicationInfoManager = eurekaClient.getApplicationInfoManager();
    }

    PeerAwareInstanceRegistry registry;
    if (isAws(applicationInfoManager.getInfo())) {
        registry = new AwsInstanceRegistry(
                eurekaServerConfig,
                eurekaClient.getEurekaClientConfig(),
                serverCodecs,
                eurekaClient
        );
        awsBinder = new AwsBinderDelegate(eurekaServerConfig, eurekaClient.getEurekaClientConfig(), registry, applicationInfoManager);
        awsBinder.start();
    } else {
        registry = new PeerAwareInstanceRegistryImpl(
                eurekaServerConfig,
                eurekaClient.getEurekaClientConfig(),
                serverCodecs,
                eurekaClient
        );
    }

    PeerEurekaNodes peerEurekaNodes = getPeerEurekaNodes(
            registry,
            eurekaServerConfig,
            eurekaClient.getEurekaClientConfig(),
            serverCodecs,
            applicationInfoManager
    );

    serverContext = new DefaultEurekaServerContext(
            eurekaServerConfig,
            serverCodecs,
            registry,
            peerEurekaNodes,
            applicationInfoManager
    );

    EurekaServerContextHolder.initialize(serverContext);

    serverContext.initialize();
    logger.info("Initialized server context");

    // Copy registry from neighboring eureka node
    int registryCount = registry.syncUp();
    registry.openForTraffic(applicationInfoManager, registryCount);

    // Register all monitoring statistics.
    EurekaMonitors.registerAllStats();
}
```
主要流程简介：  
1.采用非公平读写锁方式获取读锁，当其他线程持有写锁是，当前注册线程取锁失败，等待写锁释放，保证实例状态最新。  
2.根据实例名称判断是否存在历史缓存实例，不存在则新增实例到缓存，并且更新续租客户端数量和每分钟发送续租次数的阈值。  
3.Eureka根据状态不能解决客户端-服务端状态同步的问题，所以引入覆盖状态，如果当前实例覆盖状态不是UNKNOWN，则更新注册实例的覆盖状态为当前实例的覆盖状态，注册时默认为UNKNOWN。  
4.根据覆盖状态规则更新状态，覆盖状态规则使用组合模式设计，有四种实现，分别为：DownOrStartingRule、OverrideExistsRule、AsgEnabledRule、LeaseExistsRule。  
5.删除缓存数据。  
6.释放读写锁。  

### 其他
#### 服务实例状态
```bash
public enum InstanceStatus {
        UP, // Ready to receive traffic
        DOWN, // Do not send traffic- healthcheck callback failed
        STARTING, // Just about starting- initializations to be done - do not
        // send traffic
        OUT_OF_SERVICE, // Intentionally shutdown for traffic
        UNKNOWN;

        public static InstanceStatus toEnum(String s) {
            if (s != null) {
                try {
                    return InstanceStatus.valueOf(s.toUpperCase());
                } catch (IllegalArgumentException e) {
                    // ignore and fall through to unknown
                    logger.debug("illegal argument supplied to InstanceStatus.valueOf: {}, defaulting to {}", s, UNKNOWN);
                }
            }
            return UNKNOWN;
        }
    }
```

| 状态  | 描述 |
| :-----| :---- | 	
| STARTING 	| 实例初始化状态 |
| DOWN 	| 服务下线，当健康检查失败时，实例的状态转变到DOWN |
| UP 	| 正常状态 |
| OUT_OF_SERVICE | 	服务已启动但是不可用 |
| UNKNOWN | 	未知状态，可能是新实例注册未完成 |


### 总结
略
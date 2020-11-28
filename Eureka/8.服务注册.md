---
layout: post
#标题配置
title:  Eureka源码分析8-服务端处理实例注册流程
#时间配置
date:   2020-05-16 22:00:00 +0800
#大类配置
categories: 服务注册
#小类配置
tag: Eureka
---

* content
{:toc}

## 说明
Eureka客户端启动后，会发送一个实例注册请求到服务端，服务端接收请求验证保存实例数据，集群同步到其他节点。  
这里只讲服务端注册流程，客户端发起注册流程参考5-客户端服务续约，差不多一样的调用过程。

## 服务端注册流程时序图
![/styles/images/eureka/eureka_server_register.png]({{ '/styles/images/eureka/eureka_server_register.png' | prepend: site.baseurl  }})

### 注册入口
```bash
@Path("/{version}/apps")
@Produces({"application/xml", "application/json"})
public class ApplicationsResource {
    private static final String HEADER_ACCEPT = "Accept";
    private static final String HEADER_ACCEPT_ENCODING = "Accept-Encoding";
    private static final String HEADER_CONTENT_ENCODING = "Content-Encoding";
    private static final String HEADER_CONTENT_TYPE = "Content-Type";
    private static final String HEADER_GZIP_VALUE = "gzip";
    private static final String HEADER_JSON_VALUE = "json";

    private final EurekaServerConfig serverConfig;
    private final PeerAwareInstanceRegistry registry;
    private final ResponseCache responseCache;

    @Inject
    ApplicationsResource(EurekaServerContext eurekaServer) {
        this.serverConfig = eurekaServer.getServerConfig();
        this.registry = eurekaServer.getRegistry();
        this.responseCache = registry.getResponseCache();
    }

    /**
     * Gets information about a particular {@link com.netflix.discovery.shared.Application}.
     *
     * @param version
     *            the version of the request.
     * @param appId
     *            the unique application identifier (which is the name) of the
     *            application.
     * @return information about a particular application.
     */
    @Path("{appId}")
    public ApplicationResource getApplicationResource(
            @PathParam("version") String version,
            @PathParam("appId") String appId) {
        CurrentRequestVersion.set(Version.toEnum(version));
        try {
            return new ApplicationResource(appId, serverConfig, registry);
        } finally {
            CurrentRequestVersion.remove();
        }
    }
}
```
客户端请求地址http://hostname:port/eureka/{version}/apps/{appId}。  
服务端接口使用jsr-api方式，这里@Path("/{version}/apps")映射客户端路径，也就是getApplicationResource方法接收客户端实例注册请求。  
这个方法比较简单，接收到注册请求后会new一个特定的实例请求处理类ApplicationResource，开始处理客户端实例注册。

### 实例请求处理
```bash
@Produces({"application/xml", "application/json"})
public class ApplicationResource {
    private final String appName;
    private final EurekaServerConfig serverConfig;
    private final PeerAwareInstanceRegistry registry;
    private final ResponseCache responseCache;

    /**
     * Registers information about a particular instance for an
     * {@link com.netflix.discovery.shared.Application}.
     *
     * @param info
     *            {@link InstanceInfo} information of the instance.
     * @param isReplication
     *            a header parameter containing information whether this is
     *            replicated from other nodes.
     */
    @POST
    @Consumes({"application/json", "application/xml"})
    public Response addInstance(InstanceInfo info,
                                @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
        logger.debug("Registering instance {} (replication={})", info.getId(), isReplication);
        // validate that the instanceinfo contains all the necessary required fields
        if (isBlank(info.getId())) {
            return Response.status(400).entity("Missing instanceId").build();
        } else if (isBlank(info.getHostName())) {
            return Response.status(400).entity("Missing hostname").build();
        } else if (isBlank(info.getIPAddr())) {
            return Response.status(400).entity("Missing ip address").build();
        } else if (isBlank(info.getAppName())) {
            return Response.status(400).entity("Missing appName").build();
        } else if (!appName.equals(info.getAppName())) {
            return Response.status(400).entity("Mismatched appName, expecting " + appName + " but was " + info.getAppName()).build();
        } else if (info.getDataCenterInfo() == null) {
            return Response.status(400).entity("Missing dataCenterInfo").build();
        } else if (info.getDataCenterInfo().getName() == null) {
            return Response.status(400).entity("Missing dataCenterInfo Name").build();
        }

        // handle cases where clients may be registering with bad DataCenterInfo with missing data
        DataCenterInfo dataCenterInfo = info.getDataCenterInfo();
        if (dataCenterInfo instanceof UniqueIdentifier) {
            String dataCenterInfoId = ((UniqueIdentifier) dataCenterInfo).getId();
            if (isBlank(dataCenterInfoId)) {
                boolean experimental = "true".equalsIgnoreCase(serverConfig.getExperimental("registration.validation.dataCenterInfoId"));
                if (experimental) {
                    String entity = "DataCenterInfo of type " + dataCenterInfo.getClass() + " must contain a valid id";
                    return Response.status(400).entity(entity).build();
                } else if (dataCenterInfo instanceof AmazonInfo) {
                    AmazonInfo amazonInfo = (AmazonInfo) dataCenterInfo;
                    String effectiveId = amazonInfo.get(AmazonInfo.MetaDataKey.instanceId);
                    if (effectiveId == null) {
                        amazonInfo.getMetadata().put(AmazonInfo.MetaDataKey.instanceId.getName(), info.getId());
                    }
                } else {
                    logger.warn("Registering DataCenterInfo of type {} without an appropriate id", dataCenterInfo.getClass());
                }
            }
        }

        registry.register(info, "true".equals(isReplication));
        return Response.status(204).build();  // 204 to be backwards compatible
    }
}
```
这一步主要是验证客户端注册实例参数是否合法(那一堆if else)，DataCenterInfo判断是否注册到坏的数据中心导致数据丢失。  
验证通过后调用PeerAwareInstanceRegistry的register注册实例，最后register过程无异常的话响应成功状态204到客户端。

### PeerAwareInstanceRegistry
```bash
@Singleton
public class PeerAwareInstanceRegistryImpl extends AbstractInstanceRegistry implements PeerAwareInstanceRegistry {

    /**
     * Registers the information about the {@link InstanceInfo} and replicates
     * this information to all peer eureka nodes. If this is replication event
     * from other replica nodes then it is not replicated.
     *
     * @param info
     *            the {@link InstanceInfo} to be registered and replicated.
     * @param isReplication
     *            true if this is a replication event from other replica nodes,
     *            false otherwise.
     */
    @Override
    public void register(final InstanceInfo info, final boolean isReplication) {
        // 默认续租时间
        int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
        // 判断客户端是否配置续租时间，覆盖默认值
        if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
            leaseDuration = info.getLeaseInfo().getDurationInSecs();
        }
        // 调用父类注册方法
        super.register(info, leaseDuration, isReplication);
        // 同步注册实例信息到其他集群节点
        replicateToPeers(Action.Register, info.getAppName(), info.getId(), info, null, isReplication);
    }

}
```
获取实例续租周期，调用父类register方法

### AbstractInstanceRegistry
```bash
/**
 * Handles all registry requests from eureka clients.
 *
 * <p>
 * Primary operations that are performed are the
 * <em>Registers</em>, <em>Renewals</em>, <em>Cancels</em>, <em>Expirations</em>, and <em>Status Changes</em>. The
 * registry also stores only the delta operations
 * </p>
 *
 * @author Karthik Ranganathan
 *
 */
public abstract class AbstractInstanceRegistry implements InstanceRegistry {
    private static final String[] EMPTY_STR_ARRAY = new String[0];
    private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry
            = new ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>();
    protected Map<String, RemoteRegionRegistry> regionNameVSRemoteRegistry = new HashMap<String, RemoteRegionRegistry>();
    protected final ConcurrentMap<String, InstanceStatus> overriddenInstanceStatusMap = CacheBuilder
            .newBuilder().initialCapacity(500)
            .expireAfterAccess(1, TimeUnit.HOURS)
            .<String, InstanceStatus>build().asMap();

    // CircularQueues here for debugging/statistics purposes only
    private final CircularQueue<Pair<Long, String>> recentRegisteredQueue;
    private final CircularQueue<Pair<Long, String>> recentCanceledQueue;
    private ConcurrentLinkedQueue<RecentlyChangedItem> recentlyChangedQueue = new ConcurrentLinkedQueue<RecentlyChangedItem>();

    private final ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    private final Lock read = readWriteLock.readLock();
    private final Lock write = readWriteLock.writeLock();
    protected final Object lock = new Object();

    private Timer deltaRetentionTimer = new Timer("Eureka-DeltaRetentionTimer", true);
    private Timer evictionTimer = new Timer("Eureka-EvictionTimer", true);
    private final MeasuredRate renewsLastMin;

    private final AtomicReference<EvictionTask> evictionTaskRef = new AtomicReference<EvictionTask>();

    protected String[] allKnownRemoteRegions = EMPTY_STR_ARRAY;
    protected volatile int numberOfRenewsPerMinThreshold;
    protected volatile int expectedNumberOfClientsSendingRenews;

    protected final EurekaServerConfig serverConfig;
    protected final EurekaClientConfig clientConfig;
    protected final ServerCodecs serverCodecs;
    protected volatile ResponseCache responseCache;
    
    /**
     * 注册续租期的新实例
     * Registers a new instance with a given duration.
     *
     * @see com.netflix.eureka.lease.LeaseManager#register(java.lang.Object, int, boolean)
     */
    public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
        try {
            // 上读锁
            read.lock();
            // 判断集群缓存中是否存在当前注册实例
            Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
            REGISTER.increment(isReplication);
            if (gMap == null) {
                // 不能存在则新增
                final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
                gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
                if (gMap == null) {
                    gMap = gNewMap;
                }
            }
            Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
            // 保留最后一个脏时间戳，而不覆盖它(如果已经有租约)
            // Retain the last dirty timestamp without overwriting it, if there is already a lease
            if (existingLease != null && (existingLease.getHolder() != null)) {
                Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
                Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
                logger.debug("Existing lease found (existing={}, provided={}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);

                // 这里判断缓存中的实例持有者和当前注册实例的时间戳大小，取最新的实例作为注册实例
                // this is a > instead of a >= because if the timestamps are equal, we still take the remote transmitted
                // InstanceInfo instead of the server local copy.
                if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                    logger.warn("There is an existing lease and the existing lease's dirty timestamp {} is greater" +
                            " than the one that is being registered {}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                    logger.warn("Using the existing instanceInfo instead of the new instanceInfo as the registrant");
                    registrant = existingLease.getHolder();
                }
            } else {
                // The lease does not exist and hence it is a new registration
                synchronized (lock) {
                    if (this.expectedNumberOfClientsSendingRenews > 0) {
                        // Since the client wants to register it, increase the number of clients sending renews
                        // 更新续租客户端数量+1
                        this.expectedNumberOfClientsSendingRenews = this.expectedNumberOfClientsSendingRenews + 1;
                        // 更新每分钟发送续租次数的阈值
                        updateRenewsPerMinThreshold();
                    }
                }
                logger.debug("No previous lease information found; it is new registration");
            }
            // 创建实例租约对象
            Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
            if (existingLease != null) {
                lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
            }
            gMap.put(registrant.getId(), lease);
            // 新注册队列中添加当前注册的信息
            recentRegisteredQueue.add(new Pair<Long, String>(
                    System.currentTimeMillis(),
                    registrant.getAppName() + "(" + registrant.getId() + ")"));

            // 判断更新覆盖状态
            // This is where the initial state transfer of overridden status happens
            if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
                logger.debug("Found overridden status {} for instance {}. Checking to see if needs to be add to the "
                                + "overrides", registrant.getOverriddenStatus(), registrant.getId());
                if (!overriddenInstanceStatusMap.containsKey(registrant.getId())) {
                    logger.info("Not found overridden id {} and hence adding it", registrant.getId());
                    overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
                }
            }
            // 取当前实例的覆盖状态
            InstanceStatus overriddenStatusFromMap = overriddenInstanceStatusMap.get(registrant.getId());
            if (overriddenStatusFromMap != null) {
                logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
                registrant.setOverriddenStatus(overriddenStatusFromMap);
            }

            // 根据状态规则重新更新状态
            // Set the status based on the overridden status rules
            InstanceStatus overriddenInstanceStatus = getOverriddenInstanceStatus(registrant, existingLease, isReplication);
            registrant.setStatusWithoutDirty(overriddenInstanceStatus);

            // If the lease is registered with UP status, set lease service up timestamp
            if (InstanceStatus.UP.equals(registrant.getStatus())) {
                lease.serviceUp();
            }
            registrant.setActionType(ActionType.ADDED);
            recentlyChangedQueue.add(new RecentlyChangedItem(lease));
            registrant.setLastUpdatedTimestamp();
            // 删除缓存数据
            invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
            logger.info("Registered instance {}/{} with status {} (replication={})",
                    registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication);
        } finally {
            // 释放锁
            read.unlock();
        }
    }

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
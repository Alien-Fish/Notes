---
layout: post
#标题配置
title:  Eureka源码分析4-客户端缓存服务刷新
#时间配置
date:   2019-11-15 22:25:00 +0800
#大类配置
categories: 服务注册
#小类配置
tag: Eureka
---

* content
{:toc}

## 说明
客户端启动过程中会注册到服务端，并定时去请求服务端拉取注册数据缓存到本地，第一次本地无缓存时或者版本号不对应时会拉取全部注册数据，否则增量拉取注册数据。  

第三节中讲了客户端初始化过程，客户端启动会初始化三个定时任务线程，其中一个是TimedSupervisorTask负责周期性(默认每三十秒一次)执行拉取服务端注册数据，其中HeartbeatThread线程中为具体的执行逻辑。  

下面来看下Eureka client缓存服务列表的源码

## 1.本地缓存
```bash
private final AtomicReference<Applications> localRegionApps = new AtomicReference<Applications>();
```
本地缓存使用AtomicReference原子性操作，防止并发修改，不会引发线程安全问题。
## 2.TimedSupervisorTask任务
```bash
private void initScheduledTasks() {
    // 当配置获取注册信息为true(默认true)，执行周期定时任务
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
}
```
初始化定时器，默认每隔30秒执行一次。
## 3.CacheRefreshThread
```bash
/**
 * The task that fetches the registry information at specified intervals.
 *
 */
class CacheRefreshThread implements Runnable {
    public void run() {
        refreshRegistry();
    }
}
```
## 4.CacheRefreshThread执行逻辑
```bash
@VisibleForTesting
void refreshRegistry() {
    try {
        boolean isFetchingRemoteRegionRegistries = isFetchingRemoteRegionRegistries();

        // 判断是否需要全量获取标志
        boolean remoteRegionsModified = false;
        // This makes sure that a dynamic change to remote regions to fetch is honored.
        String latestRemoteRegions = clientConfig.fetchRegistryForRemoteRegions();
        if (null != latestRemoteRegions) {
            String currentRemoteRegions = remoteRegionsToFetch.get();
            if (!latestRemoteRegions.equals(currentRemoteRegions)) {
                // Both remoteRegionsToFetch and AzToRegionMapper.regionsToFetch need to be in sync
                synchronized (instanceRegionChecker.getAzToRegionMapper()) {
                    if (remoteRegionsToFetch.compareAndSet(currentRemoteRegions, latestRemoteRegions)) {
                        String[] remoteRegions = latestRemoteRegions.split(",");
                        remoteRegionsRef.set(remoteRegions);
                        instanceRegionChecker.getAzToRegionMapper().setRegionsToFetch(remoteRegions);
                        remoteRegionsModified = true;
                    } else {
                        logger.info("Remote regions to fetch modified concurrently," +
                                " ignoring change from {} to {}", currentRemoteRegions, latestRemoteRegions);
                    }
                }
            } else {
                // Just refresh mapping to reflect any DNS/Property change
                instanceRegionChecker.getAzToRegionMapper().refreshMapping();
            }
        }

        // 拉取注册方法
        boolean success = fetchRegistry(remoteRegionsModified);
        if (success) {
            registrySize = localRegionApps.get().size();
            lastSuccessfulRegistryFetchTimestamp = System.currentTimeMillis();
        }

        // 打印日志
        if (logger.isDebugEnabled()) {
            StringBuilder allAppsHashCodes = new StringBuilder();
            allAppsHashCodes.append("Local region apps hashcode: ");
            allAppsHashCodes.append(localRegionApps.get().getAppsHashCode());
            allAppsHashCodes.append(", is fetching remote regions? ");
            allAppsHashCodes.append(isFetchingRemoteRegionRegistries);
            for (Map.Entry<String, Applications> entry : remoteRegionVsApps.entrySet()) {
                allAppsHashCodes.append(", Remote region: ");
                allAppsHashCodes.append(entry.getKey());
                allAppsHashCodes.append(" , apps hashcode: ");
                allAppsHashCodes.append(entry.getValue().getAppsHashCode());
            }
            logger.debug("Completed cache refresh task for discovery. All Apps hash code is {} ",
                    allAppsHashCodes);
        }
    } catch (Throwable e) {
        logger.error("Cannot fetch registry from server", e);
    }
}
```
## 5.拉取注册数据方法
```bash
/**
 * Fetches the registry information.
 *
 * <p>
 * This method tries to get only deltas after the first fetch unless there
 * is an issue in reconciling eureka server and client registry information.
 * </p>
 *
 * @param forceFullRegistryFetch Forces a full registry fetch.
 *
 * @return true if the registry was fetched
 */
private boolean fetchRegistry(boolean forceFullRegistryFetch) {
    Stopwatch tracer = FETCH_REGISTRY_TIMER.start();

    try {
        // If the delta is disabled or if it is the first time, get all
        // applications
        Applications applications = getApplications();

        // 判断是否禁用增量获取or是否第一次获取服务列表
        if (clientConfig.shouldDisableDelta()
                || (!Strings.isNullOrEmpty(clientConfig.getRegistryRefreshSingleVipAddress()))
                || forceFullRegistryFetch
                || (applications == null)
                || (applications.getRegisteredApplications().size() == 0)
                || (applications.getVersion() == -1)) //Client application does not have latest library supporting delta
        { // 全量拉取
            logger.info("Disable delta property : {}", clientConfig.shouldDisableDelta());
            logger.info("Single vip registry refresh property : {}", clientConfig.getRegistryRefreshSingleVipAddress());
            logger.info("Force full registry fetch : {}", forceFullRegistryFetch);
            logger.info("Application is null : {}", (applications == null));
            logger.info("Registered Applications size is zero : {}",
                    (applications.getRegisteredApplications().size() == 0));
            logger.info("Application version is -1: {}", (applications.getVersion() == -1));
            getAndStoreFullRegistry();
        } else { // 增量拉取
            getAndUpdateDelta(applications);
        }
        // 缓存实例数据后，增量获取时会对AppsHashCode进行对比，不同则全量获取
        applications.setAppsHashCode(applications.getReconcileHashCode());
        logTotalInstances();
    } catch (Throwable e) {
        logger.error(PREFIX + "{} - was unable to refresh its cache! status = {}", appPathIdentifier, e.getMessage(), e);
        return false;
    } finally {
        if (tracer != null) {
            tracer.stop();
        }
    }

    // Notify about cache refresh before updating the instance remote status
    onCacheRefreshed();

    // Update remote status based on refreshed data held in the cache
    updateInstanceRemoteStatus();

    // registry was fetched successfully, so return true
    return true;
}
```
## 6.全量拉取注册数据方法
```bash
/**
 * Gets the full registry information from the eureka server and stores it locally.
 * When applying the full registry, the following flow is observed:
 *
 * if (update generation have not advanced (due to another thread))
 *   atomically set the registry to the new registry
 * fi
 *
 * @return the full registry information.
 * @throws Throwable
 *             on error.
 */
private void getAndStoreFullRegistry() throws Throwable {
    long currentUpdateGeneration = fetchRegistryGeneration.get();

    logger.info("Getting all instance registry info from the eureka server");

    Applications apps = null;
    // 请求/eureka/apps接口，拉取所有服务实例信息
    EurekaHttpResponse<Applications> httpResponse = clientConfig.getRegistryRefreshSingleVipAddress() == null
            ? eurekaTransport.queryClient.getApplications(remoteRegionsRef.get())
            : eurekaTransport.queryClient.getVip(clientConfig.getRegistryRefreshSingleVipAddress(), remoteRegionsRef.get());
    if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
        apps = httpResponse.getEntity();
    }
    logger.info("The response status is {}", httpResponse.getStatusCode());

    if (apps == null) {
        logger.error("The application is null for some reason. Not storing this information");
    } else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {
        localRegionApps.set(this.filterAndShuffle(apps));
        logger.debug("Got full registry with apps hashcode {}", apps.getAppsHashCode());
    } else {
        logger.warn("Not updating applications as another thread is updating it already");
    }
}
```
## 7.增量拉取注册数据方法
```bash
/**
 * Get the delta registry information from the eureka server and update it locally.
 * When applying the delta, the following flow is observed:
 *
 * if (update generation have not advanced (due to another thread))
 *   atomically try to: update application with the delta and get reconcileHashCode
 *   abort entire processing otherwise
 *   do reconciliation if reconcileHashCode clash
 * fi
 *
 * @return the client response
 * @throws Throwable on error
 */
private void getAndUpdateDelta(Applications applications) throws Throwable {
    long currentUpdateGeneration = fetchRegistryGeneration.get();

    Applications delta = null;
    EurekaHttpResponse<Applications> httpResponse = eurekaTransport.queryClient.getDelta(remoteRegionsRef.get());
    if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
        delta = httpResponse.getEntity();
    }

    // 判断delta是否为空，为空则全量拉取
    if (delta == null) {
        logger.warn("The server does not allow the delta revision to be applied because it is not safe. "
                - "Hence got the full registry.");
        getAndStoreFullRegistry();
    } else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {
        // CAS防止增量信息并发修改
        logger.debug("Got delta update with apps hashcode {}", delta.getAppsHashCode());
        String reconcileHashCode = "";
        if (fetchRegistryUpdateLock.tryLock()) {
            try {
                updateDelta(delta);
                reconcileHashCode = getReconcileHashCode(applications);
            } finally {
                fetchRegistryUpdateLock.unlock();
            }
        } else {
            logger.warn("Cannot acquire update lock, aborting getAndUpdateDelta");
        }
        // 拉取增量成功后，检查hashcode是否一样，不相等则全量拉取
        // There is a diff in number of instances for some reason
        if (!reconcileHashCode.equals(delta.getAppsHashCode()) || clientConfig.shouldLogDeltaDiff()) {
            reconcileAndLogDifference(delta, reconcileHashCode);  // this makes a remoteCall
        }
    } else {
        logger.warn("Not updating application delta as another thread is updating it already");
        logger.debug("Ignoring delta update with apps hashcode {}, as another thread is updating it already", delta.getAppsHashCode());
    }
}
```
## 8.请求服务端apps/接口
```bash
private EurekaHttpResponse<Applications> getApplicationsInternal(String urlPath, String[] regions) {
    Response response = null;
    try {
        WebTarget webTarget = jerseyClient.target(serviceUrl).path(urlPath);
        if (regions != null && regions.length > 0) {
            webTarget = webTarget.queryParam("regions", StringUtil.join(regions));
        }
        Builder requestBuilder = webTarget.request();
        addExtraProperties(requestBuilder);
        addExtraHeaders(requestBuilder);
        response = requestBuilder.accept(MediaType.APPLICATION_JSON_TYPE).get();

        Applications applications = null;
        if (response.getStatus() == Status.OK.getStatusCode() && response.hasEntity()) {
            applications = response.readEntity(Applications.class);
        }
        return anEurekaHttpResponse(response.getStatus(), applications).headers(headersOf(response)).build();
    } finally {
        if (logger.isDebugEnabled()) {
            logger.debug("Jersey2 HTTP GET {}/{}; statusCode={}", serviceUrl, urlPath, response == null ? "N/A" : response.getStatus());
        }
        if (response != null) {
            response.close();
        }
    }
}
```
## 9.总结
* EurekaClient第一次全量拉取，定时增量拉取应用服务实例信息，保存在缓存中。
* EurekaClient增量拉取失败，或者增量拉取之后对比hashcode发现不一致，就会执行全量拉取，这样避免了网络某时段分片带来的问题。
* 同时对于服务调用，如果涉及到ribbon负载均衡，那么ribbon对于这个实例列表也有自己的缓存，这个缓存定时从EurekaClient的缓存更新

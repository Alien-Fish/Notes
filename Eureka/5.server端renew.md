---
layout: post
#标题配置
title:  Eureka源码分析5-客户端服务续约
#时间配置
date:   2019-11-16 00:28:00 +0800
#大类配置
categories: 服务注册
#小类配置
tag: Eureka
---

* content
{:toc}

## 说明
Eureka客户端注册到服务端后，会定时向服务端发送服务续约(心跳)，告诉服务端我还存活着，不要把我剔除。  

## 客户端调用时序图
![/styles/images/eureka/eureka_renew_dg.png]({{ '/styles/images/eureka/eureka_renew_dg.png' | prepend: site.baseurl  }})

## 客户端请求
回顾一下第三节的内容，其中有一段是创建线程池调度计划scheduler，以及创建了创建服务心跳检测线程池，如下：
```bash
if (clientConfig.shouldRegisterWithEureka()) {
    // 心跳(续约)频率，默认30秒
    int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
    // 请求超时异常发送延迟时间边界值，默认10秒，每次延迟2的倍数增长
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
    ......
}
```
HeartbeatThread为周期执行renew心跳的线程。
```bash
/**
 * The heartbeat task that renews the lease in the given intervals.
 */
private class HeartbeatThread implements Runnable {

    public void run() {
        if (renew()) {
            lastSuccessfulHeartbeatTimestamp = System.currentTimeMillis();
        }
    }
}
```
新建一个HeartbeatThread线程类，实现run方法，执行renew()，发送客户端心跳请求到服务端。 
```bash
/**
 * Renew with the eureka service by making the appropriate REST call
 */
boolean renew() {
    EurekaHttpResponse<InstanceInfo> httpResponse;
    try {
        httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), instanceInfo.getId(), instanceInfo, null);
        logger.debug(PREFIX + "{} - Heartbeat status: {}", appPathIdentifier, httpResponse.getStatusCode());
        if (httpResponse.getStatusCode() == Status.NOT_FOUND.getStatusCode()) {
            REREGISTER_COUNTER.increment();
            logger.info(PREFIX + "{} - Re-registering apps/{}", appPathIdentifier, instanceInfo.getAppName());
            long timestamp = instanceInfo.setIsDirtyWithTime();
            boolean success = register();
            if (success) {
                instanceInfo.unsetIsDirty(timestamp);
            }
            return success;
        }
        return httpResponse.getStatusCode() == Status.OK.getStatusCode();
    } catch (Throwable e) {
        logger.error(PREFIX + "{} - was unable to send heartbeat!", appPathIdentifier, e);
        return false;
    }
}
```

## 服务端调用时序图
![/styles/images/eureka/eureka_renew_server_dg.png]({{ '/styles/images/eureka/eureka_renew_server_dg.png' | prepend: site.baseurl  }})

## 服务端接收请求
```bash
/**
 * A put request for renewing lease from a client instance.
 *
 * @param isReplication
 *            a header parameter containing information whether this is
 *            replicated from other nodes.
 * @param overriddenStatus
 *            overridden status if any.
 * @param status
 *            the {@link InstanceStatus} of the instance.
 * @param lastDirtyTimestamp
 *            last timestamp when this instance information was updated.
 * @return response indicating whether the operation was a success or
 *         failure.
 */
@PUT
public Response renewLease(
        @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication,
        @QueryParam("overriddenstatus") String overriddenStatus,
        @QueryParam("status") String status,
        @QueryParam("lastDirtyTimestamp") String lastDirtyTimestamp) {
    boolean isFromReplicaNode = "true".equals(isReplication);
    boolean isSuccess = registry.renew(app.getName(), id, isFromReplicaNode);

    // Not found in the registry, immediately ask for a register
    if (!isSuccess) {
        logger.warn("Not Found (Renew): {} - {}", app.getName(), id);
        return Response.status(Status.NOT_FOUND).build();
    }
    // Check if we need to sync based on dirty time stamp, the client
    // instance might have changed some value
    Response response;
    if (lastDirtyTimestamp != null && serverConfig.shouldSyncWhenTimestampDiffers()) {
        response = this.validateDirtyTimestamp(Long.valueOf(lastDirtyTimestamp), isFromReplicaNode);
        // Store the overridden status since the validation found out the node that replicates wins
        if (response.getStatus() == Response.Status.NOT_FOUND.getStatusCode()
                && (overriddenStatus != null)
                && !(InstanceStatus.UNKNOWN.name().equals(overriddenStatus))
                && isFromReplicaNode) {
            registry.storeOverriddenStatusIfRequired(app.getAppName(), id, InstanceStatus.valueOf(overriddenStatus));
        }
    } else {
        response = Response.ok().build();
    }
    logger.debug("Found (Renew): {} - {}; reply status={}", app.getName(), id, response.getStatus());
    return response;
}
```
调用PeerAwareInstanceRegistryImpl实现类中的renew方法
```bash
/*
 * (non-Javadoc)
 *
 * @see com.netflix.eureka.registry.InstanceRegistry#renew(java.lang.String,
 * java.lang.String, long, boolean)
 */
public boolean renew(final String appName, final String id, final boolean isReplication) {
    // 调用父类renew更新实例状态，再调replicateToPeers方法同步更新操作到其他节点
    if (super.renew(appName, id, isReplication)) {
        replicateToPeers(Action.Heartbeat, appName, id, null, null, isReplication);
        return true;
    }
    return false;
}
```
调用父类AbstractInstanceRegistry中的renew方法
```bash
/**
 * Marks the given instance of the given app name as renewed, and also marks whether it originated from
 * replication.
 *
 * @see com.netflix.eureka.lease.LeaseManager#renew(java.lang.String, java.lang.String, boolean)
 */
public boolean renew(String appName, String id, boolean isReplication) {
    RENEW.increment(isReplication);
    Map<String, Lease<InstanceInfo>> gMap = registry.get(appName);
    Lease<InstanceInfo> leaseToRenew = null;
    if (gMap != null) {
        leaseToRenew = gMap.get(id);
    }
    if (leaseToRenew == null) {
        RENEW_NOT_FOUND.increment(isReplication);
        logger.warn("DS: Registry: lease doesn't exist, registering resource: {} - {}", appName, id);
        return false;
    } else {
        InstanceInfo instanceInfo = leaseToRenew.getHolder();
        if (instanceInfo != null) {
            // touchASGCache(instanceInfo.getASGName());
            InstanceStatus overriddenInstanceStatus = this.getOverriddenInstanceStatus(
                    instanceInfo, leaseToRenew, isReplication);
            if (overriddenInstanceStatus == InstanceStatus.UNKNOWN) {
                logger.info("Instance status UNKNOWN possibly due to deleted override for instance {}"
                        - "; re-register required", instanceInfo.getId());
                RENEW_NOT_FOUND.increment(isReplication);
                return false;
            }
            if (!instanceInfo.getStatus().equals(overriddenInstanceStatus)) {
                logger.info(
                        "The instance status {} is different from overridden instance status {} for instance {}. "
                                - "Hence setting the status to overridden status", instanceInfo.getStatus().name(),
                                instanceInfo.getOverriddenStatus().name(),
                                instanceInfo.getId());
                instanceInfo.setStatusWithoutDirty(overriddenInstanceStatus);

            }
        }
        renewsLastMin.increment();
        leaseToRenew.renew();
        return true;
    }
}
```
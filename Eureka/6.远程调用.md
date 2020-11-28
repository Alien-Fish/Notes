---
layout: post
#标题配置
title:  Eureka源码分析6-远程通讯模块浅析
#时间配置
date:   2020-01-02 23:11:00 +0800
#大类配置
categories: 服务注册
#小类配置
tag: Eureka
---

* content
{:toc}

## 说明
从第一节知道Eureka内部模块包括eureka-client，eureka-server模块，即采用了C-S架构模式，服务端之间也会存在集群通信。本文通过源码分析eureka客户端和服务端通信过程，学习有哪些通信接口及设计模式。

## 客户端接口
EurekaHttpClient中维护了通讯交互所需的基本接口，根据不同通讯级别分为Low level和Top level实现。

### 接口列表
```bash
package com.netflix.discovery.shared.transport;

import com.netflix.appinfo.InstanceInfo;
import com.netflix.appinfo.InstanceInfo.InstanceStatus;
import com.netflix.discovery.shared.Application;
import com.netflix.discovery.shared.Applications;

/**
 * Low level Eureka HTTP client API.
 *
 * @author Tomasz Bak
 */
public interface EurekaHttpClient {
    /**
     * 服务注册
     * @param info
     * @return
     */
    EurekaHttpResponse<Void> register(InstanceInfo info);

    /**
     * 服务下线
     * @param appName
     * @param id
     * @return
     */
    EurekaHttpResponse<Void> cancel(String appName, String id);

    /**
     * 服务续约
     * @param appName
     * @param id
     * @param info
     * @param overriddenStatus
     * @return
     */
    EurekaHttpResponse<InstanceInfo> sendHeartBeat(String appName, String id, InstanceInfo info, InstanceStatus overriddenStatus);

    /**
     * 服务状态更新
     * @param appName
     * @param id
     * @param newStatus
     * @param info
     * @return
     */
    EurekaHttpResponse<Void> statusUpdate(String appName, String id, InstanceStatus newStatus, InstanceInfo info);

    /**
     * 覆盖状态，续约，注册的时候覆盖状态
     * @param appName
     * @param id
     * @param info
     * @return
     */
    EurekaHttpResponse<Void> deleteStatusOverride(String appName, String id, InstanceInfo info);

    /**
     * 获取服务注册列表
     * @param regions
     * @return
     */
    EurekaHttpResponse<Applications> getApplications(String... regions);

    /**
     * 获取增量列表
     * @param regions
     * @return
     */
    EurekaHttpResponse<Applications> getDelta(String... regions);

    /**
     * 根据vip获取列表
     * @param vipAddress
     * @param regions
     * @return
     */
    EurekaHttpResponse<Applications> getVip(String vipAddress, String... regions);

    /**
     * 根据svip获取列表
     * @param secureVipAddress
     * @param regions
     * @return
     */
    EurekaHttpResponse<Applications> getSecureVip(String secureVipAddress, String... regions);

    /**
     * 根据集群ID获取服务集群列表
     * @param appName
     * @return
     */
    EurekaHttpResponse<Application> getApplication(String appName);

    /**
     * 根据集群ID与服务ID获取服务
     * @param appName
     * @param id
     * @return
     */
    EurekaHttpResponse<InstanceInfo> getInstance(String appName, String id);

    /**
     * 根据服务ID获取服务
     * @param id
     * @return
     */
    EurekaHttpResponse<InstanceInfo> getInstance(String id);

    /**
     * 关闭
     */
    void shutdown();
}
```

### 集群接口列表
```base
package com.netflix.eureka.cluster;

import com.netflix.discovery.shared.transport.EurekaHttpClient;
import com.netflix.discovery.shared.transport.EurekaHttpResponse;
import com.netflix.eureka.cluster.protocol.ReplicationList;
import com.netflix.eureka.cluster.protocol.ReplicationListResponse;
import com.netflix.eureka.resources.ASGResource.ASGStatus;

/**
 * @author Tomasz Bak
 */
public interface HttpReplicationClient extends EurekaHttpClient {

    /**
     * 状态更新同步
     * @param asgName
     * @param newStatus
     * @return
     */
    EurekaHttpResponse<Void> statusUpdate(String asgName, ASGStatus newStatus);

    /**
     * 批量执行更新同步任务
     * @param replicationList
     * @return
     */
    EurekaHttpResponse<ReplicationListResponse> submitBatchUpdates(ReplicationList replicationList);
}
```

### 接口实现类
这里使用idea快速查看接口的实现类（快捷键：ctrl + h或ctrl + alt +B）
![/styles/images/eureka/clientApiImpl.png]({{ '/styles/images/eureka/clientApiImpl.png' | prepend: site.baseurl  }})

可看到有四个实现类：
- EurekaHttpClientDecorator为高级别的实现
- AbstractJersey2EurekaHttpClient和AbstractJerseyEurekaHttpClient为低级别的实现
- HttpReplicationClient为集群实现。

扩展：如果以上实现不能满足需求，可自定义通讯实现类，如netty、httpclient等。  


### EurekaHttpClient接口的low level实现
以AbstractJerseyEurekaHttpClient中的register为例：  
```base
/**
 * @author Tomasz Bak
 */
public abstract class AbstractJerseyEurekaHttpClient implements EurekaHttpClient {

    private static final Logger logger = LoggerFactory.getLogger(AbstractJerseyEurekaHttpClient.class);
    protected static final String HTML = "html";

    protected final Client jerseyClient;
    protected final String serviceUrl;

    protected AbstractJerseyEurekaHttpClient(Client jerseyClient, String serviceUrl) {
        this.jerseyClient = jerseyClient;
        this.serviceUrl = serviceUrl;
        logger.debug("Created client for url: {}", serviceUrl);
    }

    @Override
    public EurekaHttpResponse<Void> register(InstanceInfo info) {
        String urlPath = "apps/" + info.getAppName();
        ClientResponse response = null;
        try {
            Builder resourceBuilder = jerseyClient.resource(serviceUrl).path(urlPath).getRequestBuilder();
            addExtraHeaders(resourceBuilder);
            response = resourceBuilder
                    .header("Accept-Encoding", "gzip")
                    .type(MediaType.APPLICATION_JSON_TYPE)
                    .accept(MediaType.APPLICATION_JSON)
                    .post(ClientResponse.class, info);
            return anEurekaHttpResponse(response.getStatus()).headers(headersOf(response)).build();
        } finally {
            if (logger.isDebugEnabled()) {
                logger.debug("Jersey HTTP POST {}/{} with instance {}; statusCode={}", serviceUrl, urlPath, info.getId(),
                        response == null ? "N/A" : response.getStatus());
            }
            if (response != null) {
                response.close();
            }
        }
    }
    ...
```
AbstractJerseyEurekaHttpClient并没有参与底层的通讯，只是通过构造传入一个jerseyhttp客户端，通过jerseyhttp客户端发送请求。

### EurekaHttpClient接口的Top level实现
#### EurekaHttpClientDecorator
```base
public abstract class EurekaHttpClientDecorator implements EurekaHttpClient {

    public enum RequestType {
        Register,
        Cancel,
        SendHeartBeat,
        StatusUpdate,
        DeleteStatusOverride,
        GetApplications,
        GetDelta,
        GetVip,
        GetSecureVip,
        GetApplication,
        GetInstance,
        GetApplicationInstance
    }

    public interface RequestExecutor<R> {
        EurekaHttpResponse<R> execute(EurekaHttpClient delegate);

        RequestType getRequestType();
    }

    protected abstract <R> EurekaHttpResponse<R> execute(RequestExecutor<R> requestExecutor);

    @Override
    public EurekaHttpResponse<Void> register(final InstanceInfo info) {
    	// 创建注册请求的执行器
        return execute(new RequestExecutor<Void>() {
            @Override
            public EurekaHttpResponse<Void> execute(EurekaHttpClient delegate) {
                return delegate.register(info);
            }

            @Override
            public RequestType getRequestType() {
                return RequestType.Register;
            }
        });
    }
    ...
```
使用装饰模式，在抽象类中创建RequestExecutor执行器，子类执行execute方法时传入，子类在RequestExecutor执行前或后进行增强处理。

#### EurekaHttpClientDecorator实现

| 实现类 | 描述 |
| :-----| :---- | 
| MetricsCollectingEurekaHttpClient | 实现请求响应状态指标采集的Eureka远程通讯客户端装饰器，为EurekaHttpClient添加响应状态指标采集向功能 |
| RedirectingEurekaHttpClient | 实现请求重定向的Eureka远程通讯客户端装饰器，为EurekaHttpClient添加请求重定向功能 | 
| RetryableEurekaHttpClient | 实现请求失败重试的Eureka远程通讯客户端装饰器，为EurekaHttpClient添加失败重试功能 |
| SessionedEurekaHttpClient | 实现请求会话刷新的Eureka远程通讯客户端装饰器，强制定期(会话)完全重新连接，防止一个客户端永远坚持一个特定的Eureka服务器实例 |

## 客户端接口工厂
客户端接口通过工厂模式进行实例化，我们知道接口可分为low level和top level，那工厂也对应有low、top之分，分别创建对应级别的接口实现。

### low level工厂接口
```base
package com.netflix.discovery.shared.transport;

import com.netflix.discovery.shared.resolver.EurekaEndpoint;

/**
 * A low level client factory interface. Not advised to be used by top level consumers.
 *
 * @author David Liu
 */
public interface TransportClientFactory {

    EurekaHttpClient newClient(EurekaEndpoint serviceUrl);

    void shutdown();

}
```

### low level工厂接口实现类
![/styles/images/eureka/clientFasctoryImpl.png]({{ '/styles/images/eureka/clientFasctoryImpl.png' | prepend: site.baseurl  }})
从上图可以看出，TransportClientFactory为低级别的工厂接口，主要有jersey、jersey2工厂实现。  
所以jersey、jersey2客户端接口实现类通过这个工厂创建，所以通过jersey方式通讯为low级别接口。  

### top level工厂接口
```base
package com.netflix.discovery.shared.transport;

/**
 * A top level factory to create http clients for application/eurekaClient use
 *
 * @author Tomasz Bak
 */
public interface EurekaHttpClientFactory {

    EurekaHttpClient newClient();

    void shutdown();

}
```

### top level工厂接口实现类
![/styles/images/eureka/clientFactoryHignImpl.png]({{ '/styles/images/eureka/clientFactoryHignImpl.png' | prepend: site.baseurl  }})
top level接口类为EurekaHttpClientFactory，并没有具体的实现类，而是通过EurekaHttpClients工具类进行工厂接口实例化。

EurekaHttpClients类中实例化工厂的主要方法如下：
```base
static EurekaHttpClientFactory canonicalClientFactory(final String name,
	  final EurekaTransportConfig transportConfig,
	  final ClusterResolver<EurekaEndpoint> clusterResolver,
	  final TransportClientFactory transportClientFactory) {

    return new EurekaHttpClientFactory() {
        @Override
        public EurekaHttpClient newClient() {
            return new SessionedEurekaHttpClient(
                    name,
                    RetryableEurekaHttpClient.createFactory(
                            name,
                            transportConfig,
                            clusterResolver,
                            RedirectingEurekaHttpClient.createFactory(transportClientFactory),
                            ServerStatusEvaluators.legacyEvaluator()),
                    transportConfig.getSessionedClientReconnectIntervalSeconds() * 1000
            );
        }

        @Override
        public void shutdown() {
            wrapClosable(clusterResolver).shutdown();
        }
    };
}
```

## 总结  
- 1.工厂和接口区分low、top级别，不同级别工厂创建不同级别客户端接口实现。  
- 2.高级别的工厂没有具体实现类，通过EurekaHttpClients工具类创建，高级别客户端接口实现采用装饰模式，对原请求执行器进行新功能增强，如采集、重试、重定向等。  
- 3.Eureka接口模块设计清晰，支持自定义扩展接口实现，可采用别的通讯解决方案接入，如netty、okhttp等。  
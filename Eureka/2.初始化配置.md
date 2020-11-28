---
layout: post
#标题配置
title:  Eureka源码分析2-客户端配置初始化
#时间配置
date:   2019-11-10 11:00:00 +0800
#大类配置
categories: 服务注册
#小类配置
tag: Eureka
---

* content
{:toc}

## 说明
客户端启动时需要读取配置文件属性，用于初始化服务实例注册元数据，这一步实在eureka-client模块完成。  
## 1.创建AmazonInfo对象
```bash
/**
 + [建立一个HTTP获取注册到eureka的AWS特定信息，主要包含实例的元数据]
 + @type {[type]}
 */
AmazonInfo initialAmazonInfo = AmazonInfo.Builder.newBuilder().build();
```

## 2.创建CloudInstanceConfig对象
```bash
/**
  + 初始化注册所需信息的类
  + Eureka Server并被其他组件发现。
  *
  + < p >
  + 注册所需的信息由用户通过pass提供
  + {@link EurekaInstanceConfig}中契约定义的配置
  + }。AWS客户端可以使用或扩展{@link CloudInstanceConfig
  + }。其他非aws客户端可以使用或扩展其中之一
  + {@link MyDataCenterInstanceConfig}或非常基本
  + {@link AbstractInstanceConfig}。
  + < / p >
  + @作者Karthik Ranganathan, Greg Kim
  */
CloudInstanceConfig config = spy(new CloudInstanceConfig(initialAmazonInfo));
```

## 3.创建InstanceInfo对象
```bash
/ * *
  + 持有注册所需信息的类
  + Eureka Server并被其他组件发现。
  + < p >
  + @Auto带注释的字段按原样序列化;其他字段
  + 序列化为指定的@Serializer。
  + < / p >
  + @作者Karthik Ranganathan, Greg Kim
  + /
InstanceInfo instanceInfo = InstanceInfoGenerator.takeOne();
```

## 4.创建ApplicationInfoManager对象
```bash
/**
  + 初始化注册所需信息的类
  + Eureka Server并被其他组件发现。
  *
  + < p >
  + 注册所需的信息由用户通过pass提供
  + {@link EurekaInstanceConfig}中契约定义的配置
  + }。AWS客户端可以使用或扩展{@link CloudInstanceConfig
  + }。其他非aws客户端可以使用或扩展其中之一
  + {@link MyDataCenterInstanceConfig}或非常基本
  + {@link AbstractInstanceConfig}。
  + < / p >
  + @作者Karthik Ranganathan, Greg Kim
  */
ApplicationInfoManager applicationInfoManager = new ApplicationInfoManager(config, instanceInfo, null);
```

## 5.单元测试
```bash
import com.netflix.discovery.CommonConstants;
import com.netflix.discovery.util.InstanceInfoGenerator;
import org.junit.Before;
import org.junit.Test;
import static com.netflix.appinfo.AmazonInfo.MetaDataKey.localIpv4;
import static com.netflix.appinfo.AmazonInfo.MetaDataKey.publicHostname;
import static org.hamcrest.CoreMatchers.is;
import static org.hamcrest.CoreMatchers.not;
import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertNotEquals;
import static org.junit.Assert.assertThat;
import static org.mockito.Matchers.anyBoolean;
import static org.mockito.Mockito.spy;
import static org.mockito.Mockito.when;
/**
 + @author David Liu
 */
public class ApplicationInfoManagerTest {
    private CloudInstanceConfig config;
    private String dummyDefault = "dummyDefault";
    private InstanceInfo instanceInfo;
    private ApplicationInfoManager applicationInfoManager;
    @Before
    public void setUp() {
      // 创建Eureka实例的元数据对象
      AmazonInfo initialAmazonInfo = AmazonInfo.Builder.newBuilder().build();
      // 创建元数据扩展配置对象实例
      config = spy(new CloudInstanceConfig(initialAmazonInfo));
      // 创建持有注册所需信息的类
      instanceInfo = InstanceInfoGenerator.takeOne();
      // 创建初始化注册所需信息的类
      this.applicationInfoManager = new ApplicationInfoManager(config, instanceInfo, null);
      when(config.getDefaultAddressResolutionOrder()).thenReturn(new String[]{
              publicHostname.name(),
              localIpv4.name()
      });
      when(config.getHostName(anyBoolean())).thenReturn(dummyDefault);
    }
    @Test
    public void testRefreshDataCenterInfoWithAmazonInfo() {
      String newPublicHostname = "newValue";
      assertThat(instanceInfo.getHostName(), is(not(newPublicHostname)));
      ((AmazonInfo)config.getDataCenterInfo()).getMetadata().put(publicHostname.getName(), newPublicHostname);
      applicationInfoManager.refreshDataCenterInfoIfRequired();
      assertThat(instanceInfo.getHostName(), is(newPublicHostname));
    }
}
```
 **至此，读取配置过程就完成了！**

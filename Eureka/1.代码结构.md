---
layout: post
#标题配置
title:  Eureka源码分析1-代码结构
#时间配置
date:   2019-09-21 02:32:00 +0800
#大类配置
categories: 服务注册
#小类配置
tag: Eureka
---

* content
{:toc}


简介							{#description}
====================================
[![Build Status](https://travis-ci.org/Netflix/eureka.svg?branch=master)](https://travis-ci.org/Netflix/eureka)
+ Eureka是一个基于REST(具象状态传输)的服务，主要用于AWS云中定位服务，以实现中间层服务器的负载平衡和故障转移。
+ 在Netflix, Eureka除了在中间层负载平衡中扮演重要角色外，还用于以下目的。
+ 帮助Netflix Asgard——一个开源服务，使云部署更容易。
+ 快速回滚版本，以防出现问题，避免重新启动100个实例，这可能需要很长时间。
+ 在滚动推送中，避免在出现问题时向所有实例传播新版本。
+ 让我们的cassandra部署从通信流中取出实例进行维护。
+ 来标识环中的节点列表。
+ 用于由于各种其他原因携带其他特定于应用程序的关于服务的其他特定元数据。

编译               {#Building}
------------------------------------
构建需要java8，因为一些必需的库是java8 (servo)，但是源和目标兼容性仍然设置为1.7。

支持              {#Support}
------------------------------------
(Eureka谷歌集团)(https://groups.google.com/forum/?fromgroups#!forum/eureka_netflix)

文档              {#Documentation}
------------------------------------
详细文档请参见[wiki](https://github.com/Netflix/eureka/wiki)。

高可用
------------------------------------
![/styles/images/eureka/code-checkstyle.png]({{ '/styles/images/eureka/eureka_architecture.png' | prepend: site.baseurl  }})

项目模块							    {#module}
====================================
Eureka包含的模块比较多，如下图：
![/styles/images/eureka/module_lists.png]({{ '/styles/images/eureka/module_lists.png' | prepend: site.baseurl  }})
这...么多...模块！  
Eureka项目是使用gradle编译的，所以需要提前安装好gradle环境。  
不过现在开发工具一般都自带maven，gradle编译模块，省去不少安装配置麻烦，本文使用INTELJ IDEA。

模块解析					{#module-explain}
====================================

模块一、codequality							{#m1}
------------------------------------
![/styles/images/eureka/code-checkstyle.png]({{ '/styles/images/eureka/code-checkstyle.png' | prepend: site.baseurl  }})

这个模块只有一个名为checkstyle.xml文件，是一个检查器，主要检查代码规范，校验这些。

比如，截取以下部分代码讲解：
```bash
<module name="Checker">

    <!-- Checks that a package-info.java file exists for each package.     -->
    <!-- 检查每个java package中是否有java注释文件 -->
    <!-- See http://checkstyle.sf.net/config_javadoc.html#JavadocPackage -->
    <!--
    <module name="JavadocPackage">
      <property name="allowLegacy" value="true"/>
    </module>
    -->

    <!-- Checks whether files end with a new line.                        -->
    <!-- 检查文件是否以一个空行结束 -->
    <!-- See http://checkstyle.sf.net/config_misc.html#NewlineAtEndOfFile -->
    <module name="NewlineAtEndOfFile"/>

    <!-- Checks that property files contain the same keys.         -->
    <!-- 检查property文件中是否有相同的key -->
    <!-- See http://checkstyle.sf.net/config_misc.html#Translation -->
    <module name="Translation"/>

    <!-- Checks for Size Violations.                    -->
    <!-- 检查java文件的长度。 -->
    <!-- See http://checkstyle.sf.net/config_sizes.html -->
    <module name="FileLength"/>

    <!-- Checks for whitespace                               -->
    <!-- 检查文件中是否含有'\t' -->
    <!-- See http://checkstyle.sf.net/config_whitespace.html -->
    <module name="FileTabCharacter"/>

    <!-- Miscellaneous other checks.                   -->
    <!-- 正则表达式单行匹配 -->
    <!-- See http://checkstyle.sf.net/config_misc.html -->
    <module name="RegexpSingleline">
       <property name="format" value="\s+$"/>
       <property name="minimum" value="0"/>
       <property name="maximum" value="0"/>
       <property name="message" value="Line has trailing spaces."/>
       <property name="severity" value="info"/>
    </module>
    ...
</module>
```

模块二、eureka-client							{#m2}
------------------------------------
Eureka客户端，此模块完成发起和服务端通讯实现，包含appinfo包和discovery包。
+ **appinfo包**
  1. **客户端实例信息配置，用于注册服务的元数据类和接口。**
  2. **结构如下图：**
  ![/styles/images/eureka/eureka-client_appinfo.png]({{ '/styles/images/eureka/eureka-client_appinfo.png' | prepend: site.baseurl  }})
  目录说明： 

+ **discovery包**  
  **1. 实现服务注册与发现的功能。**
  **2. 结构如下图：**
  ![/styles/images/eureka/eureka-client_discovery.png]({{ '/styles/images/eureka/eureka-client_discovery.png' | prepend: site.baseurl  }})
  目录说明：  
    (1)、com.netflix.discovery 包：Eureka-Client 的注册与发现相关功能。    
    (2)、com.netflix.discovery.DiscoveryClient 类：客户端注册发现服务端实例实现类。 
    (3)、com.netflix.discovery.guice 包：Eureka 计划使用 Google Guice 实现依赖注入。Guice是Google开发的一个轻量级，基于Java5（主要运用泛型与注释特性）的依赖注入框架(IOC)。  
    (4)、com.netflix.discovery.converters 包：Eureka 内部传输数据编解码转换器，支持 XML / JSON 格式。  
    (5)、com.netflix.discovery.endpoint 包：目前该包正在重构，和下文的com.netflix.discovery.shared.dns 和 com.netflix.discovery.shared.resolver 用途相近。  
    (6)、com.netflix.disvoery.provider 包：目前仅有 DiscoveryJerseyProvider 类。该类声明自定义的 Jersey 请求和响应的序列化和反序列化实现。  
    (7)、com.netflix.disvoery.providers 包：目前仅有 DefaultEurekaClientConfigProvider 类。该类实现 javax.inject.Provider 接口，设置 EurekaClientConfig ( Eureka 客户端配置 ) 的生成工厂。  
    (8)、com.netflix.discovery.shared 包：Eureka-Client 和 Eureka-Server 注册发现相关的共享重用的代码。  
    下文你会看到，Eureka-Server 通过 eureka-core 模块实现，eureka-core 依赖 eureka-client。  
    粗一看，我们会感觉 What ？Eureka-Server 代码依赖 Eureka-Client 代码！？这个和 Eureka-Server 多节点注册信息 P2P 同步的实现有关。  
    一个 Eureka-Server 收到 Eureka-Client 注册请求后，Eureka-Server 会自己模拟 Eureka-Client 发送注册请求到其它的   Eureka-Server，因此部分实现代码就使用到了这个包。  
    (9)、com.netflix.discovery.shared.transport 包：Eureka-Client 对 Eureka-Server RESTful 的 HTTP 客户端，基于 Jersey Client 实现。Jersey 在下文「2.1 eureka-client-jersey2」详细解析。  
    (10)、com.netflix.discovery.shared.dns 包 ：DNS 解析器。  
    (11)、com.netflix.discovery.shared.resolver 包：Eureka Endpoint 解析器。在 《Eureka 源码解析 —— EndPoint 与 解析器》 有详细解析。  
    (12)、com.netflix.discovery.util 包 ：工具类。  


模块三、eureka-client-archaius2				{#m3}
------------------------------------
#### 特征
Archaius包含一组由Netflix使用的配置管理api。它提供了以下功能:
* 动态、类型化属性
* 高吞吐量和线程安全的配置操作
* 一个轮询框架，允许获取配置源的属性更改
* 在有效/“成功”属性突变时调用的回调机制(在有序的配置层次结构中)
一个JMX MBean，可以通过JConsole访问它来检查和调用属性上的操作
* 开箱即用，应用程序的复合配置(具有有序的层次结构)(大多数web应用程序愿意使用基于约定的属性文件位置)
* url、JDBC和Amazon DynamoDB的动态配置源的实现
* Scala动态属性包装器


#### 文档
Please see [wiki](https://github.com/Netflix/archaius/wiki) for detail documentation.  


#### 来源
这个项目的代号来自一种濒临灭绝的变色龙。我们选择了Archaius，因为变色龙以根据环境和环境改变颜色(一种属性)而闻名。这个项目产生于一个强烈的愿望，即使用动态属性更改来影响基于特定上下文的运行时行为。  


#### Eureka READEME描述  
这是一个使用Archaius 2.x作为后台配置系统版本的eureka客户端。  
注意此客户端仍在进行中，这个客户机也只兼容java8(如Archaius 2.x只兼容java8)。  


模块四、eureka-client-jersey2					{#m4}
------------------------------------
***[jersey](https://eclipse-ee4j.github.io/jersey/)***简介：  
如果没有一个好的工具包，开发能够无缝地支持以各种表示媒体类型公开数据并抽象出客户端-服务器通信的底层细节的RESTful Web服务不是一项容易的任务。  
为了简化基于rest的Web服务及其客户端在Java中的开发，设计了一个标准的、可移植的JAX-RS API。  
Jersey RESTful Web服务框架是一个开源的、高质量的、用于在Java中开发RESTful Web服务的框架，它支持JAX-RS api，并作为JAX-RS (JSR 311和JSR 339)参考实现。  


Jersey框架不仅仅是JAX-RS参考实现。  
Jersey提供了自己的API，它扩展了JAX-RS工具包，提供了附加的功能和实用程序，以进一步简化RESTful服务和客户端开发。  
Jersey还公开了许多扩展spi，以便开发人员可以扩展Jersey以满足他们的需要。


Jersey项目的目标可以概括为以下几点:
+ 跟踪JAX-RS API，并定期发布随GlassFish发布的产品质量参考实现;
+ 提供api来扩展Jersey并建立一个用户和开发人员的社区;
+ 最后利用Java和Java虚拟机轻松构建RESTful Web服务。


Jersey的最新稳定版本是2.29.1。


Eureka模块中的READEME描述  
请注意，这个jersey2兼容的Eureka客户机(Eureka -client-jersey2)是由社区创建和维护的。Netflix目前并不在内部使用这个库。  

模块五、eureka-core           {#m5}
------------------------------------
eureka-core 模块为 Eureka-Server 的功能实现：
+ com.netflix.eureka.EurekaBootStrap 类：Eureka-Server 启动类。
+ com.netflix.eureka.aws 包：与亚马逊 AWS 服务相关的类。
+ com.netflix.eureka.cluster 包：Eureka-Server 集群数据复制相关的代码。
+ com.netflix.eureka.lease 包：应用注册后的租约管理( 注册 / 取消 / 续期 / 过期 )。
+ com.netflix.eureka.resousrces 包：资源，基于 Jersey Server 实现，相当于 Spring MVC 的控制层代码。
+ com.netflix.eureka.transport 包：Eureka-Server 对 Eureka-Server 的 RESTful HTTP 客户端，基于 com.netflix.discovery.shared.transport 封装实现。
+ com.netflix.eureka.util 包：工具类。 


模块六、eureka-core-jersey2					{#m6}
------------------------------------
Eureka-Server 使用 Jersey Server 创建 RESTful Server 。
Eureka-Client 使用 Jersey Client 请求 Eureka-Server 。   

Eureka模块中的READEME描述  
请注意，这个jersey2兼容的Eureka core (Eureka -core-jersey2)是由社区创建和维护的。Netflix目前并不在内部使用这个库。

jersey参考模块四

模块七、eureka-examples           {#m7}
------------------------------------
提供 Eureka-Client 使用例子。  


模块八、eureka-resources						{#m8}
------------------------------------
管理后台界面，就是查询服务实例信息，服务器运行状态界面，使用jsp实现；  


模块九、eureka-server						{#m9}
------------------------------------
集中打包，将eureka-client， eureka-core， eureka-resources三个模块打包成 Eureka-Server的 war包。

模块十、eureka-server-governator						{#m10}
------------------------------------
使用 Netflix Governator 管理 Eureka-Server 的生命周期。  


模块十一、eureka-test-utils				{#m11}
------------------------------------
提供 Eureka 单元测试工具类。  


总结							    {#result}
====================================

总结

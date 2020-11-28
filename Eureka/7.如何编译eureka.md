---
layout: post
#标题配置
title:  Eureka源码分析7-使用IDEA编译部署Eureka源码
#时间配置
date:   2020-02-06 11:04:00 +0800
#大类配置
categories: 服务注册
#小类配置
tag: Eureka
---

* content
{:toc}

## 说明
Netflix开源的Eureka是使用Gradle构建的，因此这里使用Gradle来编译它

## 环境准备
IntelliJ IDEA，git，gradle，tomcat

### IntelliJ IDEA
下载官网地址 https://www.jetbrains.com/idea/download/  
本人使用的版本 2019.3.2，具体安装破解参考网上

### git
官网下载 https://git-scm.com/downloads  
idea中配置 Settings --> Git --> Path to Git executable  指定git.exe安装路径

### gradle
官网下载 https://services.gradle.org/distributions/  
这里使用idea自带的gradle，gradle版本必须小于4.0，否则会出现不能依赖nebula.netflixoss、jetty插件问题  
修改gradle->wrapper->gradle-wrapper.properties 指定gradle版本，这里使用3.5版本，修改如下  
```bash
#Wed Feb 05 19:29:07 CST 2020
#distributionUrl=https\://services.gradle.org/distributions/gradle-4.8.1-all.zip
#使用3.5版本
distributionUrl=https\://services.gradle.org/distributions/gradle-3.5-all.zip
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStorePath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
```

配置gradle下载源为国内阿里云  
在项目中的build.gradle修改内容
```bash
buildscript {
    //repositories { jcenter() }

    // 优先使用国内镜像
    repositories {
        maven { url 'http://maven.aliyun.com/nexus/content/groups/public/'}
        maven { url 'http://maven.aliyun.com/nexus/content/repositories/jcenter'}
    }

    dependencies {
        classpath 'com.netflix.nebula:gradle-extra-configurations-plugin:2.2.+'
    }
}

allprojects {
    ext {
        githubProjectName = 'eureka'

        awsVersion = '1.11.277'
        servletVersion = '2.5'
        jerseyVersion = '1.19.1'
        jettisonVersion = '1.3.7'
        apacheHttpClientVersion = '4.5.3'
        guiceVersion = '4.1.0'
        servoVersion = '0.12.21'
        governatorVersion = '1.17.5'
        archaiusVersion = '0.7.6'
        jacksonVersion = '2.9.4'
        woodstoxVersion = '5.2.1'

        // test deps
        jetty_version = '7.2.0.v20101020'
        junit_version = '4.11'
        mockitoVersion = '1.10.19'
        mockserverVersion = '3.9.2'
    }

    // 优先使用国内镜像
    repositories {
        maven { url 'http://maven.aliyun.com/nexus/content/groups/public/'}
        maven { url 'http://maven.aliyun.com/nexus/content/repositories/jcenter'}
    }
}
```

idea中gradle配置：Setting->Build,Execution,Deployment->Build Tools->Gradle  
Gradle user home  指定安装目录(选择一个目录即可)  
Use Gradle from 指定Gradle源为指定Gradle源 

![/styles/images/eureka/gradle_build.png]({{ '/styles/images/eureka/gradle_build.png' | prepend: site.baseurl  }})

### tomcat
官网下载 http://tomcat.apache.org/
版本 这里使用8.5

## idea构建部署
### idea检出eureka源码
eureka github开源地址：https://github.com/Netflix/eureka.git
![/styles/images/eureka/eureka_git_check.png]({{ '/styles/images/eureka/eureka_git_check.png' | prepend: site.baseurl  }})


### 检出完成后开始构建  
![/styles/images/eureka/eureka_build.png]({{ '/styles/images/eureka/eureka_build.png' | prepend: site.baseurl  }})
看到控制台输出CONFIGURE SUCCESSFUL，即成功。

### 编译打包war文件
![/styles/images/eureka/eureka_war_package.png]({{ '/styles/images/eureka/eureka_war_package.png' | prepend: site.baseurl  }})
控制台看到输出BUILD SUCCESSFUL，即打包war成功，在build-->lib可以看到war包

但是这样打出来的war包是没有包含Eureka管理界面的，修改eureka-server-->build.gradle文件，加入eureka-resources模块
![/styles/images/eureka/eureka_war_package2.png]({{ '/styles/images/eureka/eureka_war_package2.png' | prepend: site.baseurl  }})
重新执行编译打包操作

## 部署Eureka到Tomcat  
### 部署目录
将前面打包的 eureka-server-1.9.19-SNAPSHOT.war 拷贝到 tomcat webapps
![/styles/images/eureka/tomcat_depy.png]({{ '/styles/images/eureka/tomcat_depy.png' | prepend: site.baseurl  }})

### 启动tomcat  
![/styles/images/eureka/tomcat_start.png]({{ '/styles/images/eureka/tomcat_start.png' | prepend: site.baseurl  }})
启动过程会定时任务去发现其他服务，所以会出现Cannot execute request on any known server错误，不用管它

### 服务管理界面
访问地址：http://127.0.0.1:8080/eureka-server-1.9.19-SNAPSHOT/  
![/styles/images/eureka/eureka_manage_page.png]({{ '/styles/images/eureka/eureka_manage_page.png' | prepend: site.baseurl  }})
至此，eureka服务部署完成！
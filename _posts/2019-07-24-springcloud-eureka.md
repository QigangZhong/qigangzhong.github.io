---
layout: post
title:  "springcloud-eureka(Finchley版)"
categories: springcloud
tags:  微服务 springcloud
author: 网络
---

* content
{:toc}









## 单节点注册中心

[示例代码](https://gitee.com/qigangzhong/springcloud.f/tree/master/springcloud.f.eureka)

### 服务端-添加maven依赖

1.项目根pom中添加对springcloud及springboot的maven依赖

```xml
<properties>
    <springcloud.version>Finchley.RELEASE</springcloud.version>
    <springboot.version>2.0.3.RELEASE</springboot.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${springcloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-parent</artifactId>
            <version>${springboot.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

关于springboot及springcloud的版本对应，可以参考[springcloud官网](https://spring.io/projects/spring-cloud)的说明：

| Release Train | Boot Version |
| ------------- | ------------ |
| Greenwich     | 2.1.x        |
| Finchley      | 2.0.x        |
| Edgware       | 1.5.x        |
| Dalston       | 1.5.x        |
{: .table.table-bordered }

具体的版本号参考maven仓库：

* [spring-cloud-dependencies maven依赖](https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-dependencies)

* [spring-boot-starter-parent maven依赖](https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-parent)

2.在项目子pom中添加eureka依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
</dependencies>
```

### 服务端-入口类添加注解

添加`@EnableEurekaServer`注解启用服务端

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class,args);
    }
}
```

### 服务端-配置

```bash
spring.application.name=single-eureka-server
server.port=8761

# springcloud集群中其他机器可以通过hostname加端口访问到对方
eureka.instance.hostname=localhost
# 是否注册到注册中心，这个应用本身就是eureka-server注册中心（默认也是eureka-client），设置为false代表在注册中心不注册自己
eureka.client.register-with-eureka=false
# 是否获取注册中心的信息，这个应用本身就是注册中心，不需要获取注册中心信息
eureka.client.fetch-registry=false
eureka.client.service-url.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka/
```

各项配置的意思可以查找[springcloud官方说明](https://cloud.spring.io/spring-cloud-static/Finchley.RELEASE/single/spring-cloud.html)来了解

### 客户端

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
@EnableDiscoveryClient
public class EurekaClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaClientApplication.class,args);
    }
}
```

```bash
spring.application.name=single-eureka-client
server.port=8861

eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
```

## 注册中心集群搭建

[示例代码](https://gitee.com/qigangzhong/springcloud.f/tree/master/springcloud.f.eureka)

eureka-server集群搭建的思路就是，将每个eureka-server节点注册到集群的其它节点上去，这样客户端注册到任意一个eureka-server上之后，注册信息会在注册中心集群中同步，与单节点相比，仅仅是配置需要修改，以下示例一个3节点的eureka-server集群，每个节点的代码及pom引用同上面的单节点一致，配置如下：

```bash
spring.application.name=cluster-eureka-server1
server.port=8761

# springcloud集群中其他机器可以通过hostname加端口访问到对方
eureka.instance.hostname=eureka-server1
# 是否注册到注册中心，这里配置true来注册到集群的其它eureka-server节点上
eureka.client.register-with-eureka=true
# 是否获取注册中心的信息
eureka.client.fetch-registry=true
# 配置eureka-server其它节点的地址，将此节点注册到集群中其它几个节点上
eureka.client.service-url.defaultZone=http://localhost:8762/eureka/,http://localhost:8763/eureka/
# 默认情况下，eureka间隔60s将服务清单中没有续约的服务剔除（默认90s内没有续约），本地测试，关闭保护机制，直接让它实时剔除
eureka.server.enable-self-preservation=false
```

```bash
spring.application.name=cluster-eureka-server2
server.port=8762

# springcloud集群中其他机器可以通过hostname加端口访问到对方
eureka.instance.hostname=eureka-server2
# 是否注册到注册中心，这里配置true来注册到集群的其它eureka-server节点上
eureka.client.register-with-eureka=true
# 是否获取注册中心的信息
eureka.client.fetch-registry=true
# 配置eureka-server其它节点的地址，将此节点注册到集群中其它几个节点上
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/,http://localhost:8763/eureka/
# 默认情况下，eureka间隔60s将服务清单中没有续约的服务剔除（默认90s内没有续约），本地测试，关闭保护机制，直接让它实时剔除
eureka.server.enable-self-preservation=false
```

```bash
spring.application.name=cluster-eureka-server3
server.port=8763

# springcloud集群中其他机器可以通过hostname加端口访问到对方
eureka.instance.hostname=eureka-server3
# 是否注册到注册中心，这里配置true来注册到集群的其它eureka-server节点上
eureka.client.register-with-eureka=true
# 是否获取注册中心的信息
eureka.client.fetch-registry=true
# 配置eureka-server其它节点的地址，将此节点注册到集群中其它几个节点上
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/,http://localhost:8762/eureka/
# 默认情况下，eureka间隔60s将服务清单中没有续约的服务剔除（默认90s内没有续约），本地测试，关闭保护机制，直接让它实时剔除
eureka.server.enable-self-preservation=false
```

客户端的代码同样和单节点示例代码一样，配置中的注册中心地址需要将eureka-server集群的3个节点全部配上：

```bash
spring.application.name=cluster-eureka-client1
server.port=8861
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/,http://localhost:8762/eureka/,http://localhost:8763/eureka/
```

### 集群原理

![eureka集群原理.png](/images/springcloud/eureka集群原理.png)

* eureka客户端与eureka服务器默认保持30s的心跳，重新续约服务

#### 集群保护机制

Eureka Server 在运行期间会去统计心跳失败比例在 15 分钟之内是否低于 85%，如果低于 85%，Eureka Server 即会进入自我保护机制。

Eureka Server 进入自我保护机制，会出现以下几种情况：

1. Eureka 不再从注册列表中移除因为长时间没收到心跳而应该过期的服务
2. Eureka 仍然能够接受新服务的注册和查询请求，但是不会被同步到其它节点上(即保证当前节点依然可用)
3. 当网络稳定时，当前实例新的注册信息会被同步到其它节点中

### 集群二级缓存机制

![eureka集群缓存机制.png](/images/springcloud/eureka集群缓存机制.png)

* 为什么要有二级缓存？为什么不直接用ConcurrentHashMap？

如果集群数量非常大，直接用ConcurrentHashMap肯定性能受到影响，通过缓存的方式来解决这个问题，缓存使用读写分离机制，性能更佳。

## 问题

1. 心跳机制是如何实现的？

## 开源注册中心对比

[服务的注册与发现（Consul、zookeeper、etcd、eureka、Nacos）](https://xinyuan.blog.csdn.net/article/details/102608442)

[Nacos整体概况](https://www.jianshu.com/p/3f4f6554b67c)

## 参考

[***拜托！面试请不要再问我Spring Cloud底层原理！](https://blog.csdn.net/qq_42046105/article/details/83793787)

[springcloud官网](https://spring.io/projects/spring-cloud)

[A-G各个版本的release notes](https://github.com/spring-projects/spring-cloud/wiki)

[Finchley.RELEASE documentation](https://cloud.spring.io/spring-cloud-static/Finchley.RELEASE/single/spring-cloud.html)

[服务的注册与发现Eureka(Finchley版本)](https://blog.csdn.net/forezp/article/details/81040925)

[搭建Spring Cloud Eureka 三个节点高可用集群](https://blog.csdn.net/huanxianglove/article/details/80690130)

[Spring Cloud底层原理](http://www.imooc.com/article/283843)

[Eureka与ZooKeeper对比，Eureka的优势](https://blog.csdn.net/vtopqx/article/details/81537004)

[Eureka源码解析(三)—服务续约(心跳)](https://blog.csdn.net/yuu1009/article/details/80778928?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task)

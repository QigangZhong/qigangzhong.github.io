---
layout: post
title:  "springcloud-hystrix(Finchley版)"
categories: springcloud
tags:  微服务 springcloud
author: 网络
---

* content
{:toc}









## 介绍

断路器的功能可以有效防止由于某个服务超时或者停止服务导致依赖服务的连锁雪崩效应。

使用feign进行服务通信的时候，默认集成了hystrix断路器功能，但是默认是关闭的，需要在服务调用方配置中显示开启：

```bash
# 打开feign默认集成的hystrix断路器功能
feign.hystrix.enabled=true
```

如果使用ribbon+restTemplate的方式，需要手动添加pom依赖，入口类添加`@EnableHystrix`注解，需要断路器功能的方法上面添加`@HystrixCommand(fallbackMethod = "demoMethod")`注解。

[示例代码](https://gitee.com/qigangzhong/springcloud.f/tree/master/springcloud.f.hystrix)

hystrix主要提供以下几个功能：

* 对依赖服务调用时出现的调用延迟和调用失败进行控制和容错保护
* 在复杂的分布式系统中，阻止某一个依赖服务的故障在整个系统中蔓延，服务A->服务B->服务C，服务C故障了，服务B也故障了，服务A故障了，整套分布式系统全部故障，整体宕机
* 提供fail-fast（快速失败）和快速恢复的支持
* 提供fallback优雅降级的支持
* 支持近实时的监控、报警以及运维操作

## feign+hystrix进行客户端熔断

### 服务端

就是一个普通的服务，仅依赖springboot+eureka，服务接口我们模拟一下超时情况

```java
@RestController
public class HiController {
    @Value("${server.port}")
    String port;

    @GetMapping("/hi")
    public String sayHi(@RequestParam("name") String name) throws InterruptedException {

        //模拟超时
        Random random = new Random();
        int r = random.nextInt(5);
        System.out.println(r);
        TimeUnit.SECONDS.sleep(r);

        return String.format("Hi %s，i'm from port %s",name,port);
    }
}
```

### hystrix客户端

pom依赖与普通的feign客户端依赖一样，默认包含了hystrix功能

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
</dependencies>
```

入口函数没什么需要特殊修改的

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class ServiceHiClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceHiClientApplication.class,args);
    }
}
```

定义的feign服务代理需要修改，指定熔断措施fallback

```java
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
@FeignClient(value = "service-hi-hystrix", fallback = ServiceHiHystrix.class)
public interface ServiceHi {
    @GetMapping("/hi")
    String callHiService(@RequestParam("name") String name);
}


import org.springframework.stereotype.Component;
@Component
public class ServiceHiHystrix implements ServiceHi {
    @Override
    public String callHiService(String name) {
        return "sorry "+ name+", something is wrong, that's why you see this page";
    }
}
```

## hystrix-dashboard监控

上面的示例是在feign客户端使用hystrix断路器，hystrix-dashboard可以配置在服务端，通过dashboard可以观察到改服务的流量访问情况。hystrix-dashboard相当于是hystrix断路数据的消费方。

[示例代码](https://gitee.com/qigangzhong/springcloud.f/tree/master/springcloud.f.hystrix/springcloud.f.hystrix.dashboard.service-hi)

### 改造微服务服务端，添加hystrix服务端熔断支持以及hystrix-dashboard

微服务服务pom依赖中除去springboot，eureka依赖还需要加上hystrix-dashboard相关的3个依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!--hystrix-dashboard相关依赖-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
    </dependency>
</dependencies>
```

入口类添加`@EnableHystrix`，`@EnableHystrixDashboard`注解

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;
import org.springframework.cloud.netflix.hystrix.dashboard.EnableHystrixDashboard;

@SpringBootApplication
@EnableDiscoveryClient
@EnableHystrix
@EnableHystrixDashboard
public class HystrixDashboardServiceHiApplication {
    public static void main(String[] args) {
        SpringApplication.run(HystrixDashboardServiceHiApplication.class,args);
    }
}
```

需要进行服务端熔断机制的接口上添加`@HystrixCommand(fallbackMethod = "xxx")`注解

```java
import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.Random;
import java.util.concurrent.TimeUnit;

@RestController
public class HiController {
    @Value("${server.port}")
    String port;

    @GetMapping("/hi")
    @HystrixCommand(fallbackMethod = "sayHiError")
    public String sayHi(@RequestParam("name") String name) throws InterruptedException {

        //模拟随机超时
        /*Random random = new Random();
        int r = random.nextInt(5);
        System.out.println(r);
        TimeUnit.SECONDS.sleep(r);*/

        return String.format("Hi %s，i'm from port %s",name,port);
    }

    public String sayHiError(String name){
        return String.format("Oops! %s, something goes wrong! I'm from port %s",name,port);
    }
}
```

同时，配置文件中需要暴露hystrix-dashboard需要使用到的url端点（默认是关闭的）

```bash
spring.application.name=service-hi-hystrix-dashboard
server.port=8861
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
# 每间隔2s，向服务端发送一次心跳，证明自己依然“活着”
eureka.instance.lease-renewal-interval-in-seconds=2
# 告诉服务端，如果我5s之内没有给你发心跳，就代表我“死”了，将我剔除掉
eureka.instance.lease-expiration-duration-in-seconds=5

# 放开所有的web端点，包括/actuator/hystrix.stream, /actuator/health, /actuator/info
management.endpoints.web.exposure.include=*
management.endpoints.web.cors.allowed-origins=*
management.endpoints.web.cors.allowed-methods=*
```

### 测试hystrix-dashboard

1.启动eureka注册中心springcloud.f.eureka.server

2.启动微服务springcloud.f.hystrix.dashboard.service-hi

3.访问hystrix-dashboard的web界面，输入本服务的hystrix.stream接口地址

hystrix-dashboard的web界面：http://localhost:8861/hystrix

hystrix.stream接口地址：http://localhost:8861/actuator/hystrix.stream

访问添加`@HystrixCommand`注解的api接口，可以看到图形界面上的流量信息

http://localhost:8861/hi?name=zhangsan

## hystrix-dashboard turbine聚合监控

每个微服务都集成hystrix-dashboard之后，如何统一查看所有服务的监控数据呢？turbine就是对所有服务的hystrix-dashboard监控数据进行整合。

为了查看多个微服务，再创建一个独立的微服务并集成hystrix-dashboard，这样就有2个服务了

* springcloud.f.hystrix.dashboard.service-hi
* springcloud.f.hystrix.dashboard.service-hello

### 创建独立turbine站点，用来聚合所有微服务的hystrix-dashboard监控数据

[示例代码](https://gitee.com/qigangzhong/springcloud.f/tree/master/springcloud.f.hystrix/springcloud.f.hystrix.dashboard.service-turbine)

pom依赖如下

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!--hystrix-dashboard相关依赖-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
    </dependency>

    <!--turbine相关依赖-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-turbine</artifactId>
    </dependency>
</dependencies>
```

入口类添加相关注解

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;
import org.springframework.cloud.netflix.hystrix.dashboard.EnableHystrixDashboard;
import org.springframework.cloud.netflix.turbine.EnableTurbine;

@SpringBootApplication
@EnableDiscoveryClient
@EnableHystrix
@EnableHystrixDashboard
@EnableTurbine
public class TurbineApplication {
    public static void main(String[] args) {
        SpringApplication.run(TurbineApplication.class,args);
    }
}
```

再添加配置文件

```bash
spring.application.name=service-turbine
server.port=8866
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
# 每间隔2s，向服务端发送一次心跳，证明自己依然“活着”
eureka.instance.lease-renewal-interval-in-seconds=2
# 告诉服务端，如果我5s之内没有给你发心跳，就代表我“死”了，将我剔除掉
eureka.instance.lease-expiration-duration-in-seconds=5

# 放开所有的web端点，包括/actuator/hystrix.stream, /actuator/health, /actuator/info
management.endpoints.web.exposure.include=*
management.endpoints.web.cors.allowed-origins=*
management.endpoints.web.cors.allowed-methods=*

# turbine相关配置
# 配置需要聚合的服务service-id列表
turbine.app-config=service-hi-hystrix-dashboard,service-hello-hystrix-dashboard
turbine.aggregator.cluster-config=default
turbine.cluster-name-expression=new String("default")
turbine.combine-host-port=true
turbine.instanceUrlSuffix.default=actuator/hystrix.stream
```

这个服务仅仅是用来聚合其它微服务的hystrix-dashboard监控数据，不需要其它业务逻辑代码

### 测试turbine

1.启动eureka注册中心springcloud.f.eureka.server

2.启动微服务springcloud.f.hystrix.dashboard.service-hi、springcloud.f.hystrix.dashboard.service-hello

3.启动turbine站点springcloud.f.hystrix.dashboard.service-turbine

4.开启turbine站点的hystrix-dashboard的web界面，添加turbine.stream接口的监控

hystrix-dashboard的web界面：http://localhost:8866/hystrix

turbine.stream接口地址：http://localhost:8866/actuator/turbine.stream

访问两个微服务的接口，可以看到turbine站点hystrix-dashboard图形界面上出现了两个微服务的监控信息

http://localhost:8861/hi?name=zhangsan

http://localhost:8862/hello?name=lisi

## hystrix原理

### 隔离策略(线程池隔离vs信号量隔离)

[Hystrix 服务的隔离策略对比，信号量与线程池隔离的差异](https://my.oschina.net/u/867417/blog/2120713)

| 隔离方式 | 是否支持超时                                           | 是否支持熔断                                              | 隔离原理         | 是否是异步调用                  | 资源消耗                                 |
| ---------- | ------------------------------------------------------------ | --------------------------------------------------------------- | -------------------- | -------------------------------------- | -------------------------------------------- |
| 线程池隔离 | 支持，可直接返回                                     | 支持，当线程池到达maxSize后，再请求会触发fallback接口进行熔断 | 每个服务单独用线程池 | 可以是异步，也可以是同步。看调用的方法 | 大，大量线程的上下文切换，容易造成机器负载高 |
| 信号量隔离 | 不支持，如果阻塞，只能通过调用协议（如：socket超时才能返回） | 支持，当信号量达到maxConcurrentRequests后。再请求会触发fallback | 通过信号量的计数器 | 同步调用，不支持异步         | 小，只是个计数器                     |
{: .table.table-bordered }

[Hystrix的线程池隔离和信号量隔离](https://blog.csdn.net/javaer_lee/article/details/87942816)

#### 线程池隔离

通过线程池的可以控制请求的数量，并且可以控制超时时间（Future.get(timeout)方法？），超过数量或者时间都可以选择抛弃请求或者执行fallback请求

```java
public MyHystrixCommand(String name) {
    super(HystrixCommand.Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("MyGroup"))
            .andCommandPropertiesDefaults(HystrixCommandProperties.Setter()
                    .withExecutionIsolationStrategy(HystrixCommandProperties.ExecutionIsolationStrategy.THREAD))
            .andThreadPoolPropertiesDefaults(HystrixThreadPoolProperties.Setter().withCoreSize(10)
                    .withMaxQueueSize(100).withMaximumSize(100)));
    this.name = name;
}
```

#### 信号量隔离

通过信号量隔离只能控制请求的数量，不支持超时

```java
super(HystrixCommand.Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey("MyGroup"))
        .andCommandPropertiesDefaults(HystrixCommandProperties.Setter()
            .withExecutionIsolationStrategy(HystrixCommandProperties.ExecutionIsolationStrategy.SEMAPHORE

            )));
    this.name = name;
}
```

### 熔断器

熔断器Circuit Breaker是位于线程池之前的组件。用户请求某一服务之后，Hystrix会先经过熔断器，此时如果熔断器的状态是打开（跳起），则说明已经熔断，这时将直接进行降级处理，不会继续将请求发到线程池。熔断器相当于在线程池之前的一层屏障。每个熔断器默认维护10个bucket ，每秒创建一个bucket ，每个blucket记录成功,失败,超时,拒绝的次数。当有新的bucket被创建时，最旧的bucket会被抛弃。

* Closed：熔断器关闭状态，调用失败次数积累，到了阈值（或一定比例）则启动熔断机制；
* Open：熔断器打开状态，此时对下游的调用都内部直接返回错误，不走网络，但设计了一个时钟选项，默认的时钟达到了一定时间（这个时间一般设置成平均故障处理时间，也就是MTTR），到了这个时间，进入半熔断状态；
* Half-Open：半熔断状态，允许定量的服务请求，如果调用都成功（或一定比例）则认为恢复了，关闭熔断器，否则认为还没好，又回到熔断器打开状态；

## hystrix监控数据持久化

hystrix+InfluxDB+grafana

参考：[采集Hystrix线程池指标并使用influxDB+Grafana实时监控（HystrixDashboard升级方案）](https://blog.csdn.net/wk52525/article/details/90550171)

## 类似hystrix的产品

[alibaba Sentinel](https://github.com/alibaba/Sentinel/wiki/%E4%BB%8B%E7%BB%8D)，也可以支持springcloud

## 参考

[springcloud官网](https://spring.io/projects/spring-cloud)

[A-G各个版本的release notes](https://github.com/spring-projects/spring-cloud/wiki)

[Finchley.RELEASE documentation](https://cloud.spring.io/spring-cloud-static/Finchley.RELEASE/single/spring-cloud.html)

[Hystrix官方说明-Metrics and Monitoring](https://github.com/Netflix/Hystrix/wiki/Metrics-and-Monitoring)

[断路器（Hystrix）(Finchley版本)](https://blog.csdn.net/forezp/article/details/81040990)

[断路器监控(Hystrix Dashboard)(Finchley版本)](https://blog.csdn.net/forezp/article/details/81041113)

[Turbine(Finchley版本)](https://www.fangzhipeng.com/springcloud/2018/08/13/sc-f13-turbine.html)

[fallback & fallbackFactory](https://www.jianshu.com/p/3d804e99e613)

[Spring Cloud底层原理](http://www.imooc.com/article/283843)

[【一起学源码-微服务】Hystrix 源码](https://www.cnblogs.com/wang-meng/p/12195475.html)

[白话：服务降级与熔断的区别](https://segmentfault.com/a/1190000012137439)

[Spring Cloud构建微服务架构：服务容错保护（Hystrix断路器）【Dalston版】](http://blog.didispace.com/spring-cloud-starter-dalston-4-3/)

[熔断机制HYSTRIX](https://www.cnblogs.com/yawen/p/6655352.html)

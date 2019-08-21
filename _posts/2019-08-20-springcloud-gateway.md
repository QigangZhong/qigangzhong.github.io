---
layout: post
title:  "springcloud-gateway(Finchley版)"
categories: springcloud
tags:  微服务 springcloud
author: 网络
---

* content
{:toc}









## 介绍

[Spring Cloud Gateway](https://spring.io/projects/spring-cloud-gateway)是Finchley版本推出的新组件，使用非阻塞api，支持websockets，提供了安全、监控/指标、限流等功能，而[Zuul](https://github.com/Netflix/zuul)基于servlet使用阻塞api，不支持websockets，限流等功能需要自己扩展。总体比较下来，SCG的功能远比Zuul强大，而且[性能也要好](https://github.com/spencergibb/spring-cloud-gateway-bench)。

> Spring Cloud Gateway features:
>
> * Built on Spring Framework 5, Project Reactor and Spring Boot 2.0
> * Able to match routes on any request attribute.
> * Predicates and filters are specific to routes.
> * Hystrix Circuit Breaker integration.
> * Spring Cloud DiscoveryClient integration
> * Easy to write Predicates and Filters
> * Request Rate Limiting
> * Path Rewriting

* Route（路由）：这是网关的基本构建块。它由一个 ID，一个目标 URI，一组断言和一组过滤器定义。如果断言为真，则路由匹配。
* Predicate（断言）：这是一个 Java 8 的 Predicate。输入类型是一个 ServerWebExchange。我们可以使用它来匹配来自 HTTP 请求的任何内容，例如 headers 或参数。SCG默认提供了下面这些[内置的断言工厂](https://cloud.spring.io/spring-cloud-gateway/reference/html/#gateway-request-predicates-factories)，可以直接通过配置文件或者代码的方式利用这些断言工厂来判断一个请求是否匹配路由规则。

![spring-cloud-gateway_predicate.png](/images/springcloud/spring-cloud-gateway_predicate.png)

* Filter（过滤器）：这是org.springframework.cloud.gateway.filter.GatewayFilter的实例，我们可以使用它修改请求和响应。同样，SCG提供了很多[内置的过滤器工厂](https://cloud.spring.io/spring-cloud-gateway/reference/html/#_gatewayfilter_factories)，可以通过配置文件或代码的方式利用这些内置的过滤器工厂来实现路由过滤功能，其中一些过滤器可以用来实现流控、熔断降级等功能。

  * [GatewayFilter](https://cloud.spring.io/spring-cloud-gateway/reference/html/#_gatewayfilter_factories) 针对特定路由的filter
  * [GlobalFilter](https://cloud.spring.io/spring-cloud-gateway/reference/html/#_global_filters) 是特殊的filter，可以作用于所有的路由

## predicate、filter基本用法

[示例代码](https://gitee.com/qigangzhong/springcloud.f/tree/master/springcloud.f.gateway)

### SCG服务器项目搭建

pom文件添加服务发现、SCG、actuator依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>

    <!--spring-cloud-gateway基于netty+webflux不需要springboot-web支持-->
    <!--<dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>-->

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>

    <!--actuator依赖-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

入口类添加服务发现注解

```java
@SpringBootApplication
@EnableDiscoveryClient
public class SpringCloudGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringCloudGatewayApplication.class,args);
    }
}
```

properties文件的配置方式如下，如果用yml文件，格式转换一下就可以了，参考[spring-cloud-gateway-documentation](https://cloud.spring.io/spring-cloud-gateway/reference/html/)配置

```bash
spring.application.name=scg-server
server.port=8900
# eureka地址配置，如果eureka是个集群，那么逗号隔开配置所有地址
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
# 每间隔2s，向eureka服务端发送一次心跳，证明自己依然“活着”
eureka.instance.lease-renewal-interval-in-seconds=2
# 告诉服务端，如果我5s之内没有给你发心跳，就代表我“死”了，将我剔除掉
eureka.instance.lease-expiration-duration-in-seconds=5


# 打印debug日志，方便调试跟踪问题
logging.level.org.springframework.cloud.gateway=debug

# 启用基于服务发现的路由定位
spring.cloud.gateway.discovery.locator.enabled=true
# 启用服务实例id名称小写支持
spring.cloud.gateway.discovery.locator.lowerCaseServiceId=true

# 启用gateway这个端点
management.endpoint.gateway.enabled=true
management.endpoints.web.exposure.include=gateway

## 1. 通过url path来路由
#spring.cloud.gateway.routes[0].id=test_route
#spring.cloud.gateway.routes[0].uri=https://www.baidu.com/
#spring.cloud.gateway.routes[0].predicates[0]=Path=/test

# 1.1 通过url path来路由到指定的后端服务（负载均衡）
spring.cloud.gateway.routes[0].id=service_a_route
spring.cloud.gateway.routes[0].uri=lb://service-a
spring.cloud.gateway.routes[0].filters[0]=StripPrefix=1
spring.cloud.gateway.routes[0].predicates[0]=Path=/a/**

## 2. 在某个时间点【之前、之后、之间】的路由地址，所有url都路由到这个地址下面
#spring.cloud.gateway.routes[0].id=baidu_route
#spring.cloud.gateway.routes[0].uri=https://www.baidu.com/
##spring.cloud.gateway.routes[0].predicates[0]=Before=2019-08-20T14:40:00+08:00[Asia/Shanghai]
##spring.cloud.gateway.routes[0].predicates[0]=After=2019-08-20T14:40:00+08:00[Asia/Shanghai]
#spring.cloud.gateway.routes[0].predicates[0]=Between=2019-08-20T14:00:00+08:00[Asia/Shanghai], 2019-08-20T14:45:00+08:00[Asia/Shanghai]

## 3. 通过cookie来路由
## curl http://localhost:8900 --cookie "testKey=testValue"
#spring.cloud.gateway.routes[0].id=test_route
#spring.cloud.gateway.routes[0].uri=https://www.baidu.com/
#spring.cloud.gateway.routes[0].predicates[0]=Cookie=testKey, testValue

## 4. 通过header来路由
## curl http://localhost:8900 -H "X-Request-Id:666666"
#spring.cloud.gateway.routes[0].id=test_route
#spring.cloud.gateway.routes[0].uri=https://www.baidu.com/
#spring.cloud.gateway.routes[0].predicates[0]=Header=X-Request-Id, \\d+

## 5. 通过host来路由
## curl http://localhost:8900 -H "Host: abc.test.com"
#spring.cloud.gateway.routes[0].id=test_route
#spring.cloud.gateway.routes[0].uri=https://www.baidu.com/
#spring.cloud.gateway.routes[0].predicates[0]=Host=**.test.com

## 5. 通过request method来路由
## curl -X GET http://localhost:8900
#spring.cloud.gateway.routes[0].id=test_route
#spring.cloud.gateway.routes[0].uri=https://www.baidu.com/
#spring.cloud.gateway.routes[0].predicates[0]=Method=GET

## 6. 通过请求参数query string来路由
## curl -X GET http://localhost:8900?testParam=testValue111
#spring.cloud.gateway.routes[0].id=test_route
#spring.cloud.gateway.routes[0].uri=https://www.baidu.com/
##spring.cloud.gateway.routes[0].predicates[0]=Query=testParam
#spring.cloud.gateway.routes[0].predicates[0]=Query=testParam, testValue\\d

## 7. 通过请求IP来路由
## curl -X GET http://localhost:8900
#spring.cloud.gateway.routes[0].id=test_route
#spring.cloud.gateway.routes[0].uri=https://www.baidu.com/
#spring.cloud.gateway.routes[0].predicates[0]=RemoteAddr=172.17.65.0/24

## 8. 组合使用predicate
#spring.cloud.gateway.routes[0].id=test_route
#spring.cloud.gateway.routes[0].uri=https://www.baidu.com/
#spring.cloud.gateway.routes[0].predicates[0]=Path=/test
#spring.cloud.gateway.routes[0].predicates[1]=Method=GET
#spring.cloud.gateway.routes[0].predicates[2]=Query=testParam

## 9. 添加过滤器
#spring.cloud.gateway.routes[0].id=add_request_param_route
#spring.cloud.gateway.routes[0].uri=http://localhost:8001/hi
#spring.cloud.gateway.routes[0].predicates[0]=Path=/test
#spring.cloud.gateway.routes[0].filters[0]=AddRequestParameter=name, zhangsan

## 10. 添加过滤器，支持负载均衡调用服务
#spring.cloud.gateway.routes[0].id=add_request_param_route
#spring.cloud.gateway.routes[0].uri=lb://service-a
#spring.cloud.gateway.routes[0].predicates[0]=Path=/hi
#spring.cloud.gateway.routes[0].filters[0]=AddRequestParameter=name, zhangsan
```

除去使用配置文件的方式来配置路由规则以外，使用java代码的方式也可以试想同样的功能：

```java
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import java.time.LocalDateTime;
import java.time.Month;
import java.time.ZoneId;
import java.time.ZonedDateTime;

@Configuration
public class RouteConfig {
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder
                .routes()
                //.route("test_route", r -> r.path("/test").uri("https://www.baidu.com"))
                .route("service_a_route", r -> r.path("/a/**").filters(f -> f.stripPrefix(1)).uri("lb://service-a"))
                //.route("test_route",r->r.before(ZonedDateTime.of(LocalDateTime.of(2019, Month.AUGUST,20,8,30,0), ZoneId.of("Asia/Shanghai"))).uri("https://www.baidu.com"))
                //.route("test_route",r->r.after(ZonedDateTime.of(LocalDateTime.of(2019, Month.AUGUST,20,8,30,0), ZoneId.of("Asia/Shanghai"))).uri("https://www.baidu.com"))
                //.route("test_route",r->r.between(ZonedDateTime.of(LocalDateTime.of(2019, Month.AUGUST,20,8,30,0), ZoneId.of("Asia/Shanghai")),ZonedDateTime.of(LocalDateTime.of(2019, Month.AUGUST,20,20,30,0), ZoneId.of("Asia/Shanghai"))).uri("https://www.baidu.com"))
                //.route("test_route",r->r.cookie("testKey","testValue").uri("https://www.baidu.com"))
                //.route("test_route",r->r.header("X-Request-Id","\\d+").uri("https://www.baidu.com"))
                //.route("test_route",r->r.host("**.test.com").uri("https://www.baidu.com"))
                //.route("test_route",r->r.method(HttpMethod.GET).uri("https://www.baidu.com"))
                //.route("test_route",r->r.query("testParam","testValue").uri("https://www.baidu.com"))
                //.route("test_route",r->r.remoteAddr("192.168.1.1").uri("https://www.baidu.com"))

                //.route("add_request_param_route",r->r.path("/hi").filters(f->f.addRequestParameter("name","zhangsan")).uri("lb://service-a"))
                .build();
    }
}
```

### springcloud后端服务

上面搭建SCG服务器配置中路由的服务service-a就是一个普通的微服务，注册到eureka注册中心，提供一个简单的/hi接口返回字符串。

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
</dependencies>
```

```java
@SpringBootApplication
@EnableDiscoveryClient
public class ServiceAApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceAApplication.class,args);
    }
}

@RestController
public class AController {
    @Value("${server.port}")
    String port;

    @GetMapping("/hi")
    public String sayHi(@RequestParam("name") String name) {
        return String.format("Hi %s, i'm from service-a, port %s",name,port);
    }
}
```

```bash
spring.application.name=service-a
server.port=8001
eureka.client.service-url.defaultZone=http://localhost:8761/eureka/
# 每间隔2s，向eureka服务端发送一次心跳，证明自己依然“活着”
eureka.instance.lease-renewal-interval-in-seconds=2
# 告诉服务端，如果我5s之内没有给你发心跳，就代表我“死”了，将我剔除掉
eureka.instance.lease-expiration-duration-in-seconds=5
```

### 测试

1.启动eureka注册中心项目springcloud.f.eureka.server

2.启动SCG服务器springcloud.f.gateway.server

3.启动微服务springcloud.f.gateway.service-a，可以启动两个实例演示负载均衡

4.示例中使用了`/a/**`这个规则将所有请求路由到service-a服务

```bash
# 访问2个微服务实例
http://localhost:8001/hi?name=abc
http://localhost:8002/hi?name=abc
# 访问网关，可以发现服务通过负载均衡的方式访问了service-a服务
http://localhost:8900/a/hi?name=john
```

## 限流



## 熔断降级

## 其它

### 路由信息端点

通过actuator可以查看SCG项目的很多信息，访问格式为：/actuator/gateway/{ID}

| ID            | HTTP Method | Description                                                                 |
| ------------- | ----------- | --------------------------------------------------------------------------- |
| globalfilters | GET         | Displays the list of global filters applied to the routes.                  |
| routefilters  | GET         | Displays the list of GatewayFilter factories applied to a particular route. |
| refresh       | POST        | Clears the routes cache.                                                    |
| routes        | GET         | Displays the list of routes defined in the gateway.                         |
| routes/{id}   | GET         | Displays information about a particular route.                              |
| routes/{id}   | POST        | Add a new route to the gateway.                                             |
| routes/{id}   | DELETE      | Remove an existing route from the gateway.                                  |

```bash
# 启用gateway这个端点（需要依赖actuator组件）
management.endpoint.gateway.enabled=true
management.endpoints.web.exposure.include=gateway

# 查看所有的路由规则
/actuator/gateway/routes
# 查看某一个路由规则
/actuator/gateway/routes/{route_id}
# 查看GlobalFilter列表
/actuator/gateway/globalfilters
# 查看GatewayFilter列表
/actuator/gateway/routefilters
# 刷新缓存
/actuator/gateway/refresh

# 通过POST请求添加一个route
# 通过DELETE请求删除一个route
/gateway/routes/{id_route_to_create}
```

### 自定义路由规则

一般情况下通过SCG提供的内置predicate+filter基本可以实现大部分需求，如果想自定义，也可以创建自己的GlobalFilter、GatewayFilter。

[Writing Custom Global Filters](https://cloud.spring.io/spring-cloud-gateway/reference/html/#_writing_custom_gatewayfilter_factories)

### 过滤器优先级

SCG的FilteringWebHandler会将所有的过滤器（包括GlobalFilter、GatewayFilter）放在一起形成一个chain，各个过滤器之间的顺序可以通过`@Order`来指定，数字越小，优先级越高，`pre`操作越先执行，而`post`操作越后执行。示例：

```java
@Bean
@Order(-1)
public GlobalFilter a() {
    return (exchange, chain) -> {
        log.info("first pre filter");
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            log.info("third post filter");
        }));
    };
}

@Bean
@Order(0)
public GlobalFilter b() {
    return (exchange, chain) -> {
        log.info("second pre filter");
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            log.info("second post filter");
        }));
    };
}

@Bean
@Order(1)
public GlobalFilter c() {
    return (exchange, chain) -> {
        log.info("third pre filter");
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            log.info("first post filter");
        }));
    };
}
```

### CORS

## 参考

[spring-cloud-gateway-sample](https://github.com/spring-cloud-samples/spring-cloud-gateway-sample)

[spring-cloud-gateway-documentation](https://cloud.spring.io/spring-cloud-gateway/reference/html/)

[Spring Cloud Gateway入门](http://www.ityouknow.com/springcloud/2018/12/12/spring-cloud-gateway-start.html)

[spring cloud gateway系列教程](https://zhuanlan.zhihu.com/p/54697618)
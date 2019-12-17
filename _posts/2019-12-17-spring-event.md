---
layout: post
title:  "spring中的事件机制"
categories: spring
tags: spring
author: 网络
---

* content
{:toc}

总结spring 事件相关的知识点










## 介绍

spring中的事件发布-订阅机制依赖几个接口:

* 1.**ApplicationEvent**：继承自EventObject，同时是spring的application中事件的父类，需要被自定义的事件继承。

* 2.**ApplicationListener**：继承自EventListener，spring的application中的监听器必须实现的接口，需要被自定义的监听器实现其onApplicationEvent方法

* 3.**ApplicationEventPublisherAware**：在spring的context中希望能发布事件的类必须实现的接口，该接口中定义了设置ApplicationEventPublisher的方法，由ApplicationContext调用并设置。在自己实现的ApplicationEventPublisherAware子类中，需要有ApplicationEventPublisher属性的定义。

* 4.**ApplicationEventPublisher**：spring的事件发布者接口，定义了发布事件的接口方法publishEvent。因为ApplicationContext实现了该接口，因此spring的ApplicationContext实例具有发布事件的功能(publishEvent方法在AbstractApplicationContext中有实现)。在使用的时候，只需要把ApplicationEventPublisher的引用定义到ApplicationEventPublisherAware的实现中，spring容器会完成对ApplicationEventPublisher的注入。

## 示例

使用springboot应用来做示例，pom依赖添加web依赖即可

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

模拟用户注册事件的发布-订阅，首先定义事件对象：

```java
/**
 * 自定义事件
 */
public class UserRegisterEvent extends ApplicationEvent {
    public UserRegisterEvent(String name) {
        super(name);
    }
}
```

定义事件发布者，使用两种方式来发布事件，一种是实现接口ApplicationEventPublisherAware，使用其提供的ApplicationEventPublisher对象来发布事件，另外一种就是直接使用ApplicationContext来发布事件，ApplicationContext其实实现了ApplicationEventPublisher接口，本身就具有发布事件的能力：

```java
/**
 * 用来发布自定义事件的服务
 */
@Service
public class UserService implements ApplicationEventPublisherAware, ApplicationContextAware {

    private ApplicationEventPublisher applicationEventPublisher;

    private ApplicationContext applicationContext;

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        this.applicationEventPublisher = applicationEventPublisher;
    }

    /**
     * 通过ApplicationEventPublisher或者直接通过ApplicationContext都可以发布事件
     *
     * @param name
     */
    public void register(String name) {
        System.out.println("用户：" + name + " 已注册！");
        applicationEventPublisher.publishEvent(new UserRegisterEvent(name + "1"));
        applicationContext.publishEvent(new UserRegisterEvent(name + "2"));
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
```

有两种监听事件的方式，一种是直接实现ApplicationListener接口，另外一种是使用@EventListener注解：

```java
@Service
public class UserRegisterListener1 implements ApplicationListener<UserRegisterEvent> {
    @Override
    public void onApplicationEvent(UserRegisterEvent userRegisterEvent) {
        System.out.println("1邮件服务接到通知，给 " + userRegisterEvent.getSource() + " 发送邮件...");
    }
}
```

```java
@Service
public class UserRegisterListener2 {

    @Async
    @EventListener
    public void onApplicationEvent(UserRegisterEvent userRegisterEvent) {
        System.out.println("2邮件服务接到通知，给 " + userRegisterEvent.getSource() + " 发送邮件...");
    }
}
```

测试类：

```java
@SpringBootApplication
@RestController
public class EventDemoApplication {

    @Autowired
    UserService userService;

    public static void main(String[] args) {
        SpringApplication.run(EventDemoApplication.class, args);
    }

    @RequestMapping("/register")
    public String register(){
        userService.register("zhangsan");
        return "success";
    }
}
```

输出：

```text
用户：zhangsan 已注册！
2邮件服务接到通知，给 zhangsan1 发送邮件...
1邮件服务接到通知，给 zhangsan1 发送邮件...
2邮件服务接到通知，给 zhangsan2 发送邮件...
1邮件服务接到通知，给 zhangsan2 发送邮件...
```

## 参考

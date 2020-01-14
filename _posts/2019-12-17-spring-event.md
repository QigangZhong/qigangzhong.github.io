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

springboot中新增事件机制：

* **SpringApplicationRunListeners、SpringApplicationRunListener**：SpringApplicationRunListeners是SpringApplicationRunListener的简单封装，通过遍历调用每个监听器的方法

  * **EventPublishingRunListener**：是SpringApplicationRunListener的唯一一个内置实现类，springboot启动时会默认加载进来，springboot利用EventPublishingRunListener间接调用spring中的ApplicationEvent

  * **SimpleApplicationEventMulticaster**：EventPublishingRunListener利用这个广播器来广播事件
  
  * **SpringApplicationEvent**
* 

## ApplicationEvent

### 示例

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

## SpringApplicationRunListener

事件监听器在以下事件发生时会被调用：

* 开始启动
* Environment构建完成
* ApplicationContext构建完成
* ApplicationContext完成加载
* ApplicationContext完成刷新并启动
* 启动完成
* 启动失败

![SpringApplicationEvent.png](/images/spring/SpringApplicationEvent.png)

### 示例

自定义一个事件监听器

```java
public class MySpringApplicationRunListener implements SpringApplicationRunListener {
    private final SpringApplication springApplication;
    private final String[] args;

    public MySpringApplicationRunListener(SpringApplication springApplication, String[] args){
        this.springApplication = springApplication;
        this.args = args;
    }

    @Override
    public void starting() {
        System.out.println("my starting");
    }

    @Override
    public void environmentPrepared(ConfigurableEnvironment environment) {
        System.out.println("my environmentPrepared");
    }

    @Override
    public void contextPrepared(ConfigurableApplicationContext context) {
        System.out.println("my contextPrepared");
    }

    @Override
    public void contextLoaded(ConfigurableApplicationContext context) {
        System.out.println("my contextLoaded");
    }

    @Override
    public void started(ConfigurableApplicationContext context) {
        System.out.println("my started");
    }

    @Override
    public void running(ConfigurableApplicationContext context) {
        System.out.println("my running");
    }

    @Override
    public void failed(ConfigurableApplicationContext context, Throwable exception) {
        System.out.println("my failed");
    }
}
```

配置到META-INF/spring.factories中

```properties
org.springframework.boot.SpringApplicationRunListener=com.qigang.sb.e.MySpringApplicationRunListener
```

一个普通的springboot启动类启动应用，观察输出日志

```java
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class,args);
    }
}
```

### EventPublishingRunListener

EventPublishingRunListener是springboot中唯一一个SpringApplicationRunListener的实现类，默认会被加载，跟踪到EventPublishingRunListener的源码中可以看到，它其实是spring的事件ApplicationEvent的代理。

如果在上面的示例中添加一个普通的ApplicationListener，监听springboot启动完成事件，也是可以的，当然如果你要监听的事件在自定义的事件监听器被加载到容器之前肯定不行的，比如你想监听springboot启动事件ApplicationStartingEvent，在这个事件发布的时候MyApplicationListener根本都还没有被加载到容器里面，所以onApplicationEvent方法也不可能被调用。

```java
@Component
public class MyApplicationListener implements ApplicationListener<ApplicationReadyEvent> {
    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        System.out.println("my onApplicationEvent");
    }
}
```


## 参考

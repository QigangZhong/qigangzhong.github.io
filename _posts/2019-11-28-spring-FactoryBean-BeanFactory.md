---
layout: post
title:  "FactoryBean BeanFactory"
categories: spring
tags: spring
author: 网络
---

* content
{:toc}

总结FactoryBean以及BeanFactory相关的知识点










## FactoryBean

一般情况下在spring配置文件或者通过注解的方式就可以定义一个bean加载到spring容器中，当需要加载一个复杂的bean，或者动态加载指定的bean的情况下，可以使用[FactoryBean](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/beans/factory/FactoryBean.html)来完成，通过实现`FactoryBean.getObject`方法自定义实例化bean的过程。例如MyBatis中的[SqlSessionFactoryBean](https://mybatis.org/spring/zh/factorybean.html)就是利用FactoryBean来实现的。

### xml配置使用案例

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
</dependency>
```

```java
public class MyBean {
    private String name;
    private Integer age;

    //getter,setter,toString...
}

public class MyFactoryBean implements FactoryBean<MyBean> {

    private String name;
    private Integer age;

    @Override
    public MyBean getObject() throws Exception {
        MyBean myBean = new MyBean();
        myBean.setName(this.name);
        myBean.setAge(this.age);
        return myBean;
    }

    @Override
    public Class<?> getObjectType() {
        return MyBean.class;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }

    //getter,setter...
}
```

```xml
<!-- factory-bean.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="mybean" class="com.qigang.sblc.factoryBean.MyFactoryBean">
        <property name="name" value="zhangsan"/>
        <property name="age" value="30"/>
    </bean>
</beans>
```

```java
public class SpringFactoryBeanApplication {
    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("factory-bean.xml");
        //xml中定义的是MyFactoryBean，但实际从容器拿到的是getObject方法返回的MyBean对象
        MyBean myBean1 = (MyBean) applicationContext.getBean("mybean");
        System.out.println(myBean1);
        MyBean myBean2 = applicationContext.getBean(MyBean.class);
        System.out.println(myBean2);
        System.out.println(myBean1==myBean2);

        //如果想获取到MyFactoryBean本身，在bean名称前加一个&符号即可
        MyFactoryBean myFactoryBean = (MyFactoryBean) applicationContext.getBean("&mybean");
        System.out.println(myFactoryBean);
    }
}
```

### 注解使用案例

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

```java
@SpringBootApplication
@RestController
public class SpringBootFactoryBeanApplication {
    @Autowired
    private MyBean myBean;

    public static void main(String[] args) {
        SpringApplication.run(SpringBootFactoryBeanApplication.class,args);
    }

    //单独注入bean，或者在MyFactoryBean上加@Component注解
    @Bean
    public MyFactoryBean getMyFactoryBean(){
        MyFactoryBean myFactoryBean = new MyFactoryBean();
        myFactoryBean.setName("zhangsan");
        myFactoryBean.setAge(20);
        return myFactoryBean;
    }

    @GetMapping("/getMyBean")
    public String getMyBean(){
        return myBean.toString();
    }
}
```

## BeanFactory

### BeanFactory基础

#### 接口类图

![beanfactory](/images/spring/beanfactory.png)

#### 基础实现接口

* ListableBeanFactory 支持bean迭代
* HierarchicalBeanFactory 支持层级
* AutowireCapableBeanFactory 支持自动织入

#### 默认实现类

* DefaultListableBeanFactory（纯粹的IOC容器实现）

```java
//使用默认的DefaultListableBeanFactory来加载bean的简单示例代码
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(new ClassPathResource("bean-factory.xml"));
MyBean bean = factory.getBean(MyBean.class);
System.out.println(bean.toString());
```

#### BeanDefinition

#### BeanDefinitionRegistry

DefaultListableBeanFactory是BeanDefinitionRegistry的一个实现，实现方法`registerBeanDefinition`中将载入的BeanDefinition加入到ConcurrentHashMap中

### ApplicationContext

#### 类图

![applicationcontext](/images/spring/ApplicationContext.png)

#### AbstractApplicationContext.refresh（IOC初始化的核心方法）

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        //加载BeanDefinition，注册BeanDefinition的逻辑都在这里，默认返回DefaultListableBeanFactory
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            initMessageSource();

            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // Check for listener beans and register them.
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            // Bean的实例化在这个方法里面
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
```

##### Bean的创建过程（生命周期中的主要方法）

```java
AbstractApplicationContext.refresh();
//这里的beanFactory就是默认创建的DefaultListableBeanFactory
finishBeanFactoryInitialization(beanFactory);
preInstantiateSingletons();
getBean();
doGetBean();
AbstractAutowireCapableBeanFactory.createBean();
AbstractAutowireCapableBeanFactory.resolveBeforeInstantiation();
//`applyBeanPostProcessorsBeforeInstantiation()`以及`applyBeanPostProcessorsAfterInitialization()`就是在这个方法里面调用的
AbstractAutowireCapableBeanFactory.doCreateBean();
```

在`AbstractAutowireCapableBeanFactory.doCreateBean()`方法中有几个重要方法：

```java
//创建bean实例
createBeanInstance();
//执行BeanDefinitionPostProcessors
applyMergedBeanDefinitionPostProcessors();
//依赖注入
populateBean();
//初始化，调用各种aware接口方法，调用InitializingBean.afterPropertiesSet()，调用init-method
//调用applyBeanPostProcessorsBeforeInitialization、applyBeanPostProcessorsAfterInitialization
initializeBean();
```

```java
AbstractAutowireCapableBeanFactory.initializeBean();

//*******************************************************************************
//如果设置了系统权限，则忽略权限检查，直接执行
if (System.getSecurityManager() != null) {
    beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
        @Override
        public Object run() {
            return getInstantiationStrategy().instantiate(mbd, beanName, parent);
        }
    }, getAccessControlContext());
}
else {
    beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
}
//*******************************************************************************

SimpleInstantiationStrategy.instantiate();
//使用反射的方式创建bean实例
BeanUtils.instantiateClass();
```

#### ClassPathApplicationContext

![ClassPathApplicationContext](/images/spring/ClassPathApplicationContext.png)

#### AnnotationConfigApplicationContext

## 参考

[FactoryBean的作用](https://blog.csdn.net/u014082714/article/details/81166648)

[FactoryBean——Spring的扩展点之一](https://juejin.im/post/5d8e06b06fb9a04e1c07d87b)

[Spring源码分析](https://blog.csdn.net/u014634338/category_8087902.html)

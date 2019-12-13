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



## 参考

[FactoryBean的作用](https://blog.csdn.net/u014082714/article/details/81166648)

[FactoryBean——Spring的扩展点之一](https://juejin.im/post/5d8e06b06fb9a04e1c07d87b)

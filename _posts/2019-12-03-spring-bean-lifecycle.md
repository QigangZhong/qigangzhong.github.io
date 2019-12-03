---
layout: post
title:  "Spring Bean的生命周期"
categories: spring
tags:  spring
author: 网络
---

* content
{:toc}












# [(转)Spring Bean的生命周期](https://blog.csdn.net/u012385190/article/details/81368748)

Spring的生命周期是指实例化Bean时所经历的一系列阶段，即通过getBean()获取bean对象及设置对象属性时，Spring框架做了哪些事。Bean的生命周期从Spring容器实例化Bean到销毁Bean。
本文分别对 BeanFactory 和 ApplicationContext 中的生命周期进行分析。

## 一、BeanFactory实例化Bean相关接口

### Bean级生命周期接口：(4个)

#### 1、BeanNameAware

```java
//待对象实例化并设置属性之后调用该方法设置BeanName
void setBeanName(String beanName);
```

#### 2、BeanFactoryAware

```java
//待调用setBeanName之后调用该方法设置BeanFactory，BeanFactory对象默认实现类是DefaultListableBeanFactory
void setBeanFactory(BeanFactory var1) throws BeansException;
```

#### 3、InitializingBean

```java
//实例化完成之后调用（调用了BeanPostProcessor.postProcessBeforeInitialization方法之后调用该方法）
void afterPropertiesSet() throws Exception;
```

#### 4、DisposableBean

```java
//关闭容器时调用
void destroy() throws Exception;
```

这4个接口都在包 org.springframework.beans.factory 下，它们是Bean级生命周期接口，这些接口由Bean类直接实现。

### 容器级Bean生命周期接口：(2个)

#### 1、抽象类：InstantiationAwareBeanPostProcessorAdapter

实例化前／后，及框架设置Bean属性时调用该接口。可覆盖的常用方法有：

```java
//在Bean对象实例化前调用
@Override
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException;
 
//在Bean对象实例化后调用（如调用构造器之后调用）
@Override
public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException;
 
/**
 * 在设置某个属性前调用，然后再调用设置属性的方法
 * 注意：这里的设置属性是指通过配置设置属性，直接调用对象的setXX方法不会调用该方法，如bean配置中配置了属性address/age属性，将会调用该方法
 * @param pvs 如 PropertyValues: length=2; bean property 'address'; bean property 'age'
 */
@Override
public PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException;
```
#### 2、接口BeanPostProcessor

实例化完成之后调用该接口。可实现的接口方法有：

```java
//实例化完成，setBeanName/setBeanFactory完成之后调用该方法
public Object postProcessBeforeInitialization(Object o, String s) throws BeansException;
 
//全部是实例化完成以后调用该方法
public Object postProcessAfterInitialization(Object o, String s) throws BeansException;
```
这两个接口都在包 org.springframework.beans.factory.config 下，一般称它们的实现类为“后处理器”。后处理器接口一般不由Bean本身实现，实现类以容器附加装置的形式注册到Spring容器中。
当Sprig容器创建任何Bean的时候，这两个后处理器都会发生作用，所以这两个后处理器的影响是全局的。用户可以通过合理的代码控制后处理器只对固定的Bean创建进行处理。
Bean级生命周期接口解决Bean个性化处理的问题，Bean容器级生命周期接口解决容器中某些Bean共性化处理的问题。

## 二、BeanFactory的bean生命周期相关代码

### 1、xml bean 配置

```xml
<bean id="person" class="demo02.bean.Person"
      p:address="上海市"
      p:age="25"
      init-method="myInit"
      destroy-method="myDestroy"
      />
```

p:address／p:age 在实例化的时候会调用Bean的对应setXX方法设置属性。

### 2、java相关代码

#### InstantiationAwareBeanPostProcessorAdapter实现类相关代码

```java
/**
 * 实例化前／后，及框架设置Bean属性时调用该接口
 */
public class MyInstantiationAwareBeanPostProcessor extends InstantiationAwareBeanPostProcessorAdapter {

    //在Bean对象实例化前调用
    @Override
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        //仅对容器中的person bean处理
        if ("person".equals(beanName)) {
            System.out.println("实例化前调用：InstantiationAwareBeanPostProcessorAdapter.postProcessBeforeInstantiation");
        }
        return null;
    }

    //在Bean对象实例化后调用（如调用构造器之后调用）
    @Override
    public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        //仅对容器中的person bean处理
        if ("person".equals(beanName)) {
            System.out.println("实例化后调用：InstantiationAwareBeanPostProcessorAdapter.postProcessAfterInstantiation");
        }
        return true;
    }

   /**
    * 在设置某个属性前调用，然后再调用设置属性的方法
    * 注意：这里的设置属性是指通过配置设置属性，直接调用对象的setXX方法不会调用该方法，如bean配置中配置了属性address/age属性，将会调用该方法
    * @param pvs 如 PropertyValues: length=2; bean property 'address'; bean property 'age'
    */
    @Override
    public PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {
        //仅对容器中的person bean处理
        if ("person".equals(beanName)) {
            System.out.println("在Bean设置属性时调用：InstantiationAwareBeanPostProcessorAdapter.postProcessPropertyValues，设置值为" + pvs);
        }
        return pvs;
    }
}
```

#### BeanPostProcessor实现类相关代码

```java
/**
 * 实例化完成之后调用该接口
 */
public class MyBeanPostProcessor implements BeanPostProcessor {

    //实例化完成，setBeanName/setBeanFactory完成之后调用该方法
    public Object postProcessBeforeInitialization(Object o, String s) throws BeansException {
        if ("person".equals(s)) {
            Person person = (Person) o;
            if (person.getName() == null) {
                System.out.println("调用BeanPostProcessor.postProcessBeforeInitialization,name为空，设置默认名为无名氏");
                person.setName("无名氏");
            }
        }
        return o;
    }

    //全部是实例化完成以后调用该方法
    public Object postProcessAfterInitialization(Object o, String s) throws BeansException {
        if ("person".equals(s)) {
            Person person = (Person) o;
            if (person.getAge()>20) {
                System.out.println("调用BeanPostProcessor.postProcessAfterInitialization,age大于20，调整为20");
                person.setAge(20);
            }
        }
        return o;
    }
}
```

#### BeanFactoryLifeCycleMain 容器装载配置文件并启动

```java
/**
 * BeanFactory中的Bean的生命周期
 * 容器装载配置文件，注册 BeanPostProcessor 和 InstantiationAwareBeanPostProcessorAdapter 后置处理器
 */
public class BeanFactoryLifeCycleMain {

    @Test
    public void lifeCycleInBeanFactory() {
        //装载配置文件并启动容器
        Resource resource = new ClassPathResource("beans/beans.xml");
        BeanFactory beanFactory = new DefaultListableBeanFactory();
        XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader((DefaultListableBeanFactory) beanFactory);
        reader.loadBeanDefinitions(resource);

        //向容器中注册后处理器 MyBeanPostProcessor
        ((DefaultListableBeanFactory) beanFactory).addBeanPostProcessor(new MyBeanPostProcessor());

        //向容器中注册后处理器 MyInstantiationAwareBeanPostProcessor
        //注意：后处理器调用顺序和注册顺序无关。在处理多个后处理器的情况下，必需通过实现Ordered接口来确定顺序
        ((DefaultListableBeanFactory) beanFactory).addBeanPostProcessor(new MyInstantiationAwareBeanPostProcessor());

        //第一次从容器中获取Person，将触发容器实例化该bean，这将引发Bean生命周期方法的调用
        Person person = (Person) beanFactory.getBean("person");
        person.introduce();
        person.setName("zhangsan");

        //第二次从容器中获取，直接从缓存中获取(默认就是单例)
        Person person2 = (Person) beanFactory.getBean("person");
        System.out.println(person == person2);//true

        //关闭容器
        ((DefaultListableBeanFactory) beanFactory).destroySingletons();
    }
}
```

执行结果：（DEBUG信息为Spring框架输出的，说明了底层调用方法和作用）

> 实例化前调用：InstantiationAwareBeanPostProcessorAdapter.postProcessBeforeInstantiation
> 调用了Person的无参构造器
> DEBUG 2018-08-02 13:51:07: [org.springframework.beans.factory.support.DefaultListableBeanFactory.(523)doCreateBean] - Eagerly caching bean 'person' to allow for resolving potential circular references
> 实例化后调用：InstantiationAwareBeanPostProcessorAdapter.postProcessAfterInstantiation
> 在Bean设置属性时调用：InstantiationAwareBeanPostProcessorAdapter.postProcessPropertyValues，设置值为PropertyValues: length=2; bean property 'address'; bean property 'age'
> 调用了setAddress方法设置了属性
> 调用了setAge方法设置了属性
> 调用了BeanNameAware.setBeanName, value is person
> 调用了BeanFactoryAware.setBeanFactory, value is org.springframework.beans.factory.support.DefaultListableBeanFactory@44c8afef: defining beans [car,person]; root of factory hierarchy
> 调用BeanPostProcessor.postProcessBeforeInitialization,name为空，设置默认名为无名氏
> 调用了setName方法设置了属性
> DEBUG 2018-08-02 13:51:07: [org.springframework.beans.factory.support.DefaultListableBeanFactory.(1595)invokeInitMethods] - Invoking afterPropertiesSet() on bean with name 'person'
> 调用InitializingBean.afterPropertiesSet
> DEBUG 2018-08-02 13:51:07: [org.springframework.beans.factory.support.DefaultListableBeanFactory.(1653)invokeCustomInitMethod] - Invoking init method  'myInit' on bean with name 'person'
> 调用bean配置的init-method
> 调用BeanPostProcessor.postProcessAfterInitialization,age大于20，调整为20
> 调用了setAge方法设置了属性
> DEBUG 2018-08-02 13:51:07: [org.springframework.beans.factory.support.DefaultListableBeanFactory.(477)createBean] - Finished creating instance of bean 'person'
> 调用了introduce方法--> name:无名氏,age:20,address:中国
> 调用了setName方法设置了属性
> DEBUG 2018-08-02 13:51:07: [org.springframework.beans.factory.support.DefaultListableBeanFactory.(249)doGetBean] - Returning cached instance of singleton bean 'person'
> true
> DEBUG 2018-08-02 13:51:07: [org.springframework.beans.factory.support.DefaultListableBeanFactory.(474)destroySingletons] - Destroying singletons in org.springframework.beans.factory.support.DefaultListableBeanFactory@44c8afef: defining beans [car,person]; root of factory hierarchy
> DEBUG 2018-08-02 13:51:07: [org.springframework.beans.factory.support.DisposableBeanAdapter.(244)destroy] - Invoking destroy() on bean with name 'person'
> 调用DisposableBean.destroy
> DEBUG 2018-08-02 13:51:07: [org.springframework.beans.factory.support.DisposableBeanAdapter.(322)invokeCustomDestroyMethod] - Invoking destroy method 'myDestroy' on bean with name 'person'
> 调用bean配置的destroy-method

BeanFactory的Bean生命周期执行步骤如下：

![](https://img-blog.csdn.net/20180802223029954?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTIzODUxOTA=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

## 三、关于BeanFactory的Bean生命周期接口的总结

* 1、通过实现BeanFactory的Bean生命周期相关接口，虽然让Bean具有了更详细的生命周期阶段，但是也让Spring框架和Bean紧密联系在了一起，同时也增加了代码复杂度。因此，如果用户希望在Bean的生命周期中实现自己的业务，不需要和特定框架关联，可以通过 bean 的 init-method 和 destroy-method 配置方法来进行业务实现。
* 2、Spring还拥有一个Bean后置处理器InitDestroyAnnotationBeanPostProcessor，它负责对标注了 @PostConstruct 和 @PreDestroy 的 Bean 进行处理，在 Bean 初始化及销毁前执行相应的业务逻辑。
* 3、通常情况下，可以抛弃Bean级的生命周期的4个接口，用更加方便的方法进行代替（如1、2点中的方法）。但是容器级Bean生命周期接口可以合理的使用，处理一些共同的业务。
* 4、当bean的Scope为非单例，对象销毁的时候，Spring容器不会帮我们调用任何方法，因为是非单例，这个类型的对象有很多个，Spring容器一旦把这个对象交给你之后，就不再管理这个对象了。如果bean的scope设为prototype时，当容器关闭时，destroy方法不会被调用。对于prototype作用域的bean，Spring不能对一个该bean的整个生命周期负责：容器在初始化、配置、装饰或者是装配完一个prototype实例后，将它交给客户端，随后就对该prototype实例不闻不问了。

## 四、ApplicationContext的Bean生命周期

* 1、Bean在应用上下文中的生命周期和在BeanFactory中生命周期类似，可以实现所有的BeanFactory的Bean级和容器级生命周期接口。但是，如果Bean实现了org.springframework.context.ApplicationContextAware接口，会增加一个调用该接口方法setApplicationContext()的步骤。
* 2、如果配置文件中声明了工厂后处理器接口BeanFactoryPostProcessor的实现类，则应用上下文在装载配置文件之后初始化Bean实例之前将调用这些BeanFactoryPostProcessor对配置信息进行加工处理。Spring框架提供了多个工厂后处理器如CustomEditorConfigurer、PopertyPlaceholderConfigurer等。如果配置文件中定义了多个工厂后处理器，最好让它们实现org.springframework.core.Ordered接口，以便Spring以确定的顺序调用它们。工厂后处理器是容器级的，仅在应用上下文初始化时调用一次，其目的是完成一些配置文件的加工处理工作。
  注意：BeanFactoryPostProcessor的接口方法参数是BeanFactory，而在BeanFactory级Bean生命周期中提到的BeanPostProcessor接口方法参数是Bean对象和Bean的ID值。所以前者是针对应用上下文的，后者是针对BeanFactory的。
* 3、ApplicationContext和BeanFactory另一个最大的不同之处在于：前者会利用Java反射机制自动识别出配置文件中定义的BeanPostProcessor、InstantiationAwareBeanPostProcessor和BeanFactoryPostProcessor，并自动将它们注册到应用上下文中；而后者需要在代码中通过手工调用addBeanPostProcessor()方法进行注册。这也是为什么在应用开发时，我们普遍使用ApplicationContext而很少使用BeanFactory的原因之一。
* 4、在ApplicationContext中，我们只需要在配置文件中通过<bean>定义工厂后处理器和Bean后处理器，它们就会按预期的方式运行。

## 五、ApplicationContext的Bean生命周期代码演示

#### 1、java代码

实现BeanFactoryPostProcessor接口方法

```java
/**
 * ApplicationContext bean生命周期演示
 */
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

    //先调用本方法，将所有bean生成BeanDefinition对象并设置相关属性。
    //本方法运行完之后，会调用bean构造器，并调用set相关属性方法
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        BeanDefinition beanDefinition = beanFactory.getBeanDefinition("person");
        beanDefinition.getPropertyValues().addPropertyValue("address", "王二麻子");
        System.out.println("调用了BeanFactoryPostProcessor.postProcessBeanFactory");
    }
}
```

ApplicationContext在启动时，将首先为配置文件中每个<bean>生成一个BeanDefinition对象，BeanDefinition是<bean>在Spring容器中的内部表示。当配置文件中所有的<bean>都被解析成BeanDefinition时，ApplicationContext将调用工厂后处理器的方法，因此我们有机会通过程序的方式调整Bean的配置信息。如上对person的属性address进行更改设置。

#### 2、xml 相关配置

```xml
<!--1、注册Bean-->
<bean id="person" class="demo02.bean.Person"
      p:address="上海市"
      p:age="25"
      init-method="myInit"
      destroy-method="myDestroy"
      />
<!--下面注册bean后处理器是为了演示ApplicationContext bean生命周期的。-->
<!--ApplicationContext和BeanFactory另一个最大的不同之处在于：
前者会利用Java反射机制自动识别出配置文件中定义的BeanPostProcessor、InstantiationAwareBeanPostProcessor和BeanFactoryPostProcessor，并自动将它们注册到应用上下文中；
而后者需要在代码中通过手工调用addBeanPostProcessor()方法进行注册。-->
<!--2、注册Bean后处理器-->
<bean id="myBeanPostProcessor" class="demo02.BeanFactoryLifeCycle.MyBeanPostProcessor"/>
<!--3、注册Bean后处理器-->
<bean id="myBeanFactoryPostProcessor" class="demo02.BeanFactoryLifeCycle.MyBeanFactoryPostProcessor"/>
```
> ②和③处定义的BeanPostProcessor和BeanFactoryPostProcessor会自动被ApplicationContext识别并注册到容器中。②处注册的工厂后处理器将会对①处配置的属性值进行调整。在③处，我们还声明了一个Bean后处理器，它也可以对Bean的属性进行调整。启动容器并查看car Bean的信息，我们将发现car Bean的brand属性成功被工厂后处理器更改了。

#### 3、测试类

```java
/**
 * ApplicationContext bean生命演示
 */
public class ApplicationContexBeanLifeCycleMain {
    public static void main(String[] args) {
        //ApplicationContext applicationContext = new ClassPathXmlApplicationContext("beans/beans.xml");
        //ApplicationContext 接口中没有实现close方法，所以可以用该类（该类是实现了ConfigurableApplicationContext接口和DisposableBean接口）
        AbstractApplicationContext applicationContext = new ClassPathXmlApplicationContext("beans/beans.xml");
        Person person = (Person) applicationContext.getBean("person");
        person.introduce();
        applicationContext.close();
    }
}
```

#### 4、运行结果

最后输出：

> 调用了BeanFactoryPostProcessor.postProcessBeanFactory
> 调用了Person的无参构造器
> 调用了setAddress方法设置了属性，上海市
> 调用了setAge方法设置了属性，25
> 调用了setName方法设置了属性，王二麻子
> 调用了BeanNameAware.setBeanName, value is person
> 调用了BeanFactoryAware.setBeanFactory, value ……
> 调用InitializingBean.afterPropertiesSet
> 调用bean配置的init-method
> 调用BeanPostProcessor.postProcessAfterInitialization,age大于20，调整为20
> 调用了setAge方法设置了属性，20
> 调用了introduce方法--> name:王二麻子,age:20,address:中国
> 调用DisposableBean.destroy
> 调用bean配置的destroy-method

## 六、总结

* 1、Spring的生命周期可以从 BeanFactory 和 ApplicationContext 中的生命周期进行分析；

* 2、BeanFactory实例化Bean相关接口分为Bean级和容器级。bean级接口我们一般不实现，容器级如果需要处理一些共有特性，可以考虑实现；

* 3、ApplicationContext中bean的生命周期与BeanFactory类似。beanFactory中的后处理器接口需要调用addBeanPostProcessor方法进行注册，而ApplicationContext中的后处理器可以通过配置和反射进行调用；

* 4、从上可以看到Bean的生命周期相关接口很多，如果都实现很复杂。通常我们的业务都可以通过 init-method 和 destory-method 方法配置来解决，这样做简单粗暴。

## 参考

参考书籍：《精通Spring4.x企业应用开发实战》
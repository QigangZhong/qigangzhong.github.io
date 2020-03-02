---
layout: post
title:  "spring ioc"
categories: spring
tags: spring
author: 网络
---

* content
{:toc}

总结spring ioc相关的知识点










## 介绍

IOC(Inversion Of Control)/DI(Dependence Injection)的目的是将对象交由容器来管理，在spring中容器的顶层接口是BeanFactory，这个容器接口里面定义了对象bean的各种操作方法。

```java
public interface BeanFactory {
    String FACTORY_BEAN_PREFIX = "&";
    Object getBean(String name) throws BeansException;
    <T> T getBean(String name, Class<T> requiredType) throws BeansException;
    Object getBean(String name, Object... args) throws BeansException;
    <T> T getBean(Class<T> requiredType) throws BeansException;
    <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;
    boolean containsBean(String name);
    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;
    boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;
    boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;
    Class<?> getType(String name) throws NoSuchBeanDefinitionException;
    String[] getAliases(String name);
}
```

BeanFactory在spring中有一个常用的默认实现DefaultListableBeanFactory

![beanfactory.png](/images/spring/beanfactory.png)

## BeanFactory

### 基本示例

对象bean的定位(Resource)、解析(XmlBeanDefinitionReader)、注册(BeanDefinitionReaderUtils.registerBeanDefinition)都通过BeanFactory来完成。下面以spring中BeanFactory的实现DefaultListableBeanFactory来做一个简单示例，通过xml文件配置的方式来管理对象bean：

```java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(new ClassPathResource("bean-factory.xml"));
MyBean bean = factory.getBean(MyBean.class);
System.out.println(bean.toString());
```

```java
public class MyBean {
    private String name;
    private Integer age;

    //getters,setters,toString...
}
```

```xml
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

### 源码解析

上面示例的核心逻辑就在下面：

```java
// 加载xml配置文件中的信息
// 使用的是DOM的方式解析XML
XmlBeanDefinitionReader#doLoadDocument
// 使用BeanDefinitionDocumentReader解析出Document对象
XmlBeanDefinitionReader#registerBeanDefinitions
DefaultBeanDefinitionDocumentReader#registerBeanDefinitions#doRegisterBeanDefinitions
DefaultBeanDefinitionDocumentReader#parseBeanDefinitions
//【核心方法】解析XML文件中spring默认namespace的XML节点
DefaultBeanDefinitionDocumentReader#parseDefaultElement
//*************************************************************
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
    if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
        importBeanDefinitionResource(ele);
    }
    else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
        processAliasRegistration(ele);
    }
    else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
        //这里是<bean>节点的处理逻辑
        processBeanDefinition(ele, delegate);
    }
    else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
        // recurse
        doRegisterBeanDefinitions(ele);
    }
}
//*************************************************************

//解析BeanDefinition，放入一个holder中
DefaultBeanDefinitionDocumentReader#processBeanDefinition
BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
BeanDefinitionParserDelegate#parseBeanDefinitionElement
BeanDefinitionParserDelegate#createBeanDefinition
//【创建BeanDefinition】
BeanDefinitionReaderUtils#createBeanDefinition
BeanDefinitionParserDelegate#parseBeanDefinitionAttributes
//...针对BeanDefinition的其它操作
BeanDefinitionParserDelegate#parsePropertyElements


//【注册BeanDefinition到spring容器中】
//回到DefaultBeanDefinitionDocumentReader#processBeanDefinition
//其实就是注册到DefaultListableBeanFactory的ConcurrentHashMap中
BeanDefinitionReaderUtils#registerBeanDefinition
```

经过上面的复杂步骤，BeanDefinition就被注册到spring容器里面了，但这个时候bean还没有实例化，真正实例化是在DefaultListableBeanFactory.getBean方法中

```java
DefaultListableBeanFactory#getBean
DefaultListableBeanFactory#resolveNamedBean
AbstractBeanFactory#getBean#doGetBean
//默认bean是单例的
AbstractBeanFactory#createBean
AbstractAutowireCapableBeanFactory#createBean
//在这里调用了InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation
AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation
AbstractAutowireCapableBeanFactory#doCreateBean
AbstractAutowireCapableBeanFactory#createBeanInstance
AbstractAutowireCapableBeanFactory#instantiateBean
SimpleInstantiationStrategy#instantiate
//【通过反射实例化bean】
BeanUtils.instantiateClass
//返回AbstractAutowireCapableBeanFactory#doCreateBean
//设置bean的属性，并且在这里调用了InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation/postProcessPropertyValues
AbstractAutowireCapableBeanFactory#populateBean
//【实例化之后进行初始化】
//这个方法也比较重要，在这个方法里面调用了invokeAwareMethods/applyBeanPostProcessorsBeforeInitialization/invokeInitMethods/applyBeanPostProcessorsAfterInitialization
AbstractAutowireCapableBeanFactory#initializeBean
```

## ApplicationContext

这种直接使用BeanFactory的方式比较原始，且使用不方便，实际开发中会使用BeanFactory的更高级的接口ApplicationContext以及其具体的实现类，ApplicationContext继承了BeanFactory的同时还提供了其它功能：

* ClassPathXmlApplicationContext/FileSystemXmlApplicationContext

* AnnotationConfigApplicationContext

* XmlWebApplicationContext

这两个实现类在new的时候都调用了`AbstractApplicationContext.refresh`这个spring容器最重要的方法。

### AbstractApplicationContext.refresh

这个方法包含了bean的整个生命周期，主要做的事情如下：

![bean-lifecycle.jpg](/images/spring/bean-lifecycle.jpg)

#### invokeBeanFactoryPostProcessors

##### @Configuration加载过程

核心是ConfigurationClassPostProcessor这个类，它是Spring-Context包提供的内置的BeanFactoryPostProcessor类，从refresh方法开始，跟踪一下一个配置类是如何被加载的。

![ConfigurationClassPostProcessor.jpg](/images/spring/ConfigurationClassPostProcessor.jpg)

```java
AbstractApplicationContext#refresh
AbstractApplicationContext#invokeBeanFactoryPostProcessors
PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors
BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry
ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry
ConfigurationClassPostProcessor#processConfigBeanDefinitions
// 下面这个方法是判断配置类是full模式还是lite模式
// 【full模式】：就是标记@Configuration的类，里面的@Bean都是交给容器管理的，默认是单例的
// 【lite模式】：就是标记@Component,@ComponentScan,@Import,@ImportResource的类
// 与完整的@Configuration不同，lite @Bean方法不能声明bean之间的依赖关系，【并且不会被CGLIB代理，**不是单例的**】
//如果bean标记了@Configuration,@Component,@ComponentScan,@Import,@ImportResource会被加入到`List<BeanDefinitionHolder> configCandidates`集合，都当做配置类来处理
ConfigurationClassUtils#checkConfigurationClassCandidate
//回到ConfigurationClassPostProcessor类
// 【核心方法】，这个方法里面处理了ImportSelector的实现类
ConfigurationClassParser#parse

// ************************************************************************
public void parse(Set<BeanDefinitionHolder> configCandidates) {
    this.deferredImportSelectors = new LinkedList<DeferredImportSelectorHolder>();

    for (BeanDefinitionHolder holder : configCandidates) {
        BeanDefinition bd = holder.getBeanDefinition();
        try {
            if (bd instanceof AnnotatedBeanDefinition) {
                parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
            }
            else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
                parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
            }
            else {
                parse(bd.getBeanClassName(), holder.getBeanName());
            }
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                    "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
        }
    }

    // 最后才处理DeferredImportSelector的实现类
    processDeferredImportSelectors();
}
// ************************************************************************

// 处理顺序：内部类->@PropertySources->@ComponentScan->@Import->@ImportResource
// 这里@ComponentScan扫描到的bean会直接注册到容器的BeanDefinition列表里面，其它还没有注册
ConfigurationClassParser#parse#doProcessConfigurationClass

// 回到ConfigurationClassParser#processConfigBeanDefinitions方法，parse方法执行之后会加载BeanDefinition
// 这里会遍历所有的@Configuration配置类，顺序就是processConfigBeanDefinitions方法中根据配置类按照Ordered或者@Order排序后的顺序
// 针对每个配置类，内部加载BeanDefinition的顺序为：@Import->@Bean->@ImportResource
ConfigurationClassBeanDefinitionReader#loadBeanDefinitions

// ConfigurationClassPostProcessor实现了BeanFactoryPostProcessor
// 返回Bean的生命周期方法AbstractApplicationContext#refresh#invokeBeanFactoryPostProcessors->PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors
// 在BeanDefinition都注册完成之后执行实现方法postProcessBeanFactory，在这个方法里面对full模式的配置类进行了加强，目的是处理bean的依赖关系
ConfigurationClassPostProcessor#postProcessBeanFactory
ConfigurationClassPostProcessor#enhanceConfigurationClasses
```

###### @Import

这个annotation一般放在@Configuration配置类上，一般导入以下3个实现类

* ImportSelector

在@Configuration中所有的bean处理之前执行

* DeferredImportSelector

  * AutoConfigurationImportSelector
  * EnableCircuitBreakerImportSelector

  > 在@Configuration中所有的bean处理之后执行，SpringBoot的自动配置EnableAutoConfigurationImportSelector(老版本)/AutoConfigurationImportSelector(新版本)就是使用的这种方式，自动配置必须在自定义配置之后执行，可以通过实现Ordered接口或者打上Order标签来制定顺序。

* ImportBeanDefinitionRegistrar

  * AspectJAutoProxyRegistrar
  * AutoProxyRegistrar

> 这三个类只能配合@Import使用，不能单独通过@Bean加载到容器，但是@Import可以独立使用导入普通的bean

如果配置类加了@Import，在`ConfigurationClassParser#parse#doProcessConfigurationClass#processImports`中进行了处理，`DeferredImportSelector`、`ImportBeanDefinitionRegistrar`先保存在内存变量中，在后面的`ConfigurationClassParser#processDeferredImportSelectors`中最后才进行处理

#### @Autowired

```java
AbstractApplicationContext.refresh
AbstractApplicationContext.finishBeanFactoryInitialization
DefaultListableBeanFactory.preInstantiateSingletons
AbstractBeanFactory.getBean
AbstractBeanFactory.doGetBean
AbstractAutowireCapableBeanFactory.createBean
AbstractAutowireCapableBeanFactory.doCreateBean
AbstractAutowireCapableBeanFactory.createBeanInstance
AbstractAutowireCapableBeanFactory.applyMergedBeanDefinitionPostProcessors
//*********************************************************
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
        if (bp instanceof MergedBeanDefinitionPostProcessor) {
            MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
            bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
        }
    }
}
//*********************************************************
```

##### AutowiredAnnotationBeanPostProcessor

```java
public class AutowiredAnnotationBeanPostProcessor extends InstantiationAwareBeanPostProcessorAdapter
    implements MergedBeanDefinitionPostProcessor, PriorityOrdered, BeanFactoryAware {}
```

AutowiredAnnotationBeanPostProcessor实现了MergedBeanDefinitionPostProcessor接口，postProcessMergedBeanDefinition()方法在上面的步骤里面被调用

```java
AutowiredAnnotationBeanPostProcessor.postProcessMergedBeanDefinition
AutowiredAnnotationBeanPostProcessor.findAutowiringMetadata
AutowiredAnnotationBeanPostProcessor.buildAutowiringMetadata
//这里通过反射的方式遍历所有的filed和method，检查是否打了@Autowired、@Value标签
//得到AutowiredFieldElement、AutowiredMethodElement的集合，最后都放入到LinkedList<InjectionMetadata.InjectedElement>链表里面
ReflectionUtils.doWithLocalFields
ReflectionUtils.doWithLocalMethods
```

**实际注入对象的时机**

```java
//回到AbstractAutowireCapableBeanFactory.populateBean
AbstractAutowireCapableBeanFactory.populateBean
//根据名称或者类型进行注入
AutowiredAnnotationBeanPostProcessor.autowireByName
AutowiredAnnotationBeanPostProcessor.autowireByType

AbstractAutowireCapableBeanFactory.postProcessPropertyValues
AutowiredAnnotationBeanPostProcessor.postProcessPropertyValues
AutowiredAnnotationBeanPostProcessor.postProcessProperties
//*********************************************************
for (BeanPostProcessor bp : getBeanPostProcessors()) {
    if (bp instanceof InstantiationAwareBeanPostProcessor) {
        InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
        PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
        if (pvsToUse == null) {
            if (filteredPds == null) {
                filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
            }
            //AutowiredAnnotationBeanPostProcessor是InstantiationAwareBeanPostProcessor的子类，所以postProcessPropertyValues方法会被调用
            pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
            if (pvsToUse == null) {
                return;
            }
        }
        pvs = pvsToUse;
    }
}


@Override
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
    InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
    try {
        metadata.inject(bean, beanName, pvs);
    }
    catch (BeanCreationException ex) {
        throw ex;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
    }
    return pvs;
}
//*********************************************************

//通过InjectionMetadata.inject->InjectElement.inject
//实际执行的是AutowiredFieldElement.inject和AutowiredMethodElement.inject
```

## 问题

### bean循环依赖问题

* 通过构造函数的方式循环依赖，不支持

* 通过setter的方式循环依赖，默认单例，支持

* 通过setter的方式循环依赖，prototype多例，不支持

## 参考

[Spring源码-IOC容器(六)-bean的循环依赖](https://my.oschina.net/u/2377110/blog/979226)

[@Import、DeferredImportSelector、ImportBeanDefinitionRegistrar的使用](https://blog.csdn.net/f641385712/article/details/88554592)

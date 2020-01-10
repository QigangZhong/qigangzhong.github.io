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

## BeanFactory

## ApplicationContext

### refresh

#### invokeBeanFactoryPostProcessors

##### @Configuration加载过程

核心是ConfigurationClassPostProcessor这个类，它是Spring-Context包提供的内置的BeanFactoryPostProcessor类

![ConfigurationClassPostProcessor.jpg](/images/spring/ConfigurationClassPostProcessor.jpg)

```java
AbstractApplicationContext#refresh
AbstractApplicationContext#invokeBeanFactoryPostProcessors
PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors
BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry
ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry
ConfigurationClassPostProcessor#processConfigBeanDefinitions
// 下面这个方法是判断配置类是full模式还是lite模式
// full模式：就是标记@Configuration的类，里面的@Bean都是交给容器管理的，默认是单例的
// lite模式：就是标记@Component,@ComponentScan,@Import,@ImportResource的类
ConfigurationClassUtils#checkConfigurationClassCandidate
// 【核心方法】，这个方法里面处理了ImportSelect的实现类
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

  > 在@Configuration中所有的bean处理之后执行，SpringBoot的自动配置EnableAutoConfigurationImportSelector就是使用的这种方式，自动配置必须在自定义配置之后执行，可以通过实现Ordered接口或者打上Order标签来制定顺序。

* ImportBeanDefinitionRegistrar

  * AspectJAutoProxyRegistrar
  * AutoProxyRegistrar

> 这三个类只能配合@Import使用，不能单独通过@Bean加载到容器，但是@Import可以独立使用导入普通的bean

如果配置类加了@Import，在`ConfigurationClassParser#parse#doProcessConfigurationClass#processImports`中进行了处理，`DeferredImportSelector`、`ImportBeanDefinitionRegistrar`先保存在内存变量中，在后面的`ConfigurationClassParser#processDeferredImportSelectors`中最后才进行处理

## 参考

[Spring源码-IOC容器(六)-bean的循环依赖](https://my.oschina.net/u/2377110/blog/979226)

[@Import、DeferredImportSelector、ImportBeanDefinitionRegistrar的使用](https://blog.csdn.net/f641385712/article/details/88554592)

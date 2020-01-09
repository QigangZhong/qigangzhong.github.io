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

## @Import

一般放在@Configuration配置类上，一般导入以下3个实现类，而且这三个类只能配合@Import使用，不能单独通过@Bean加载到容器

### ImportSelector

在@Configuration中所有的bean处理之前执行

### DeferredImportSelector

在@Configuration中所有的bean处理之后执行，SpringBoot的自动配置EnableAutoConfigurationImportSelector就是使用的这种方式，自动配置必须在自定义配置之后执行

### ImportBeanDefinitionRegistrar


## 参考

[Spring源码-IOC容器(六)-bean的循环依赖](https://my.oschina.net/u/2377110/blog/979226)

[@Import、DeferredImportSelector、ImportBeanDefinitionRegistrar的使用](https://blog.csdn.net/f641385712/article/details/88554592)

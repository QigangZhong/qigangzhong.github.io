---
layout: post
title:  "spring中的各种aware接口"
categories: spring
tags: spring
author: 网络
---

* content
{:toc}











## 介绍

* ApplicationContextAware: 获得ApplicationContext对象,可以用来获取所有Bean definition的名字。
* BeanFactoryAware: 获得BeanFactory对象，可以用来检测Bean的作用域。
* BeanNameAware: 获得Bean在配置文件中定义的名字。
* ResourceLoaderAware: 获得ResourceLoader对象，可以获得classpath中某个文件。
* BeanClassLoaderAware: 获取当前bean的类加载器。
* ServletContextAware: 在一个MVC应用中可以获取ServletContext对象，可以读取context中的参数。
* ServletConfigAware: 在一个MVC应用中可以获取ServletConfig对象，可以读取config中的参数。
* EnvironmentAware: 获取Environment对象，通过这个对象可以获取系统变量信息。
* MessageSourceAware: 消息国际化

## 参考

[spring的aware们](https://www.jianshu.com/p/bd565eb777f7)

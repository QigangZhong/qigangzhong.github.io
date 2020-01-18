---
layout: post
title:  "springboot配置"
categories: spring
tags: spring
author: 网络
---

* content
{:toc}











## spring中访问配置的几种方式

* Environment#getProperty

环境变量包括：system properties、-D参数、application.properties(.yml)、通过@PropertySource引入的配置文件配置

* properties.get("property")

直接读取配置文件信息到Properties对象实例中进行访问

* @Value

* @ConfigurationProperties

## 参考

---
layout: post
title:  "序列化"
categories: serialization
tags:  serialization
author: 网络
---

* content
{:toc}

总结java序列化相关的知识




hession2

Thrift

protobuf

jackson

gson

msgpack


对性能敏感，对开发体验要求不高的内部系统。
　　thrift/protobuf

对开发体验敏感，性能有要求的内外部系统。
　　hessian2

对序列化后的数据要求有良好的可读性
　　jackson/gson/xml


### transient

transient阻止实例中那些用此关键字声明的变量序列化

## 问题

### Serializer接口及serialVersionUID的作用

[java类中serialVersionUID的作用](https://blog.csdn.net/u014750606/article/details/80040130)

[深入分析序列化和反序列化原理，终于知道serialVersionUID到底有什么用了](https://blog.csdn.net/zwx900102/article/details/107675524)

## 参考

[在Java中如何使用transient](http://www.importnew.com/12611.html)
---
title: Spring AOP（三）：CgLib代理类详解（施工中）
date: 2022-05-04 19:22:12 +0800
categories: [源码阅读, Spring-AOP]
tags: [Java, Spring, AOP, CgLib]
---

## 前言
在上篇文章里将整个AOP加载和执行梳理了一遍，也埋下了个伏笔，CgLibAopProxy对象是通过ProxyFactory创建的，同时后者还调用了getProxy方法获取代理对象实例。
![1-1 breakpoint_getProxy](/assets/img/20220425/breakpoint_getProxy.png)_1-1 breakpoint getProxy_

同时我们在debug时，也经常能看到类似下图的堆栈：
![1-2 breakpoint_cglib](/assets/img/20220504/breakpoint_cglib.png)_1-2 breakpoint cglib_

这是因为`Autowired`注入的对象实际上是代理对象。Spring通过动态代理，生成了代理对象，从而实现AOP功能。

这篇文章就来详细看看代理类中究竟有些什么，Spring又是如何创建这个代理类的。

## 获取运行时的代理类


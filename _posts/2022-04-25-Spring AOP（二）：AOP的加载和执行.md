---
title: Spring AOP（二）：AOP的加载和执行（施工中）
date: 2022-04-25 19:05:16 +0800
categories: [源码阅读]
tags: [Java, Spring, AOP, Transation]
---
[toc]

## 前言
在上篇文章里我们梳理了AOP事务的执行流程，但对于AOP还有许多盲区没有探究。

`List<Object> chain`
![1-1 proxy_intercept](/assets/img/20220425/proxy_intercept.png)_1-1_

例如在堆栈调用链刚进入intercept方法时，代理类从某个池中获取了切面的执行链，就能列出一系列问题：
- `advised`对象是何时，从哪里来的，包含哪些内容？
- `chain`是如何被筛选出来的？
- AOP执行顺序是如何保证的

这一章我们就来梳理AOP逻辑。

## AOP加载过程

当完全不理解一段源码的逻辑时，最需要的就是找到分析的切入点，从图1-1我们可以看得出`advised`承担了缓存切面处理器`Advisor`以及筛选当前所需处理器的功能，是个很重要的对象，因此我先选择`advised`对象来分析，

### AdvisedSupport对象分析

首先跳转定义，可以看到`advised`是`AdvisedSupport`的实例，查看下运行时参数：
![2-1 breakpoint advised value](/assets/img/20220425/breakpoint_advised_value.png)_2-1 breakpoint advised value_
虽然大部分参数都不明白意义，但可以看到对象包含了代理方法和切面处理器，说明哪些方法有切面、哪些切面会可能会被执行，是在启动时就决定的。

#### getInterceptorsAndDynamicInterceptionAdvice
接着看看`getInterceptorsAndDynamicInterceptionAdvice`方法

### advised加载
还是从图1-1的线索开始，追踪`advised`的加载流程，由于`DynamicAdvisedInterceptor`是`CglibAopProxy`的内部类，并且在Proxy方法中初始化，因此直接给CglibAopProxy的advised变量打上断点。
![2-1 breakpoint advised](/assets/img/20220425/breakpoint_advised.png)_2-1 breakpoint advised_

然后启动项目，跟踪堆栈信息。

### 创建AopProxy实例

首先能看到在CglibAopProxy构造函数中，AdvisedSupport
![2-2 breakpoint_proxyInstance](/assets/img/20220425/breakpoint_proxyInstance.png)_2-2 breakpoint proxyInstance_



### advised初始化

## AOP执行过程

### 代理类调用

### getInterceptors
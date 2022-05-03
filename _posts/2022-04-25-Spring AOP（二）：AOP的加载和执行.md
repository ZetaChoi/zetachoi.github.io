---
title: Spring AOP（二）：AOP的加载和执行（施工中）
date: 2022-04-25 19:05:16 +0800
categories: [源码阅读, Spring-AOP]
tags: [Java, Spring, AOP, Transation]
---

## 前言
在上篇文章里我们梳理了AOP事务的执行流程，但对于AOP还有许多盲区没有探究。

例如前面调试跟踪到的拦截器方法：
![1-1 proxy_intercept](/assets/img/20220425/proxy_intercept.png)_1-1_

在堆栈调用链刚进入intercept方法时，代理类从某个池中获取了切面的执行链，就能列出一系列问题：
- `advised`对象是何时，从哪里来的，包含哪些内容？
- `chain`是如何被筛选出来的？
- AOP执行顺序是如何保证的

这一章我们就来梳理AOP逻辑。

## 揭示InterceptorChain获取过程

当完全不理解一段源码的逻辑时，最需要的就是找到分析的切入点，从图1-1我们可以看得出`advised`承担了缓存切面处理器`Advisor`以及筛选当前所需处理器的功能，是个很重要的对象，因此我先选择`advised`对象来分析，

### AdvisedSupport的作用

首先跳转定义，可以看到`advised`是`AdvisedSupport`的实例，查看下运行时参数：
![2-1 breakpoint advised value](/assets/img/20220425/breakpoint_advised_value.png)_2-1 breakpoint advised value_

虽然大部分参数都不明白意义，但可以看到包含了代理方法的缓存和切面处理器，说明能被切入的方法是在启动时就缓存好的。

### AdvisedSupport - getInterceptorsAndDynamicInterceptionAdvice
从图1-1的调用开始，进入`getInterceptorsAndDynamicInterceptionAdvice`看看方法逻辑，代码比较长，但阅读源码时一般只要留意入参的使用过程，结合源码注释，就能很快梳理出逻辑，推荐阅读本文的同时自己debug加深理解。
``` java
public class DefaultAdvisorChainFactory implements AdvisorChainFactory, Serializable {
    @Override
    public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Advised config, Method method, @Nullable Class<?> targetClass) {

        AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
        // 1 获取所有已注册的切面处理器
        Advisor[] advisors = config.getAdvisors();
        List<Object> interceptorList = new ArrayList<>(advisors.length);
        Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
        Boolean hasIntroductions = null;

        // 2 遍历获取当前方法适用的切面处理器
        for (Advisor advisor : advisors) {
            // 2.1 PointcutAdvisor类型的处理器，自定义切面中最常见的类型
            if (advisor instanceof PointcutAdvisor) {
                PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
                // 2.1.1 先判断处理器是否能处理当前类
                if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
                    MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
                    boolean match;
                    // 2.1.2 再判断是否能处理当前方法
                    if (mm instanceof IntroductionAwareMethodMatcher) {
                        if (hasIntroductions == null) {
                            hasIntroductions = hasMatchingIntroductions(advisors, actualClass);
                        }
                        // 2.1.2.1 Introductions的matches
                        match = ((IntroductionAwareMethodMatcher) mm).matches(method, actualClass, hasIntroductions);
                    }
                    else {
                        // 2.1.2.1 普通的matches
                        match = mm.matches(method, actualClass);
                    }
                    // 2.1.3 如果能处理就放入执行链列表
                    if (match) {
                        MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
                        if (mm.isRuntime()) {
                            // Creating a new object instance in the getInterceptors() method
                            // isn't a problem as we normally cache created chains.
                            for (MethodInterceptor interceptor : interceptors) {
                                interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
                            }
                        }
                        else {
                            interceptorList.addAll(Arrays.asList(interceptors));
                        }
                    }
                }
            }
            // 2.2 IntroductionAdvisor类型的处理器
            else if (advisor instanceof IntroductionAdvisor) {
                IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
                // 2.2.1 IntroductionAdvisor只作用于类，所以这里只判断能否处理当前类
                if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
                    Interceptor[] interceptors = registry.getInterceptors(advisor);
                    interceptorList.addAll(Arrays.asList(interceptors));
                }
            }
            // 2.2 其他类型的处理器，直接加入执行链，等处理器执行时自行判断可用性。
            else {
                Interceptor[] interceptors = registry.getInterceptors(advisor);
                interceptorList.addAll(Arrays.asList(interceptors));
            }
        }

        return interceptorList;
    }
}
```
这里遇到了三种类型的Advisor，查阅资料后简单列举下区别：
- PointcutAdvisor：可以作用在方法上的Advisor，事务切面就是这种类型。
- IntroductionAdvisor：作用于类的Advisor。
- 其他Advisor：暂时不知道具体有哪些。

限于篇幅这里就不继续展开说明matches方法和registry.getInterceptors了，有兴趣的可以继续跟下去看看，简单说说他们功能：
- matches决定了当前的方法和类能否被处理，是我们在声明切面时实现，而后由Spring包装而成的。具体匹配方法多种多样，例如在事务切面中，是判断方法上是否存在@Transation注解。
- registry包含了一些默认的切点，registry.getInterceptors其实是对当前advisor的再次包装，除了获取到当前Advisor的Interceptor，也会判断是否需要把这些默认切点拦截器加入到列表中。

### 阶段总结
至此基本了解getInterceptorsAndDynamicInterceptionAdvice的功能了，同时也了解了Advisor：每一个切面都有一个代理拦截器Interceptor，和一系列的mather来决定是否需要执行该Interceptor，Advisor就是Spring针对每一个切面创建的实例，用于管理Interceptor、mather以及其他相关的属性。

回过头来，Advised对象包含了所有已知的Advisor，为了了解Advisor如何被创建，下一步是查看advised的加载流程。

## 追踪advised，理解AOP初始化
还是从图1-1的线索开始追踪`advised`，寻找初始化advised对象的地方。由于`DynamicAdvisedInterceptor`是`CglibAopProxy`的内部类，并且在Proxy方法中初始化，因此直接给CglibAopProxy的advised变量打上断点，然后启动项目，跟踪堆栈信息：
![3-1 breakpoint advised](/assets/img/20220425/breakpoint_advised.png)_3-1 breakpoint advised_

首先能看到在CglibAopProxy构造函数中，AdvisedSupport被赋值给了advised
![3-2 breakpoint_proxyInstance](/assets/img/20220425/breakpoint_proxyInstance.png)_3-2 breakpoint proxyInstance_

往前跟踪堆栈，ProxyFactory创建Proxy实例时，通过判断当前对象是否实现了接口，决定采用CgLib还是Jdk代理，这跟我们通常印象是一致的：
![3-3 breakpoint_createAopProxy](/assets/img/20220425/breakpoint_createAopProxy.png)_3-3 breakpoint createAopProxy_

再往上，这一步非常关键，虽然前面跟踪的都是AopProxy构造方法，但上游调用的实际上是getProxy，ProxyFactory同时完成了创建AopProxy和湖区代理对象(getProxy)的功能。getProxy方法非常重要，请先记住它，我会在下一篇文章展开讨论。这里只需要知道，上游获取的实际上是目标类的代理对象：
![3-4 breakpoint_getProxy](/assets/img/20220425/breakpoint_getProxy.png)_3-4 breakpoint getProxy_

继续跟踪，终于找到了创建advisors对象的地方，但在阅读buildAdvisor源码后可以发现，buildAdvisor只是获取了Interceptor实例并创建与之匹配的Advisor对象，没有什么特别之处。此时阅读源码的重心就从"了解advisor构造过程"，转移到"了解Interceptor构造过程"上了。
![3-5 breakpoint_creator_createProxy](/assets/img/20220425/breakpoint_creator_createProxy.png)_3-5 breakpoint createProxy_

前面这部分堆栈需要连着看，可以发现一切的源头`doCreateBean`，Spring在创建完Bean实例后，进行注入成员变量等初始化工作，并进行`applyBeanPostProcessors`,PostProcessors可以理解为包装模式的包装器，这其中就包含了支撑AOP功能的`AbstractAutoProxyCreator`。AbstractAutoProxyCreator依据具体情况(是否存在Interceptor)决定是否要给Bean添加代理对象，并且将这个代理对象作为结果返回，BeanFactory最终创建并缓存的Bean实例，也会替换为这个代理对象。
![3-6 breakpoint_wrapIfNecessary](/assets/img/20220425/breakpoint_wrapIfNecessary.png)_3-6 breakpoint wrapIfNecessary_

至此，整个AOP初始化的过程就分析完了，

## 追踪Interceptor，理解切面加载和排序

## 总结

最后，我把上面的逻辑整理成流程图，整个AOP创建和使用的过程就一目了然了。
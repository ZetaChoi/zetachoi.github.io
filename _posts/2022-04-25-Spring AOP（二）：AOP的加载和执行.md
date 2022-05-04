---
title: Spring AOP（二）：AOP的加载和执行
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

当完全不理解一段源码的逻辑时，最需要的就是找到分析的切入点，从图1-1我们可以看得出`advised`承担了缓存切面监听器`Advisor`以及筛选当前所需监听器的功能，是个很重要的对象，因此我先选择`advised`对象来分析。

### AdvisedSupport的作用

首先跳转定义，可以看到`advised`是`AdvisedSupport`的实例，查看下运行时参数：
![2-1 breakpoint advised value](/assets/img/20220425/breakpoint_advised_value.png)_2-1 breakpoint advised value_

虽然大部分参数都不明白意义，但可以看到包含了代理方法的缓存和切面监听器，说明能被切入的方法是在启动时就缓存好的。

### getInterceptorsAndDynamicInterceptionAdvice
从图1-1的调用开始，进入`getInterceptorsAndDynamicInterceptionAdvice`看看方法逻辑，代码比较长，但阅读源码时一般只要留意入参的使用过程，结合源码注释，就能很快梳理出逻辑，推荐阅读本文的同时自己debug加深理解。
``` java
public class DefaultAdvisorChainFactory implements AdvisorChainFactory, Serializable {
    @Override
    public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Advised config, Method method, @Nullable Class<?> targetClass) {

        AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
        // 1 获取所有已注册的切面监听器
        Advisor[] advisors = config.getAdvisors();
        List<Object> interceptorList = new ArrayList<>(advisors.length);
        Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
        Boolean hasIntroductions = null;

        // 2 遍历获取当前方法适用的切面监听器
        for (Advisor advisor : advisors) {
            // 2.1 PointcutAdvisor类型的监听器，自定义切面中最常见的类型
            if (advisor instanceof PointcutAdvisor) {
                PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
                // 2.1.1 先判断监听器是否能处理当前类
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
            // 2.2 IntroductionAdvisor类型的监听器
            else if (advisor instanceof IntroductionAdvisor) {
                IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
                // 2.2.1 IntroductionAdvisor只作用于类，所以这里只判断能否处理当前类
                if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
                    Interceptor[] interceptors = registry.getInterceptors(advisor);
                    interceptorList.addAll(Arrays.asList(interceptors));
                }
            }
            // 2.2 其他类型的监听器，直接加入执行链，等监听器执行时自行判断可用性。
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
- PointcutAdvisor：提供了class+方法的过滤规则，事务切面就是这种类型。
- IntroductionAdvisor：提供了class类型的过滤规则。
- 其他Advisor：其他规则的Advisor，可由用户自定义实现。

限于篇幅这里就不继续展开说明matches方法和registry.getInterceptors了，有兴趣的可以继续跟下去看看，简单说说他们功能：
- matches决定了当前的方法和类能否被处理，是我们在声明切面时实现，而后由Spring包装而成的。具体匹配方法多种多样，例如在事务切面中，是判断方法上是否存在@Transation注解。
- registry包含了一些默认的切点，registry.getInterceptors其实是对当前advisor的再次包装，除了获取到当前Advisor的Interceptor，也会判断是否需要把这些默认切点拦截器加入到列表中。

### 阶段总结
至此基本了解getInterceptorsAndDynamicInterceptionAdvice的功能了。同时也了解到：Advisor作为SpringAop的顶级接口，负责管理代理拦截器Interceptor、切面过滤器PointCut；AdvisedSupport则负责在运行时管理调度这些Advisor，筛选出当前适用的切面。

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

前面这部分堆栈需要连着看，可以发现一切的源头`doCreateBean`，Spring在创建完Bean实例后，进行注入成员变量等初始化工作，并进行`applyBeanPostProcessors`,PostProcessors可以理解为对bean进行特殊处理的包装器，这其中就包含了支撑AOP功能的`AbstractAutoProxyCreator`。AbstractAutoProxyCreator依据具体情况(是否存在Interceptor)决定是否要给Bean添加代理对象，并且将这个代理对象作为结果返回，BeanFactory最终创建并缓存的Bean实例，也会替换为这个代理对象。
![3-6 breakpoint_wrapIfNecessary](/assets/img/20220425/breakpoint_wrapIfNecessary.png)_3-6 breakpoint wrapIfNecessary_

至此，整个AOP初始化的过程就分析完了。

## 追踪Interceptor，理解切面加载和排序

了解完AOP的加载过程，继续研究Interceptor初始化，在图3-6中可以看出，Spring调用`getAdvicesAndAdvisorsForBean`，并依据Interceptor集合是否为空，决定是否创建代理对象。

由于`getAdvicesAndAdvisorsForBean`存在多种实现，在Variable列表中右键this，选择Jump To Type Source：
![4-1 breakpoint_jumpToTypeSource](/assets/img/20220425/breakpoint_jumpToTypeSource.png)_4-1 breakpoint jumpToTypeSource_

点击`Ctrl+F12`搜索getAdvicesAndAdvisorsForBean，点击跳转即可找到当前流程中正确的实现：
![4-2 breakpoint_search](/assets/img/20220425/breakpoint_search.png)_4-2 breakpoint search_

继续跟踪，找到了`findEligibleAdvisors`方法
```java
public abstract class AbstractAdvisorAutoProxyCreator extends AbstractAutoProxyCreator {
    protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
        List<Advisor> candidateAdvisors = findCandidateAdvisors();
        List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
        extendAdvisors(eligibleAdvisors);
        if (!eligibleAdvisors.isEmpty()) {
            eligibleAdvisors = sortAdvisors(eligibleAdvisors);
        }
        return eligibleAdvisors;
    }
}
```

### findCandidateAdvisors - 获取所有的Advisor
一行行来看，首先是`findCandidateAdvisors`，值得注意的是，`AnnotationAwareAspectJAutoProxyCreator`是`AbstractAdvisorAutoProxyCreator`的一种实现，使得不光可以用实现Advisor接口的方式创建切面(原生的SpringAOP)，也可以用Aspectj的API更简便地声明切面。

`AnnotationAwareAspectJAutoProxyCreator`重写了`findCandidateAdvisors`方法，并且会在代码中存在Aspectj类型切面时生效：
```java
public class AnnotationAwareAspectJAutoProxyCreator extends AspectJAwareAdvisorAutoProxyCreator {
    @Override
    protected List<Advisor> findCandidateAdvisors() {
        // Add all the Spring advisors found according to superclass rules.
        List<Advisor> advisors = super.findCandidateAdvisors();
        // Build Advisors for all AspectJ aspects in the bean factory.
        if (this.aspectJAdvisorsBuilder != null) {
            advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
        }
        return advisors;
    }
}
```

原生的SpringAOP通过BeanFactoryAdvisorRetrievalHelper在BeanFactory中查找并获取所有实现了Advisor接口的类，若该类没有创建实例，也会在此时创建：
```java
public class BeanFactoryAdvisorRetrievalHelper {
    public List<Advisor> findAdvisorBeans() { 
        // Determine list of advisor bean names, if not cached already.
        String[] advisorNames = this.cachedAdvisorBeanNames;
        if (advisorNames == null) {
            // Do not initialize FactoryBeans here: We need to leave all regular beans
            // uninitialized to let the auto-proxy creator apply to them!
            // 通过设置allowEagerInit为false，避免获取时过早将对象实例化，早将实例化会导致跳过参数注入的环节
            advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                    this.beanFactory, Advisor.class, true, false);
            this.cachedAdvisorBeanNames = advisorNames;
        }
        if (advisorNames.length == 0) {
            return new ArrayList<>();
        }

        List<Advisor> advisors = new ArrayList<>();
        for (String name : advisorNames) {
            if (isEligibleBean(name)) {
                if (this.beanFactory.isCurrentlyInCreation(name)) {
                    if (logger.isTraceEnabled()) {
                        logger.trace("Skipping currently created advisor '" + name + "'");
                    }
                }
                else {
                    try {
                        advisors.add(this.beanFactory.getBean(name, Advisor.class));
                    }
                    catch (BeanCreationException ex) {
                        Throwable rootCause = ex.getMostSpecificCause();
                        if (rootCause instanceof BeanCurrentlyInCreationException) {
                            BeanCreationException bce = (BeanCreationException) rootCause;
                            String bceBeanName = bce.getBeanName();
                            if (bceBeanName != null && this.beanFactory.isCurrentlyInCreation(bceBeanName)) {
                                if (logger.isTraceEnabled()) {
                                    logger.trace("Skipping advisor '" + name +
                                            "' with dependency on currently created bean: " + ex.getMessage());
                                }
                                // Ignore: indicates a reference back to the bean we're trying to advise.
                                // We want to find advisors other than the currently created bean itself.
                                continue;
                            }
                        }
                        throw ex;
                    }
                }
            }
        }
        return advisors;
    }
}
```

接着，aspectJAdvisorsBuilder的查找方法稍微麻烦一些，他取出了所有的JavaBean并依次检查两个规则
1. 类上带了@Aspect注解
2. 没有使用AspectJ创建静态代理类，这是为了避免重复创建代理类，也是Spring规范性的要求。

```java
public class BeanFactoryAspectJAdvisorsBuilder {
    public List<Advisor> buildAspectJAdvisors() {
        List<String> aspectNames = this.aspectBeanNames;

        if (aspectNames == null) {
            synchronized (this) {
                aspectNames = this.aspectBeanNames;
                if (aspectNames == null) {
                    List<Advisor> advisors = new ArrayList<>();
                    aspectNames = new ArrayList<>();
                    String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                            this.beanFactory, Object.class, true, false);
                    for (String beanName : beanNames) {
                        if (!isEligibleBean(beanName)) {
                            continue;
                        }
                        // We must be careful not to instantiate beans eagerly as in this case they
                        // would be cached by the Spring container but would not have been weaved.
                        Class<?> beanType = this.beanFactory.getType(beanName, false);
                        if (beanType == null) {
                            continue;
                        }
                        if (this.advisorFactory.isAspect(beanType)) {
                            aspectNames.add(beanName);
                            AspectMetadata amd = new AspectMetadata(beanType, beanName);
                            if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
                                MetadataAwareAspectInstanceFactory factory =
                                        new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
                                List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
                                if (this.beanFactory.isSingleton(beanName)) {
                                    this.advisorsCache.put(beanName, classAdvisors);
                                }
                                else {
                                    this.aspectFactoryCache.put(beanName, factory);
                                }
                                advisors.addAll(classAdvisors);
                            }
                            else {
                                // Per target or per this.
                                if (this.beanFactory.isSingleton(beanName)) {
                                    throw new IllegalArgumentException("Bean with name '" + beanName +
                                            "' is a singleton, but aspect instantiation model is not singleton");
                                }
                                MetadataAwareAspectInstanceFactory factory =
                                        new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
                                this.aspectFactoryCache.put(beanName, factory);
                                advisors.addAll(this.advisorFactory.getAdvisors(factory));
                            }
                        }
                    }
                    this.aspectBeanNames = aspectNames;
                    return advisors;
                }
            }
        }
    }
}
```

### findAdvisorsThatCanApply - 过滤符合条件的Advisor
`findAdvisorsThatCanApply`方法依次取出Advisor的PointCut进行匹配，过滤出符合当前类的Advisor
``` java
public abstract class AopUtils {
    public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
        if (advisor instanceof IntroductionAdvisor) {
            return ((IntroductionAdvisor) advisor).getClassFilter().matches(targetClass);
        }
        else if (advisor instanceof PointcutAdvisor) {
            PointcutAdvisor pca = (PointcutAdvisor) advisor;
            return canApply(pca.getPointcut(), targetClass, hasIntroductions);
        }
        else {
            // It doesn't have a pointcut so we assume it applies.
            return true;
        }
    }
}
```

### extendAdvisors - 通过其他方式获取Advisor

前面是通过`findCandidateAdvisors`方法从用户代码中获取Advisor，Spring也给Creator提供了一种途径来创建额外的Advisor，例如在`AspectJAwareAdvisorAutoProxyCreator`中就添加了`ExposeInvocationInterceptor`
```java
public abstract class AspectJProxyUtils {
    public static boolean makeAdvisorChainAspectJCapableIfNecessary(List<Advisor> advisors) {
        // Don't add advisors to an empty list; may indicate that proxying is just not required
        if (!advisors.isEmpty()) {
            boolean foundAspectJAdvice = false;
            for (Advisor advisor : advisors) {
                // Be careful not to get the Advice without a guard, as this might eagerly
                // instantiate a non-singleton AspectJ aspect...
                if (isAspectJAdvice(advisor)) {
                    foundAspectJAdvice = true;
                    break;
                }
            }
            if (foundAspectJAdvice && !advisors.contains(ExposeInvocationInterceptor.ADVISOR)) {
                advisors.add(0, ExposeInvocationInterceptor.ADVISOR);
                return true;
            }
        }
        return false;
    }
}
```

### sortAdvisors - 排序

先看代码：
```java
public class AspectJAwareAdvisorAutoProxyCreator extends AbstractAdvisorAutoProxyCreator {
    protected List<Advisor> sortAdvisors(List<Advisor> advisors) {
        List<PartiallyComparableAdvisorHolder> partiallyComparableAdvisors = new ArrayList<>(advisors.size());
        for (Advisor advisor : advisors) {
            partiallyComparableAdvisors.add(
                    new PartiallyComparableAdvisorHolder(advisor, DEFAULT_PRECEDENCE_COMPARATOR));
        }
        List<PartiallyComparableAdvisorHolder> sorted = PartialOrder.sort(partiallyComparableAdvisors);
        if (sorted != null) {
            List<Advisor> result = new ArrayList<>(advisors.size());
            for (PartiallyComparableAdvisorHolder pcAdvisor : sorted) {
                result.add(pcAdvisor.getAdvisor());
            }
            return result;
        }
        else {
            return super.sortAdvisors(advisors);
        }
    }
}
```

排序的代码比较简单，直接通过Advisor的order属性进行排序，值得注意的是，当两个Advicor的order相同时，队列中靠前的对象会被排到前列，虽然`BeanFactoryUtils.beanNamesForTypeIncludingAncestors`是无序的，但`AnnotationAwareAspectJAutoProxyCreator`的`findCandidateAdvisors`分两步加载原生SpringAOP和AspectJ，注定了原生SpringAOP优先级比AspectJ更高。

通过观察`BeanFactoryTransactionAttributeSourceAdvisor`，可以知道他是继承Advisor实现的切面，并且order是整形最大值，也就是在`before`中会最先执行。但如果同时又存在其他同类型的切面，就无法保证切面之间事务的有效性了。


## 总结

最后，把上面的逻辑整理成流程图，整个AOP创建和使用的过程就一目了然了。
![5-1 flow](/assets/img/20220425/flow.png)

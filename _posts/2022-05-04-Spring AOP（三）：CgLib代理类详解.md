---
title: Spring AOP（三）：CgLib代理类详解
date: 2022-05-04 19:22:12 +0800
categories: [源码阅读, Spring-AOP]
tags: [Java, Spring, AOP, CgLib]
---

## 前言
在上篇文章里将整个AOP加载和执行梳理了一遍，也埋下了个伏笔，CgLibAopProxy对象是通过ProxyFactory创建的，同时后者还调用了getProxy方法获取代理对象实例。
![1-1 breakpoint_getProxy](/assets/img/20220425/breakpoint_getProxy.png)_1-1 breakpoint getProxy_

我们在debug时，也经常能看到类变成`xxxx$$EnhancerBySpringCGLIB$$xxx`的形式：
![1-2 breakpoint_cglib](/assets/img/20220504/breakpoint_cglib.png)_1-2 breakpoint cglib_

这是因为`Autowired`注入的对象实际上是代理对象。Spring通过动态代理，生成了代理对象，从而实现AOP功能。

这篇文章就来详细看看代理类中究竟有些什么，Spring又是如何创建这个代理类的。

## 实现一个CgLib代理
要搞懂原理，首先要会用，第一步先手写一个CgLib代理。

引入pom依赖：
```xml
<dependency>
    <groupId>cglib</groupId>
    <artifactId>cglib</artifactId>
    <version>2.2.2</version>
</dependency>
```

写一个简单的测试类：
```java
class Person {
    public String name = "Zeta";

    public void whoAmI(){
        System.out.println("I'm " + name);
    }

    public void hello(){
        System.out.println("Hello!");
    }
}
```

定义一个方法拦截器，调用目标方法时，CgLib会代理到拦截器上。
```java
class PersonHelloInterceptor implements MethodInterceptor {

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {

        System.out.println("say hello...");
        return methodProxy.invokeSuper(o, objects);
    }
}
```

定义一个方法过滤器，`accept`方法返回值对应callbacks数组的下标
```java
class PersonCallbackFilter implements CallbackFilter {

    @Override
    public int accept(Method method) {
        if (method.getName().equals("whoAmI")) {
            return 1;
        }
        return 0;
    }
}
```


Enhancer是字节码增强器，通过它创建代理类并调用方法。
```java
public class TestMain {

    public static void main(String[] args) {
        System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "/temp");

        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(Person.class);
        enhancer.setCallbacks(new Callback[]{NoOp.INSTANCE, new PersonHelloInterceptor()});
        enhancer.setCallbackFilter(new PersonCallbackFilter());
        Person proxyPerson = (Person) enhancer.create();

        proxyPerson.hello();
        proxyPerson.whoAmI();
    }
}
```

编译并执行，可以看到执行的是被代理过的方法。
![2-1 cglib_demo](/assets/img/20220504/cglib_demo.png)_1-2 cglib demo_

## 代理类分析

通过添加`System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "/temp")`，可以指定CGLIB将代理类保存至指定路径。将class字节码反编译出来观察结构。

### 静态块
首先，代理类继承了原始类，定义了许多成员变量，并且在静态块中进行赋值，不难看出这些变量分别指代了原始方法和代理方法。
```java
static {
    CGLIB$STATICHOOK1();
}

static void CGLIB$STATICHOOK1() {
    CGLIB$THREAD_CALLBACKS = new ThreadLocal();
    CGLIB$emptyArgs = new Object[0];
    // 代理类的Class
    Class var0 = Class.forName("com.zeta.spring.demo.cglib.Person$$EnhancerByCGLIB$$20179872");
    // 原始类的Class
    Class var1;
    // 用反射工具获取原始方法
    CGLIB$whoAmI$1$Method = ReflectUtils.findMethods(new String[]{"whoAmI", "()V"}, (var1 = Class.forName("com.zeta.spring.demo.cglib.Person")).getDeclaredMethods())[0];
    // 创建代理方法执行器MethodProxy
    CGLIB$whoAmI$1$Proxy = MethodProxy.create(var1, var0, "()V", "whoAmI", "CGLIB$whoAmI$1");
}
```

### 代理方法调用
当通过代理类调用目标方法时，进入的是代理类重写的方法：
``` java
public final void whoAmI() {
    // 获取代理拦截器，这里获取的是下标为1的拦截器，也就是PersonHelloInterceptor
    MethodInterceptor var10000 = this.CGLIB$CALLBACK_1;
    if (var10000 == null) {
        CGLIB$BIND_CALLBACKS(this);
        var10000 = this.CGLIB$CALLBACK_1;
    }

    if (var10000 != null) {
        // 调用拦截器的方法
        var10000.intercept(this, CGLIB$whoAmI$1$Method, CGLIB$emptyArgs, CGLIB$whoAmI$1$Proxy);
    } else {
        super.whoAmI();
    }
}
```

原始的方法被包装，`methodProxy.invokeSuper`调用就是这个方法
```java
final void CGLIB$whoAmI$1() {
    super.whoAmI();
}
```

### CGLIB$BIND_CALLBACKS - 载入代理拦截器

CgLib为每个代理类都创建了静态方法`CGLIB$BIND_CALLBACKS`，用于获取目标的代理拦截器。可以看到CgLib优先从ThreadLocal中获取拦截器，其次才是初始化时定义的拦截器，说明在未调用任何代理方法前，仍有机会动态修改代理拦截器。
``` java
private static final void CGLIB$BIND_CALLBACKS(Object var0) {
    // this
    Person$$EnhancerByCGLIB$$20179872 var1 = (Person$$EnhancerByCGLIB$$20179872)var0;
    if (!var1.CGLIB$BOUND) {
        var1.CGLIB$BOUND = true;
        Object var10000 = CGLIB$THREAD_CALLBACKS.get();
        if (var10000 == null) {
            var10000 = CGLIB$STATIC_CALLBACKS;
            if (var10000 == null) {
                return;
            }
        }

        Callback[] var10001 = (Callback[])var10000;
        var1.CGLIB$CALLBACK_1 = (MethodInterceptor)((Callback[])var10000)[1];
        var1.CGLIB$CALLBACK_0 = (NoOp)var10001[0];
    }

}
```

### 其他

当然，CgLib的功能还不止这些，例如CgLib提供了一系列构造方法以适应并获取不同的代理实例，又如创建代理类的同时还会创建继承自FastClass的子类用来优化和代替反射调用，更多的细节就待读者深入研究了。

## Spring中的动态代理

从`getProxy`开始
```java
class CglibAopProxy implements AopProxy, Serializable {
    @Override
    public Object getProxy(@Nullable ClassLoader classLoader) {
        if (logger.isTraceEnabled()) {
            logger.trace("Creating CGLIB proxy: " + this.advised.getTargetSource());
        }

        try {
            Class<?> rootClass = this.advised.getTargetClass();
            Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");

            Class<?> proxySuperClass = rootClass;
            if (rootClass.getName().contains(ClassUtils.CGLIB_CLASS_SEPARATOR)) {
                proxySuperClass = rootClass.getSuperclass();
                Class<?>[] additionalInterfaces = rootClass.getInterfaces();
                for (Class<?> additionalInterface : additionalInterfaces) {
                    this.advised.addInterface(additionalInterface);
                }
            }

            // Validate the class, writing log messages as necessary.
            validateClassIfNecessary(proxySuperClass, classLoader);

            // Configure CGLIB Enhancer...
            Enhancer enhancer = createEnhancer();
            if (classLoader != null) {
                enhancer.setClassLoader(classLoader);
                if (classLoader instanceof SmartClassLoader &&
                        ((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
                    enhancer.setUseCache(false);
                }
            }
            enhancer.setSuperclass(proxySuperClass);
            enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
            enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
            enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(classLoader));

            Callback[] callbacks = getCallbacks(rootClass);
            Class<?>[] types = new Class<?>[callbacks.length];
            for (int x = 0; x < types.length; x++) {
                types[x] = callbacks[x].getClass();
            }
            // fixedInterceptorMap only populated at this point, after getCallbacks call above
            enhancer.setCallbackFilter(new ProxyCallbackFilter(
                    this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
            enhancer.setCallbackTypes(types);

            // Generate the proxy class and create a proxy instance.
            return createProxyClassAndInstance(enhancer, callbacks);
        }
        catch (CodeGenerationException | IllegalArgumentException ex) {
            throw new AopConfigException("Could not generate CGLIB subclass of " + this.advised.getTargetClass() +
                    ": Common causes of this problem include using a final class or a non-visible class",
                    ex);
        }
        catch (Throwable ex) {
            // TargetSource.getTarget() failed
            throw new AopConfigException("Unexpected AOP exception", ex);
        }
    }
}
```

### additionalInterfaces
通过类名中是否带有`$$`判断当前类已经是否已经是代理类，如果是则获取其父类的接口添加进`advice`，后续统一添加给`enhancer`。初步判断这么做是便于标记，否则通过接口获取实现类(例如`BeanFactoryUtils.beanNamesForTypeIncludingAncestors`或AutoWired)时需要递归判断会非常麻烦。

### enhancer.setInterfaces
这里通过`AopProxyUtils.completeProxiedInterfaces(this.advised)`获取接口并配置给代理类，主要有下列几种接口：
- 代理类的接口，即上一步通过additionalInterfaces获取到的接口。
- SpringProxy：必有，用于标记当前类是Spring生成的代理类
- Advised：必有，用来保存代理属性，例如前面流程中创建的Advisor、Advise以及代理接口。
- DecoratingProxy：仅JDK代理类有，用来获取被代理类的Class类型。

那么问题来了，Advised接口是带方法的，但这些方法在被代理类中并没有，Spring是如何添加进去的呢？带着疑问继续往下看。

### Callback
在`getCallbacks`方法中，Spring创建了一系列Interceptor，并依序组合。
```java
Callback[] mainCallbacks = new Callback[] {
        aopInterceptor,  // for normal advice
        targetInterceptor,  // invoke target without considering advice, if optimized
        new SerializableNoOp(),  // no override for methods mapped to this
        targetDispatcher, this.advisedDispatcher,
        new EqualsInterceptor(this.advised),
        new HashCodeInterceptor(this.advised)
};

... 

fixedCallbacks[x] = new FixedChainStaticTargetInterceptor(
                        chain, this.advised.getTargetSource().getTarget(), this.advised.getTargetClass());

```

Callbacks分别是：
- _aopInterceptor_  
    DynamicAdvisedInterceptor，通用拦截器，会获取所有切面并依次执行，也是前两篇文章中讲解过的拦截器。
- _targetInterceptor_   
    直接执行目标方法。配置`expose-proxy="true"`可以将代理类暴露给线程，从而通过`AopContext.currentProxy()`获取代理类，通常用于解决类内方法调方法时切面失效的问题。
- _SerializableNoOp_   
    啥也不做，也不会创建对应的拦截器，与CgLib提供的`NoOp.INSTANCE`相比添加了序列化功能。
- _targetDispatcher_  
    指向被代理类的Dispatcher
- _advisedDispatcher_  
    指向被advised的Dispatcher
- _EqualsInterceptor_  
    Equals方法拦截器。
- _HashCodeInterceptor_  
    HashCode方法拦截器。
- _fixedCallbacks_   
    当前类为静态类，并且Advice链已冻结时，会用FixedChainStaticTargetInterceptor优化性能。功能与DynamicAdvisedInterceptor完全一致。

留意出现一种新的callback类型：`Dispatcher`，它表示将方法的执行转发给Dispatcher对象。Spring通过它使得代理类能动态获取被代理类实例和advised实例。

### CallbackFilter
紧接着，Spring配置CallbackFilter来决定具体处理方法的Callback，他们遵循以下规则：
- _final类型方法_  
不使用Callback，`return 2`，即配置为`SerializableNoOp`。
- _equals()方法_  
`return 5`，即配置为`EqualsInterceptor`。
- _hashcode()方法_  
`return 6`，即配置为`HashCodeInterceptor`。
- _Advised接口中的方法_  
`return 4`，即配置为`AdvisedDispatcher`。还记得前面关于Advised接口方法在代理类中没有实现的疑问吗？从这就能看出，这些方法是通过`AdvisedDispatcher`分发给了`advised`对象处理了，从字节码也能看出调用的是advised的方法。
- _被代理的方法_  
通过`advised.getInterceptorsAndDynamicInterceptionAdvice`获取当前方法的切面，从而判断是方法是否被代理。如果是静态方法并且Advice链已冻结，优化为`fixedCallbacks`中的回调，否则`return 0`使用`DynamicAdvisedInterceptor`。
- _通过配置`expose-proxy="true"`暴露代理的方法_  
由于存在切面，同样需要使用`DynamicAdvisedInterceptor`，以取得链式执行的效果。
- _未被代理的方法_  
原则上是直接交由被代理方法执行，但要细分几种情况：
    - 暴露代理的非静态方法：`return 1`，即配置为`targetInterceptor`。从而保持AopContext的特性。
    - `return this;`的方法：由于`this`是被代理类，如果不做特殊处理，会导致后续的方法调用都无法代理，因此配置为`targetInterceptor`，后者会将返回值包装为代理对象。
    - 其他方法：`return 3`，通过`targetDispatcher`直接交由被代理类执行。

### createProxyClassAndInstance
最终，通过`enhancer.create()`完成整个代理类的创建工作。
---
title: Spring AOP（三）：CgLib代理类详解（施工中）
date: 2022-05-04 19:22:12 +0800
categories: [源码阅读, Spring-AOP]
tags: [Java, Spring, AOP, CgLib]
---

## 前言
在上篇文章里将整个AOP加载和执行梳理了一遍，也埋下了个伏笔，CgLibAopProxy对象是通过ProxyFactory创建的，同时后者还调用了getProxy方法获取代理对象实例。
![1-1 breakpoint_getProxy](/assets/img/20220425/breakpoint_getProxy.png)_1-1 breakpoint getProxy_

同时我们在debug时，也经常能看到类似下图的堆栈，我们的类变成形如`xxxx$$EnhancerBySpringCGLIB$$xxx`：
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
    public String name = "tom";

    public void whoAmI(){
        System.out.println("Hello I'm " + name);
    }
}
```

定义一个方法拦截器，代理类调用目标方法时，CgLib会回调到拦截器上。
```java
class PersonInterceptor implements MethodInterceptor {

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {

        System.out.println("say hello...");
        return methodProxy.invokeSuper(o, objects);
    }
}
```

Enhancer是字节码增强器，我们将Person指定为代理类的父类，将PersonInterceptor指定为方法回调，接着创建代理类并调用方法。
```java
public class TestMain {

    public static void main(String[] args) {
        System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "/temp");

        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(Person.class);
        enhancer.setCallback(new PersonInterceptor());
        Person proxyPerson = (Person) enhancer.create();

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
    // 当前类的Class
    Class var0 = Class.forName("com.zeta.spring.demo.cglib.Person$$EnhancerByCGLIB$$208d9cd0");
    // 原始类的Class
    Class var1;
    // 用反射工具获取原始方法
    CGLIB$whoAmI$0$Method = ReflectUtils.findMethods(new String[]{"whoAmI", "()V"}, (var1 = Class.forName("com.zeta.spring.demo.cglib.Person")).getDeclaredMethods())[0];
    // 创建代理方法
    CGLIB$whoAmI$0$Proxy = MethodProxy.create(var1, var0, "()V", "whoAmI", "CGLIB$whoAmI$0");
    // 下面都是代理Object默认方法
    Method[] var10000 = ReflectUtils.findMethods(new String[]{"finalize", "()V", "equals", "(Ljava/lang/Object;)Z", "toString", "()Ljava/lang/String;", "hashCode", "()I", "clone", "()Ljava/lang/Object;"}, (var1 = Class.forName("java.lang.Object")).getDeclaredMethods());
    CGLIB$finalize$1$Method = var10000[0];
    CGLIB$finalize$1$Proxy = MethodProxy.create(var1, var0, "()V", "finalize", "CGLIB$finalize$1");
    CGLIB$equals$2$Method = var10000[1];
    CGLIB$equals$2$Proxy = MethodProxy.create(var1, var0, "(Ljava/lang/Object;)Z", "equals", "CGLIB$equals$2");
    CGLIB$toString$3$Method = var10000[2];
    CGLIB$toString$3$Proxy = MethodProxy.create(var1, var0, "()Ljava/lang/String;", "toString", "CGLIB$toString$3");
    CGLIB$hashCode$4$Method = var10000[3];
    CGLIB$hashCode$4$Proxy = MethodProxy.create(var1, var0, "()I", "hashCode", "CGLIB$hashCode$4");
    CGLIB$clone$5$Method = var10000[4];
    CGLIB$clone$5$Proxy = MethodProxy.create(var1, var0, "()Ljava/lang/Object;", "clone", "CGLIB$clone$5");
}
```

### 代理方法调用
当通过代理类调用目标方法时，进入了代理类重写的方法：
``` java
public final void whoAmI() {
    // 获取代理拦截器
    MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
    if (var10000 == null) {
        CGLIB$BIND_CALLBACKS(this);
        var10000 = this.CGLIB$CALLBACK_0;
    }

    if (var10000 != null) {
        // 调用拦截器的方法
        var10000.intercept(this, CGLIB$whoAmI$0$Method, CGLIB$emptyArgs, CGLIB$whoAmI$0$Proxy);
    } else {
        super.whoAmI();
    }
}
```

### CGLIB$BIND_CALLBACKS - 载入代理拦截器

CgLib为每个代理类都创建了静态方法`CGLIB$BIND_CALLBACKS`，用于获取目标的代理拦截器。可以看到CgLib优先从ThreadLocal中获取拦截器，其次才是初始化时定义的拦截器，说明在未调用任何代理方法前，仍有机会动态修改代理拦截器。
``` java
private static final void CGLIB$BIND_CALLBACKS(Object var0) {
    // this
    Person$$EnhancerByCGLIB$$208d9cd0 var1 = (Person$$EnhancerByCGLIB$$208d9cd0)var0;
    if (!var1.CGLIB$BOUND) {
        var1.CGLIB$BOUND = true;
        Object var10000 = CGLIB$THREAD_CALLBACKS.get();
        if (var10000 == null) {
            var10000 = CGLIB$STATIC_CALLBACKS;
            if (var10000 == null) {
                return;
            }
        }

        var1.CGLIB$CALLBACK_0 = (MethodInterceptor)((Callback[])var10000)[0];
    }

}
```

### 其他

除了上述方法，代理类中还存在其他的一些方法，例如CgLib提供了

## Spring中的动态代理


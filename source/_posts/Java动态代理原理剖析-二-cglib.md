---
title: Java动态代理原理剖析(二)-cglib
date: 2019-03-04 14:08:21
tags:
    - Java
    - Java源码
    - 设计模式
---

前面一篇文章[Java动态代理原理剖析(一)](https://cfk1996.github.io/2019/02/28/Java%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86%E5%8E%9F%E7%90%86%E5%89%96%E6%9E%90/)分析了JDK版本的动态代理实现，但是它有一个缺陷，生成的代理类会继承Proxy类，而Java是单继承，所以无法对一个没有实现接口的类做代理。

而这篇文章将要介绍的[cglib](https://github.com/cglib/cglib)将会解决这个问题。`cglib`是一个第三方依赖，根据`cglib`的wiki描述，它是一个有力的、高性能的、高质量的代码生成库，被用来在运行时继承JAVA类或实现接口。

### 如何使用cglib

前一篇文章已经介绍了动态代理的相关知识，那么现在就直接看下cglib是如何使用的.

<!--more-->

```java
//　首先实现我们的Watch类，注意与前一篇文章的区别，这里的Watch是一个类，而不是接口
public class Watch {
    public void watchTv() {
        System.out.println("watch tv, watch tv");
    }
    // 注意这次watchPhone方法有个final修饰符
    public final void watchPhone() {
        System.out.println("watch phone, watch phone");
    }
}

//　然后需要一个实现了MethodInterceptor的类，这个类的作用与JDK中InvocationHandler的作用差不多
//　重写intercept方法来实现我们的代理逻辑，在方法调用前后输出提示信息
public class MyInterceptor implements MethodInterceptor {

    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("before----cglib");
        // 通过proxy来调用，而不是之前的method，因为这里的obj不是Watch对象，而是Enhancer对象
        // JDK动态代理中的obj是在构造参数中传入的watch实例。
        proxy.invokeSuper(obj, args);
        System.out.println("after-----cglib");
        return null;
    }
}

//　cglib的动态代理实现需要创建一个Enhancer对象，设置需要代理的类和代理逻辑等，通过create方法创建出代理类
public class CglibDemo {

    public static void main(String[] args) {
        Enhancer eh = new Enhancer();
        // 设置需要代理的类
        eh.setSuperclass(Watch.class);
        // 设置代理逻辑
        eh.setCallback(new MyInterceptor());
        //　创建代理类，类型为Watch
        //　而Watch不是一个接口，那么可以猜测出cglib是通过继承来实现代理类
        Watch watch = (Watch) eh.create();
        watch.watchPhone();
        watch.watchTv();
    }
}
```

输出结果如下:

```
watch phone, watch phone
before----cglib
watch tv, watch tv
after-----cglib
```

**注意这里，watchPhone方法没有被代理，watchTv方法被代理了，后面分析原因**

### 深入源码分析

使用方法很简单，那么现在去源码里看看它如何生成代理类的。

首先看下`MethodInterceptor`接口:
```java
public interface MethodInterceptor　extends Callback
{
    /**
     * @param obj "this", cglib生成的代理类对象
     * @param method 被拦截(代理)的方法
     * @param args 被拦截方法的参数
     * @param proxy 用来调用被代理对象的原方法
     */    
    public Object intercept(Object obj, java.lang.reflect.Method method, Object[] args,
                               MethodProxy proxy) throws Throwable;

}
```

```java
// 然后看下Enhancer的构造函数
public Enhancer() {
    // source是一个静态变量，是一个保存了Enhancer.class.getName()变量的类
    super(SOURCE);
}

// 跟着cglib动态代理的使用步骤，看一下setSuperClass方法
private Class[] interfaces;
private Class superclass;
//　父类只能一个，而接口可以多个
public void setSuperclass(Class superclass) {
    if (superclass != null && superclass.isInterface()) {
        setInterfaces(new Class[]{ superclass });
    } else if (superclass != null && superclass.equals(Object.class)) {
        // affects choice of ClassLoader
        this.superclass = null;
    } else {
        this.superclass = superclass;
    }
}
public void setInterfaces(Class[] interfaces) {
    this.interfaces = interfaces;
}

// 继续跟着使用步骤，看下setCallback方法
private Callback[] callbacks;
public void setCallback(final Callback callback) {
    setCallbacks(new Callback[]{ callback });
}
public void setCallbacks(Callback[] callbacks) {
    if (callbacks != null && callbacks.length == 0) {
        throw new IllegalArgumentException("Array cannot be empty");
    }
    this.callbacks = callbacks;
}
// 我们编写的MethodInterceptor基础自Callback
public interface MethodInterceptor　extends Callback
```

```java
// 前面几个只是做了一些赋值操作，比较简单，现在看下核心方法create
// 代理类通过Enhance.create方法生成
public Object create() {
    classOnly = false;
    argumentTypes = null;
    return createHelper();
}
private Object createHelper() {
    // 校验callback类型和filter的正确性
    //　在我们这个例子中，callback类型为`MethodInterceptor`，filter=ALL_ZERO
    preValidate();
    //　等价于KEY_FACTORY.newInstance(com.xxx.xxx.cglibDemo.Watch, null, null, {MethodInterceptor}, true, true, null)
    // 这段看不懂，生成了一个Key
    Object key = KEY_FACTORY.newInstance((superclass != null) ? superclass.getName() : null,
            ReflectUtils.getNames(interfaces),
            filter == ALL_ZERO ? null : new WeakCacheKey<CallbackFilter>(filter),
            callbackTypes,
            useFactory,
            interceptDuringConstruction,
            serialVersionUID);
    this.currentKey = key;
    Object result = super.create(key);
    return result;
}

//　进入super.create()看一看
protected Object create(Object key) {
    try {
        // 获得一个appclassloader
        ClassLoader loader = getClassLoader();
        Map<ClassLoader, ClassLoaderData> cache = CACHE;
        ClassLoaderData data = cache.get(loader);
        if (data == null) {
            synchronized (AbstractClassGenerator.class) {
                cache = CACHE;
                data = cache.get(loader);
                if (data == null) {
                    Map<ClassLoader, ClassLoaderData> newCache = new WeakHashMap<ClassLoader, ClassLoaderData>(cache);
                    data = new ClassLoaderData(loader);
                    newCache.put(loader, data);
                    CACHE = newCache;
                }
            }
        }
        this.key = key;
        Object obj = data.get(this, getUseCache());
        if (obj instanceof Class) {
            // debug过程进入该分支
            return firstInstance((Class) obj);
        }
        return nextInstance(obj);
    } catch (RuntimeException e) {
        throw e;
    } catch (Error e) {
        throw e;
    } catch (Exception e) {
        throw new CodeGenerationException(e);
    }
}
// 抱歉，后面这些代码看不懂
```

enhancer类的方法难度比JDK的大太多了，我在debug的过程中也不能很好的理解其中一些步骤，就不分析出来误导大家了。

但是没有关系，我们直接分析生成的动态代理类的源码，来推断出它是如何实现动态代理过程的。

#### 保存代理类

cglib提供了辅助方法来保存.class文件，然后用intellij idea打开
```java
// 在CglibDemo的main方法中加入这行代码
System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "/home/chenfeikun/repo/java/test/src/main/java/com/github/chenfeikun/cglibDemo");
```

### 代理类源码分析

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.github.chenfeikun.cglibDemo;

import java.lang.reflect.Method;
import net.sf.cglib.core.ReflectUtils;
import net.sf.cglib.core.Signature;
import net.sf.cglib.proxy.Callback;
import net.sf.cglib.proxy.Factory;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

// cglib通过继承的方式来实现对没有接口的类进行代理，而JDK是通过implements的方式
//　因此(Watch)可以对enhancer.create()进行强制类型转换，因为Watch是代理类的父类
//　同时又因为是继承的关系，所以导致Watch类中被final修饰的方法无法被Override，因此前面的Watchphone方法没有被代理
public class Watch$$EnhancerByCGLIB$$a4ac3b63 extends Watch implements Factory {
    // 很多变量声明，直接跳过
    private boolean CGLIB$BOUND;
    public static Object CGLIB$FACTORY_DATA;
    private static final ThreadLocal CGLIB$THREAD_CALLBACKS;
    private static final Callback[] CGLIB$STATIC_CALLBACKS;
    private MethodInterceptor CGLIB$CALLBACK_0;
    private static Object CGLIB$CALLBACK_FILTER;
    private static final Method CGLIB$watchTv$0$Method;
    private static final MethodProxy CGLIB$watchTv$0$Proxy;
    private static final Object[] CGLIB$emptyArgs;
    private static final Method CGLIB$equals$1$Method;
    private static final MethodProxy CGLIB$equals$1$Proxy;
    private static final Method CGLIB$toString$2$Method;
    private static final MethodProxy CGLIB$toString$2$Proxy;
    private static final Method CGLIB$hashCode$3$Method;
    private static final MethodProxy CGLIB$hashCode$3$Proxy;
    private static final Method CGLIB$clone$4$Method;
    private static final MethodProxy CGLIB$clone$4$Proxy;

    static void CGLIB$STATICHOOK1() {
        CGLIB$THREAD_CALLBACKS = new ThreadLocal();
        CGLIB$emptyArgs = new Object[0];
        Class var0 = Class.forName("com.github.chenfeikun.cglibDemo.Watch$$EnhancerByCGLIB$$a4ac3b63");
        Class var1;
        //　代理equal, toString, hashCode, clone方法
        Method[] var10000 = ReflectUtils.findMethods(new String[]{"equals", "(Ljava/lang/Object;)Z", "toString", "()Ljava/lang/String;", "hashCode", "()I", "clone", "()Ljava/lang/Object;"}, (var1 = Class.forName("java.lang.Object")).getDeclaredMethods());
        CGLIB$equals$1$Method = var10000[0];
        CGLIB$equals$1$Proxy = MethodProxy.create(var1, var0, "(Ljava/lang/Object;)Z", "equals", "CGLIB$equals$1");
        CGLIB$toString$2$Method = var10000[1];
        CGLIB$toString$2$Proxy = MethodProxy.create(var1, var0, "()Ljava/lang/String;", "toString", "CGLIB$toString$2");
        CGLIB$hashCode$3$Method = var10000[2];
        CGLIB$hashCode$3$Proxy = MethodProxy.create(var1, var0, "()I", "hashCode", "CGLIB$hashCode$3");
        CGLIB$clone$4$Method = var10000[3];
        CGLIB$clone$4$Proxy = MethodProxy.create(var1, var0, "()Ljava/lang/Object;", "clone", "CGLIB$clone$4");
        //　代理Watch类的非final方法
        //　因此Watch类的final方法watchPhone没有被代理
        CGLIB$watchTv$0$Method = ReflectUtils.findMethods(new String[]{"watchTv", "()V"}, (var1 = Class.forName("com.github.chenfeikun.cglibDemo.Watch")).getDeclaredMethods())[0];
        CGLIB$watchTv$0$Proxy = MethodProxy.create(var1, var0, "()V", "watchTv", "CGLIB$watchTv$0");
    }

    final void CGLIB$watchTv$0() {
        super.watchTv();
    }

    public final void watchTv() {
        // var10000指向MethodInterceptor的实现类
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        if (var10000 != null) {
            //　var10000应该不为Null,进入该分支，调用其intercept方法
            // intercept四个方法的参数为代理类对象，　被代理方法，方法参数和用来调用对象的代理
            var10000.intercept(this, CGLIB$watchTv$0$Method, CGLIB$emptyArgs, CGLIB$watchTv$0$Proxy);
        } else {
            super.watchTv();
        }
    }
    // equals形式与watchTv一样
    final boolean CGLIB$equals$1(Object var1) {
        return super.equals(var1);
    }

    public final boolean equals(Object var1) {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        if (var10000 != null) {
            Object var2 = var10000.intercept(this, CGLIB$equals$1$Method, new Object[]{var1}, CGLIB$equals$1$Proxy);
            return var2 == null ? false : (Boolean)var2;
        } else {
            return super.equals(var1);
        }
    }
    // ......
    //　篇幅原因省略其他被代理方法，包括hashcode, toString等;格式与watchTv与equals方法都是一模一样的
    // 后面还有不少静态方法暂时也没用到，可以直接跳过
}
```

那么看到这里就弄明白一大半了，只要再弄清楚

    public Object intercept(Object obj, java.lang.reflect.Method 
    method, Object[] args, MethodProxy proxy)

中`MethodProxy`对象是个什么东西就行。

### MethodProxy及FastClass机制分析

在cglib反编译的代码中是这样生成Proxy对象的:

```java
Class var1 = Class.forName("com.github.chenfeikun.cglibDemo.Watch"));
Class var0 = Class.forName("com.github.chenfeikun.cglibDemo.Watch$$EnhancerByCGLIB$$a4ac3b63");
CGLIB$watchTv$0$Proxy = MethodProxy.create(var1, var0, "()V", "watchTv", "CGLIB$watchTv$0");

// 进入create方法看一看
// 从上面的使用方式来看，c1是被代理的类，c2是cglib生成的代理类
public static MethodProxy create(Class c1, Class c2, String desc, String name1, String name2) {
    MethodProxy proxy = new MethodProxy();
/**Signature:
 * A representation of a method signature, containing the method name,
 * return type, and parameter types.
 * 函数签名，保存了方法的名字，返回值类型和参数类型
 */
    proxy.sig1 = new Signature(name1, desc);
    proxy.sig2 = new Signature(name2, desc);
    proxy.createInfo = new CreateInfo(c1, c2);
    return proxy;
}

// create方法好像看不出啥，那么回到之前demo的例子中，我们是使用了proxy.invokeSuper(obj, args)来调用被代理方法的
// 调用链路分析
// var10000.intercept(this, CGLIB$watchTv$0$Method, CGLIB$emptyArgs, CGLIB$watchTv$0$Proxy);
// CGLIB$watchTv$0$Proxy.invokeSuper(this, CGLIB$emptyArgs)
// 参数意义：obj是生成的代理类，args是方法参数
public Object invokeSuper(Object obj, Object[] args) throws Throwable {
    try {
        init();
        FastClassInfo fci = fastClassInfo;
        return fci.f2.invoke(fci.i2, obj, args);
    } catch (InvocationTargetException e) {
        throw e.getTargetException();
    }
}
//　看下init方法
private void init()　{
    //　双重检查保证fastClassInfo只被初始化一次
    if (fastClassInfo == null)　{
        synchronized (initLock)　{
            if (fastClassInfo == null)　{
                CreateInfo ci = createInfo;
                FastClassInfo fci = new FastClassInfo();
                // helper函数生成c1和c2的FastClass的类对象
                fci.f1 = helper(ci, ci.c1);
                fci.f2 = helper(ci, ci.c2);
                // i1和i2的全称可以看作是index1,index2
                //　在JDK版本中，是通过反射的机制来获取method，这样效率比较低下
                //　所以cglib通过对Signature建立索引来对加快对method的寻址,getIndex下面介绍
                fci.i1 = fci.f1.getIndex(sig1);
                fci.i2 = fci.f2.getIndex(sig2);
                fastClassInfo = fci;
                createInfo = null;
            }
        }
    }
}

//现在介绍下getIndex方法，FastClass方法中的getIndex方法是抽象方法
//cglib同时生成了一个继承自FastClass的类，重写了其getIndex方法
public int getIndex(Signature var1) {
    String var10000 = var1.toString();
    switch(var10000.hashCode()) {
    case -2055565910:
        if (var10000.equals("CGLIB$SET_THREAD_CALLBACKS([Lnet/sf/cglib/proxy/Callback;)V")) {
            return 4;
        }
        break;
    case -1909335397:
        if (var10000.equals("CGLIB$watchTv$0()V")) {
            return 7;
        }
        break;
    //...省略其他方法
    ｝
}

//　回到invokerSuper的方法中的fci.f2.invoke(fci.i2, obj, args)，obj是生成的Watch代理类
// 同样的，invoke方法也是FastClass的抽象方法，被cglib生成的继承了FastClass的类重写
//　通过index2的值，直接调用方法，省去了反射的时间消耗
public Object invoke(int var1, Object var2, Object[] var3) throws InvocationTargetException {
    Watch var10000 = (Watch)var2;
    int var10001 = var1;
    try {
        switch(var10001) {
        case 0:
            var10000.watchPhone();
            return null;
        case 1:
        //根据Index,直接调用原方法
            var10000.watchTv();
            return null;
        case 2:
            return new Boolean(var10000.equals(var3[0]));
        case 3:
            return var10000.toString();
        case 4:
            return new Integer(var10000.hashCode());
        }
    } catch (Throwable var4) {
        throw new InvocationTargetException(var4);
    }
    throw new IllegalArgumentException("Cannot find matching method/constructor");
}
```

### 总结

那么到这里，cglib实现动态代理的原理也已经分析完了。cglib是通过继承的方式来对类进行代理，因此只能代理非final方法，同时在代理调用方法时，通过FastClass机制来代替JDK中通过反射查找方法的机制来加快方法的调用，FastClass机制则是通过对函数进行签名和建立索引来快速找到方法，相当于一个Map<Index, Method>。

没看过**JDK版本**的动态代理实现原理的，可以先看一下之前的那篇再回头看这篇，就能彻底理解Java中的动态代理～

[Java动态代理原理剖析(一)](https://cfk1996.github.io/2019/02/28/Java%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86%E5%8E%9F%E7%90%86%E5%89%96%E6%9E%90/)

### 欢迎关注我的公众号

欢迎关注我的公众号，会经常分享一些技术文章和生活随笔～

![技术旅途](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/wechat.jpg)


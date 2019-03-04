---
title: Java动态代理原理剖析(一)
date: 2019-02-28 17:24:35
tags:
    - Java
    - Java源码
    - 设计模式
---

这篇文章将深入的分析Java的动态代理实现原理，首先将自己实现一个动态代理，然后顺势分析JDK中的动态代理实现并与自己实现的版本作对比，第二篇文章再分析cglib版本的动态代理实现原理，用两篇文章让你彻底搞懂动态代理。

### 静态代理

在了解动态代理之前，先介绍一下静态代理。代理模式就是实现一个代理来控制被代理对象的一些行为，一些常见的作用包括记录被代理对象的执行时间或者添加访问权限，屏蔽不必要的请求等，不理解代理模式的可以先去查找资料简单了解，不过多介绍。

首先给出个在执行方法前后输出提示信息的静态代理的例子，非常的简单。

<!--more-->

```java
//　首先定义一个接口
public interface Watch {
    void watchTv();

    void watchPhone();
}
// 编写实现类
public class WatchImp implements Watch{
    @Override
    public void watchTv() {
        System.out.println("watch tv");
    }

    @Override
    public void watchPhone() {
        System.out.println("watch phone");
    }
}
// 编写代理类,在watch方法调用前后输出提示信息
public class WatchProxy implements Watch {

    private Watch object;

    public WatchProxy(Watch object) {
        this.object = object;
    }
    @Override
    public void watchTv() {
        System.out.println("before-------");
        object.watchTv();
        System.out.println("after--------");
    }

    @Override
    public void watchPhone() {
        System.out.println("before-------");
        object.watchPhone();
        System.out.println("after--------");
    }
}
//　可以看出代理类和实现类继承了同样的接口，以组合的关系联系在一起
//　调用代理类的方法
public class ProxyDemo {
    public static void main(String[] args) {
        Watch watchProxy = new WatchProxy(new WatchImp());
        watchProxy.watchTv();
        watchProxy.watchPhone();
    }
｝
```

##### 输出结果:
```
before-------
watch tv
after--------
before-------
watch phone
after--------
```

显而易见静态代理很好的完成了我们的任务，在调用方法的前后输出提示消息，但是它还具有几个缺点

* 如果接口定义了1000个方法，代理类中的`System.out`逻辑需要重复1000次
* 如果有1000个不同的接口都需要代理，那么需要1000个逻辑相同的但是实现了不同接口的不同代理类

那么，我们是否可以用同一个代理类(例如PrintAroundProxy)来代理任意对象的任意方法呢，使它们的方法在执行前后可以输出提示消息？更进一步的，我们是不是也可以指定代理的逻辑呢，比如不是输出提示消息，而是获取方法的执行时间呢，这些都将在动态代理的章节接开谜底。

### 动态代理

我们先从上面的静态代理开始分析，能不能不要自己写`WatchProxy`类，而是让程序运行时生成这个类，然后加载它呢。我们将编写`WatchProxy`类的过程抽象成一个算法，而这个算法可以生成`WatchProxy`类。下面将借助[javapoet](https://github.com/square/javapoet)这个第三方库帮助我们生成`WatchProxy`类。

#### 生成WatchProxy

```java
//　看不懂没关系，快速的扫一遍，简单的把这段程序理解为用编程的方式写出一个类
public class Proxy {

    public static Object newProxyInstance() throws IOException {
        TypeSpec.Builder typeSpecBuilder = TypeSpec.classBuilder("WatchProxy")
                .addSuperinterface(Watch.class);

        FieldSpec fieldSpec = FieldSpec.builder(Watch.class, "object", Modifier.PRIVATE).build();
        typeSpecBuilder.addField(fieldSpec);

        MethodSpec constructorMethodSpec = MethodSpec.constructorBuilder()
                .addModifiers(Modifier.PUBLIC)
                .addParameter(Watch.class, "object")
                .addStatement("this.object = object")
                .build();
        typeSpecBuilder.addMethod(constructorMethodSpec);

        Method[] methods = Watch.class.getDeclaredMethods();
        for (Method method : methods) {
            MethodSpec methodSpec = MethodSpec.methodBuilder(method.getName())
                    .addModifiers(Modifier.PUBLIC)
                    .addAnnotation(Override.class)
                    .returns(method.getReturnType())
                    .addStatement("$T.out.println(\"before--------\")", System.class)
                    .addCode("\n")
                    .addStatement("this.object." + method.getName() + "()")
                    .addCode("\n")
                    .addStatement("$T.out.println(\"after---------\")", System.class)
                    .build();
            typeSpecBuilder.addMethod(methodSpec);
        }

        JavaFile javaFile = JavaFile.builder("com.github", typeSpecBuilder.build()).build();
        // 为了看的更清楚，我将源码文件写入文件
        javaFile.writeTo(new File("/home/chenfeikun/repo/java/test/src/main/java"));
        return null;
    }

    public static void main(String[] args) throws Exception {
        Proxy.newProxyInstance();
    }
}
```

执行这里的main方法，会在`new File("/home/chenfeikun/repo/java/test/src/main/java")`对应的路径下找到生成的`WatchProxy`类，会发现和之前自己写的静态代理是一模一样的。(这时候对照着程序生成的类再回过头看看上面的代码，会容易很多)那么到这里我们已经可以动态生成一个代理类了，但是还是有一些缺陷。

* 只能动态生成WatchProxy，只能代理Watch接口，不能生成我们想要的Time, Sing...等其他接口代理
* 代理的逻辑只是在执行前后输出提示信息，不是我们想要的任意逻辑

为了解决这两个问题，我们需要对生成类的算法进一步抽象，既然想要代理任意接口，那么我们可以把接口当作参数传递进`newProxyInstance()`方法，同理执行逻辑也可以作为参数一并传入这个方法。

接口作为参数传入进来的话，我想大家是可以很容易做到的，只要把前面的代码中一些硬编码的`Watch.class`用参数替换掉，但是方法的逻辑如何作为一个参数呢？我们需要一个额外的接口`InvocationHandlerDemo`来帮忙.

```java
public interface InvocationHandlerDemo {
     Object invoke(Method method, Object[] args) throws Exception;
}
//　实现自己的InvocationHandlerDemo类，里面封装了代理逻辑
public class MyInvocationDemo implements InvocationHandlerDemo {
    // 因为传入的是Object对象，因此可以代理所有类，而不仅限于Watch接口实现类
    private Object object;
    public MyInvocationDemo(Object object) {
        this.object = object;
    }
    // invode方法封装了代理实现逻辑-->在方法执行前后输出提示信息
    //　也可以根据自己的需求，修改代理逻辑
    @Override
    public Object invoke(Method method, Object[] args) throws Exception{
        System.out.println("before----------");
        method.invoke(object, args);
        System.out.println("after----------");
        return null;
    }
}
```

现在可以先不要往下看，思考一下我们已经有了自己实现的封装了的代理逻辑的`InvocationHandler`实现类，如何把它放入我们前面`newProxyInstance`方法中呢？

没有想出来也没关系，看我的分析，自己尝试多写几遍就能够理解，现在我们来改造一下之前的`Proxy`类。

```java
public class Proxy {
    // 新增了两个参数，clazz是传入的接口，invocatinHandler是实现了代理逻辑的实例
    public static Object newProxyInstance(Class<?> clazz, InvocationHandlerDemo invocationHandler) throws IOException {
        //　用ProxyHandler代替之前的WatchProxy，让生成的代理类名字更通用
        //　addSuperinterface用传进来的接口参数代替硬编码的Watch.class，因此生成的代理类可以成为任意一个接口的实现类
        TypeSpec.Builder typeSpecBuilder = TypeSpec.classBuilder("ProxyHandler")
                .addSuperinterface(clazz);
        //　将之前Watch.class对象的成员变量转换为Invocation对象的成员变量
        FieldSpec fieldSpec = FieldSpec.builder(InvocationHandlerDemo.class, "object", Modifier.PRIVATE).build();
        typeSpecBuilder.addField(fieldSpec);
　　　　　// 将构造函数里的Watch.class也修改成Invocation
        MethodSpec constructorMethodSpec = MethodSpec.constructorBuilder()
                .addModifiers(Modifier.PUBLIC)
                .addParameter(InvocationHandlerDemo.class, "object")
                .addStatement("this.object = object")
                .build();
        typeSpecBuilder.addMethod(constructorMethodSpec);

        Method[] methods = clazz.getDeclaredMethods();
        //　因为传入的参数可以确定是Invocation的实现类，则可以保证它实现了invoke方法
        // 而invoke方法实现了代理逻辑，那么在生成的代码中直接调用invoke方法即可。
        for (Method method : methods) {
            MethodSpec methodSpec = MethodSpec.methodBuilder(method.getName())
                    .addModifiers(Modifier.PUBLIC)
                    .addAnnotation(Override.class)
                    .returns(method.getReturnType())
                    .addCode("\n")
                    .addCode("try {")
                    .addCode("\n")
                    .addStatement("\t$T method = " + clazz.getName() + ".class.getMethod(\"" + method.getName() + "\")", Method.class)
                    .addStatement("this.object.invoke(method, null)")
                    .addCode("} catch (Exception e) {}")
                    .addCode("\n")
                    .build();
            typeSpecBuilder.addMethod(methodSpec);
        }

        JavaFile javaFile = JavaFile.builder("com.github", typeSpecBuilder.build()).build();
        // 为了看的更清楚，我将源码文件写入文件
        javaFile.writeTo(new File("/home/chenfeikun/repo/java/test/src/main/java"));

        return null;
    }

    public static void main(String[] args) throws Exception {
        //　调用时需要把接口实现类WatchImp的一个实例作为参数传给Invocation
        Proxy.newProxyInstance(Watch.class, new MyInvocationDemo(new WatchImp()));
    }
}
```

现在可以看一下生成的`ProxyHandler`长成这样：

```java
class ProxyHandler implements Watch {
    private InvocationHandlerDemo object;

    public ProxyHandler(InvocationHandlerDemo object) {
        this.object = object;
    }

    @Override
    public void watchPhone() {
        try {
    	    Method method = com.github.chenfeikun.proxy.Watch.class.getMethod("watchPhone");
            this.object.invoke(method, null);
        } catch (Exception e) {}
  }

    @Override
    public void watchTv() {
        try {
    	    Method method = com.github.chenfeikun.proxy.Watch.class.getMethod("watchTv");
            this.object.invoke(method, null);
        } catch (Exception e) {}
  }
}
```

那么现在这个`ProxyHandler`类实现了Watch接口，但是内部组合的是`InvocationHandler`的实例对象，所以可以把`ProxyHandler`看做是`InvocationHandler`的一个静态代理，理解这句话是非常关键的，好好体会下可以发现动态代理其实是用了两次静态代理来实现的

* 第一次是`MyInvocationHandler`，`MyInvocationHandler`的构造函数可以传入一个Object对象，然后接管对Object对象方法的调用，所以`MyInvocationHandler`是可以看做Object的一个静态代理
* 第二次就是动态代理生成的这个`ProxyHandler`类，其构造函数可以传入一个Invocation对象，然后接管Invocation方法的调用，但确是以Watch的接口名称(watchTv, watchPhone)来间接调用

现在已经实现了前面的目标了，可以对任意对象进行代理(通过传入不同的需要代理的接口实现到`InvocationHandler`的构造函数中)，可以定制代理的行为(修改`InvocationHandler`的`invoke`方法)。但是还有一个小问题没有解决，那就是我们现在只是把这个`ProxyHandler`写入一个新的文件，并没有在运行时加载它。那么只需要根据生成的新文件的路径，将其运行时编译后用`classloader`来将其加载进内存，这部分没有实操过，就不自己实现加载过程了，将在后面分析JDK源码时介绍。

### JDK动态代理源码分析

上面已经自己动手完成了动态代理的实现，现在我们看一看JDK是如何实现的.

首先看下JDK的动态代理如何使用

```java
public class ProxyDemo {

    public static void main(String[] args) {
        Watch proxy = (Watch) Proxy.newProxyInstance(Watch.class.getClassLoader(), WatchImp.class.getInterfaces(),
                new MyInvocationHandler(new WatchImp()));
        proxy.watchPhone();
        proxy.watchTv();
    }
}
public class MyInvocationHandler implements InvocationHandler {

    private Object object;
    public MyInvocationHandler(Object object) {
        this.object = object;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before----------");
        method.invoke(object, args);
        System.out.println("after----------");
        return null;
    }
}
```

可以看出来，我们自己实现的动态代理是JDK的一个简易版本，JDK的`newProxyInstance`方法比我们的版本多了一个`ClassLoader`参数用来加载动态生成的类，正是我们之前实现版本所缺少的部分。JDK的`InvocationHandler`接口的`invoke`方法多了一个`Object proxy`参数，剩下的其他几乎都是一样的。

#### newProxyInstance

很容易就可以看出来`newProxyInstance`是最核心的方法，那么就从它开始进行源码分析。

```java
public static Object newProxyInstance(ClassLoader loader,
                                    　Class<?>[] interfaces,
                                    　InvocationHandler h)
        throws IllegalArgumentException　{
    //　确保代理逻辑非空
    Objects.requireNonNull(h);
    //　获取传入的接口
    final Class<?>[] intfs = interfaces.clone();
    // 没看懂，看名字是安全检测相关的
    final SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
    }
    // 源码注释如下：查找或生成代理类;后面分析
    Class<?> cl = getProxyClass0(loader, intfs);
    try {           
         // 删除一些不重要的...

        // private static final Class<?>[] constructorParams = { InvocationHandler.class };
        // 获取生成的代理类带有Class数组参数的构造函数
        final Constructor<?> cons = cl.getConstructor(constructorParams);
        final InvocationHandler ih = h;
        // 修改获取权限
        if (!Modifier.isPublic(cl.getModifiers())) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                public Void run() {
                    cons.setAccessible(true);
                    return null;
                }
            });
        }
        //　返回生成对象的一个实例
        return cons.newInstance(new Object[]{h});
    } catch () {
        //　一堆异常处理
    ｝
}

//　现在看下getProxyClass0方法如何生成代理类
private static Class<?> getProxyClass0(ClassLoader loader,
                                    　　Class<?>... interfaces) {
    //　接口数量大于65535抛出异常
    if (interfaces.length > 65535) {
        throw new IllegalArgumentException("interface limit exceeded");
    }
    //　从代理类的缓存中获取代理类
    return proxyClassCache.get(loader, interfaces);
}
//　proxyClassCache是一个代理类缓存池，类似于map
private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
    proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());
// WeakCache初始化时传入两个工厂函数，其get方法调用时，如果发现entry不存在，会调用工厂函数生成key和value
//　keyFactory根据interface的数量和hashcode，生成一个key
// 看下生成Proxy的工厂函数
private static final class ProxyClassFactory
    implements BiFunction<ClassLoader, Class<?>[], Class<?>>
{
    //　生成代理类名字的前缀
    private static final String proxyClassNamePrefix = "$Proxy";
    // 一个递增的数字，用来加在前缀之后组成唯一的代理类名
    private static final AtomicLong nextUniqueNumber = new AtomicLong();
    @Override
    public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {
        Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
        for (Class<?> intf : interfaces) {
            Class<?> interfaceClass = null;
            //　检验类加载器根据接口名字加载出来的类与传入的接口是否是同一个
            try {
                interfaceClass = Class.forName(intf.getName(), false, loader);
            } catch (ClassNotFoundException e) {
            }
            if (interfaceClass != intf) {
                throw new IllegalArgumentException(
                    intf + " is not visible from class loader");
            }
            //　检验传入的interface是接口，而不是类
            if (!interfaceClass.isInterface()) {
                throw new IllegalArgumentException(
                    interfaceClass.getName() + " is not an interface");
            }
            // 检验传入的接口数组不重复
            if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
                throw new IllegalArgumentException(
                    "repeated interface: " + interfaceClass.getName());
            }
        }
        // proxy所在的package
        String proxyPkg = null;
        int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

        //　记录所有非public的接口的package,确保它们是同一个package下的接口
        // 并作为生成proxy所在的package
        for (Class<?> intf : interfaces) {
            int flags = intf.getModifiers();
            if (!Modifier.isPublic(flags)) {
                accessFlags = Modifier.FINAL;
                String name = intf.getName();
                int n = name.lastIndexOf('.');
                String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                if (proxyPkg == null) {
                    proxyPkg = pkg;
                } else if (!pkg.equals(proxyPkg)) {
                    throw new IllegalArgumentException(
                        "non-public interfaces from different packages");
                }
            }
        }
        // 如果没有非public的接口，使用com .sun.proxy package
        if (proxyPkg == null) {
            proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
        }
        // 生成新的递增数字，作为生成的Proxy class name
        long num = nextUniqueNumber.getAndIncrement();
        String proxyName = proxyPkg + proxyClassNamePrefix + num;

        //　生成代理类并且编译成class文件，ProxyGenerator是sun.misc下的类，查找不到源代码了
        byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
            proxyName, interfaces, accessFlags);
        try {
            // defineClass0是native方法，也看不到源代码，可以猜测一下这里传入了ClassLoader，用来加载生成的proxyClassFile，而且直接是一个Byte[]数组，省去了我们写入文件，再读取编译的过程
            return defineClass0(loader, proxyName,
                                proxyClassFile, 0, proxyClassFile.length);
        } catch (ClassFormatError e) {
            throw new IllegalArgumentException(e.toString());
        }
    }
}
```

很遗憾，分析到后面有些方法无法深入下去了，但是这不妨碍我们理解动态代理。因为我们自己实现的版本中已经实现了如何生成一个Java类，JDK的版本会有更多的校验机制，异常检测、更好的性能罢了(我瞎猜的)，思路应该是一样的。

虽然这些方法看不到了，但是分析源码过程中发现我们能够通过`ProxyGenerator.generateProxyClass`方法得到生成的代理类编译过后的字符数组，那么我们把它写入文件，看看JDK生成的代理是个什么模样。

```java
// 写入文件方法，在main方法中调用，生成编译过后的.class文件
saveGenerateProxyClass("Proxy0", WatchImp.class.getInterfaces());
public static void saveGenerateProxyClass(String name, Class<?> [] interfaces) {
    byte[] classFile = ProxyGenerator.generateProxyClass(name, interfaces);
    FileOutputStream out = null;
    try {
        String filePath = "/home/chenfeikun/" + name + ".class";
        out = new FileOutputStream(filePath);
        out.write(classFile);
        out.flush();
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        try {
            if(out != null) out.close();
        } catch (IOException e) {
        }
    }
}
```

用intellij打开生成的`Proxy.class`文件，intellij会帮我们反编译成我们看得懂的源码，结果如下:

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

import com.github.chenfeikun.proxy.Watch;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class Proxy0 extends Proxy implements Watch {
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m4;
    private static Method m0;

    // 我添加的，父类Proxy的构造函数长这样
    // protected Proxy(InvocationHandler h) {
    //    Objects.requireNonNull(h);
    //    this.h = h;
    //}
    public Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void watchTv() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void watchPhone() throws  {
        try {
            super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("com.github.chenfeikun.proxy.Watch").getMethod("watchTv");
            m4 = Class.forName("com.github.chenfeikun.proxy.Watch").getMethod("watchPhone");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

#### 对比我们的代理类与JDK实现的代理类

* 共同点
    * 都implement传入的接口
    * 代理类的方法调用，最终都是指向invocation.invoke方法

* 不同点
    * JDK有更多的异常检测
    * JDK实现的版本继承了Proxy父类，我们的没继承
    * JDK版本还有几个额外方法，equal, toString, hashCode
    * JDK版本把这些方法定义为final的，还定义成static的

到目前为止，我们生成的动态代理都是对接口进行代理，是为什么呢？并不是所有实现类都来自接口，如果实现类没有接口怎么办呢？

**public final class Proxy0 extends Proxy implements Watch**

这是我们生成的代理类，它继承了Proxy父类，而Java只能够实现单继承，这导致了通过JDK的Proxy方式无法对一个类进行代理，这就是JDK版本的局限性，而cglib版本的动态代理可以解决这个问题，将在下一篇文章进行分析。

### 欢迎关注我的公众号

欢迎关注我的公众号，会经常分享一些技术文章和生活随笔～

![技术旅途](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/wechat.jpg)
---
title: Java几个基础对象源码
date: 2020-06-21 16:41:36
tags:
    - Java
    - Java源码
    - Java基础
---

又到周末了，玩了一天半，还有半天了，做点什么吧...今天来看看Java中几个常用数据类型的源码：`String, Integer, Long`等。JDK中的每个源码的注释都很清楚，读一下相关的注释就可以了解很多关于这个类的信息，有时间的可以看看。

<!--more-->

### String

首先看下最常用的字符串`String`,`String`的成员变量只有三个

```java
// 字符串底层就是用char数组来存储数据，数组用final修饰，
// 所以字符串是不可变的
private final char value[];

// 初始值为0，调用hashCode方法时判断是否等于0
// 如果等于0，则计算出hash值并赋值
private int hash; 
// 序列化id，不清楚的可以看我之前写的《jdk序列化与反序列化底层机制》
private static final long serialVersionUID = -6849794470754667710L;
```

String的源码中共有16个不同形式的构造函数，我们使用字符串时一般用`String a = "a"`这种直接赋值的形式，而不会使用`String a = new String("a")`这种方式,下面通过对一小段测试代码来看看两种方式的区别。

测试代码：
```java
public class Test {

    public static void main(String[] args) {
        String a = "a";
        String b = new String("b");
        System.out.println(a);
        System.out.println(b);
    }
}
```

通过下面两个命令对测试代码进行编译和反汇编

* javac Test.java
* javap -c Test 

得到如下输出：

```
警告: 二进制文件Test包含com.chenfeikun.Test
Compiled from "Test.java"
public class com.chenfeikun.Test {
  // 省略

  public static void main(java.lang.String[]);
    Code:
// 0和2对应的是String a = "a"
       0: ldc           #2                  // String a
       2: astore_1
// 3到12对应的是String b = new String("b")
// 可以明显的看出，使用构造函数的话，会先构造一个字符串对象
// 然后还有一些其他操作，需要执行的指令更多
       3: new           #3                  // class java/lang/String
       6: dup
       7: ldc           #4                  // String b
       9: invokespecial #5                  // Method java/lang/String."<init>":(Ljava/lang/String;)V
      12: astore_2

// 两个System.out没有区别
      13: getstatic     #6                  // Field java/lang/System.out:Ljava/io/PrintStream;
      16: aload_1
      17: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      20: getstatic     #6                  // Field java/lang/System.out:Ljava/io/PrintStream;
      23: aload_2
      24: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      27: return
}
```

从上面的汇编代码可以看出，使用`String a = "a"`是一种更加高效的用法，可以减少很多汇编指令，具体是如何实现"aa"这样的字符串到char[]数组的转换，应该跟编译器相关，超出我的能力范围了。

其他方法我看了下，感觉没有什么好讲的，都是一些对数组的操作。除此之外还有一个`intern`方法，是个native方法，可以将字符串加入字符串池，可以对一些字面量相同的字符串进行复用。

### Integer

Java的集合只能存储对象，所以像int, float, long等基础类型无法直接存入集合中，因此出现了一些包装类，同时通过自动的拆包装包来让我们更方便的使用基础类型。

包装类也都很简单，只有一个唯一的成员变量，就是它们所对应的基础类型。包装类也提供了很多的方法来方便我们操作基础类型，主要就是几个基础类型之间相互转换。

```java
// 首先看下类定义，基础自Number类，实现类Comparable接口
public final class Integer extends Number implements Comparable<Integer> {}
// Number是个抽象类，定义了获取6种基础类型的数值的方法
// 所有的数字类都继承了Number,包括AtomiclLong等
public abstract class Number implements java.io.Serializable {
    public abstract int intValue();

    public abstract long longValue();

    public abstract float floatValue();

    public abstract double doubleValue();

    public byte byteValue() {
        return (byte)intValue();
    }

    public short shortValue() {
        return (short)intValue();
    }
}
```

包装类中的常量定义：

```java
// Integer相关定义
// -2的31次方
@Native public static final int   MIN_VALUE = 0x80000000;

// 2的31次方-1
@Native public static final int   MAX_VALUE = 0x7fffffff;

// 包装类对应的基础类型
@SuppressWarnings("unchecked")
public static final Class<Integer>  TYPE = (Class<Integer>) Class.getPrimitiveClass("int");
```

其他几个包装类会根据数据类型的不同，常量的定义有些区别。

#### Cache

大多数包装类有一个缓存机制(粗略看了下，像Float,Double这两个浮点数没有Cache机制)，会缓存一定范围内的数据供重复使用，这些范围可以通过jvm参数修改，也可以使用默认值。下面还是拿`Integer`的源码举例。

```java
// IntegerCache是个内部私有类，代码很短
private static class IntegerCache {
    static final int low = -128;//缓存范围下限
    static final int high; // 上限
    static final Integer cache[]; // 缓存的数据

    static {
// 上限默认是127
        int h = 127;
// 也可以运行时通过java.lang.Integer.IntegerCache.high参数指定
        String integerCacheHighPropValue =
            sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
        if (integerCacheHighPropValue != null) {
            try {
                int i = parseInt(integerCacheHighPropValue);
    // jvm参数只有大于127时才有意义
                i = Math.max(i, 127);
    // 数组的大小最大为Integer.MAX_VALUE
                h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
            } catch( NumberFormatException nfe) {
                // If the property cannot be parsed into an int, ignore it.
            }
        }
        high = h; // 赋值给最终的上限给high
// 申请cache数组
        cache = new Integer[(high - low) + 1];
        int j = low;
        // 给cache数组赋值
        for(int k = 0; k < cache.length; k++)
            cache[k] = new Integer(j++);

        assert IntegerCache.high >= 127;
    }
    private IntegerCache() {}
}
```

上面的代码很简单，申请了一个缓存数组，把[-128，127]的数据先放到缓存中，下面看下哪里调用了缓存的这些方法。

```java
public static Integer valueOf(int i) {
    // 如果i在缓存中，则返回缓存中的数据
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    // 否则生成一个新的包装类对象并返回
    return new Integer(i);
}
```

### 往期回顾

* [Java动态代理原理剖析(一)](https://cfk1996.github.io/2019/02/28/Java%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86%E5%8E%9F%E7%90%86%E5%89%96%E6%9E%90/)

* [Java线程池源码](https://cfk1996.github.io/2019/05/02/Java%E7%BA%BF%E7%A8%8B%E6%B1%A0%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)

* [jdk序列化与反序列化底层机制](https://cfk1996.github.io/2020/06/13/jdk%E5%BA%8F%E5%88%97%E5%8C%96%E4%B8%8E%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%BA%95%E5%B1%82%E6%9C%BA%E5%88%B6/)

### 欢迎关注我的公众号

欢迎关注我的公众号，会经常分享一些技术文章和生活随笔～

![飞坤吖](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/wechat.jpg)
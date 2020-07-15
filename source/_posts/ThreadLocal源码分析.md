---
title: ThreadLocal源码分析
date: 2020-07-12 10:59:45
tags:
    - Java
    - Java源码
    - Java基础
---

今天来看一下`ThreadLocal`的源码，本文将按照`ThreadLocal`的适用场景->使用方法->猜测其实现方法->方法的源码这样的思路来撰写。

### 适用场景

`ThreadLocal`主要的适用场景如下：
1. 变量需要在多个线程中隔离(每个线程都需要一个单独的变量)
2. 变量需要在很多个方法中使用到

比如对于我现在开发的游戏服务器来说，使用线程池来处理玩家的请求，那么玩家的信息就适合用<!--more-->`ThreadLocal`来保存。不同线程间的玩家是不一样的，而且整个复杂的处理流程中，很多的方法都需要使用到玩家的信息。如果不使用`ThreadLocal`来保存玩家信息，那么玩家信息这个对象就需要作为所有方法的一个参数，在方法调用时传入，使代码更冗余。

下面写一个demo更好的展示这种场景下，两种不同方式呈现出来的代码差异。

```java
// 不使用threadLocal
public class GameHandler {

    public int getLevel(Player player) {
        return player.getLevel();
    }

    public Equip getEquipment(Player player) {
        return player.getEquip();
    }
    
    public GopAnimal getGodAnimal(Player player) {
        return player.getGodAnimal();
    }


    public static void main(String[] args) {
        new Thread(() -> {
            GameHandler gameHandler = new GameHandler();
            Player player = getPlayer(id);// 玩家信息对象
            // 假设执行战斗逻辑，需要很多玩家的信息
            gameHandler.getLevel(player);// 等级
            gameHandler.getEquipment(player); // 装备
            gameHandler.getGodAnimal(player);// 神兽系统
            // ... 其他一堆玩家相关的内容
        }).start();
    }
}
```

这种方法看着也很清晰，每个方法都需要玩家的信息，所以把玩家信息作为参数全部传入，但是会导致参数太多，代码不够简洁。特别是有时候某个方法大多数的逻辑都不需要玩家信息，但是在其中调用了一个要玩家信息的方法，就需要在最外层把玩家信息传入，导致了一连串的修改。

下面看下`ThreadLocal`如何简化上面的代码。

```java

public class GameHandler {

    public static ThreadLocal<Player> playerThreadLocal =
            new ThreadLocal<>();

    public int getLevel() {
        // 需要player对象的地方，全部通过threadLocal获取
        return playerThreadLocal.get().getLevel();
}

    public Equip getEquipment() {
        return playerThreadLocal.get().getEquip();
    }

    public GopAnimal getGodAnimal() {
        return playerThreadLocal.get().getGodAnimal();
    }

    public static ThreadLocal<Player> getPlayerThreadLocal() {
        return playerThreadLocal;
    }

    public static void main(String[] args) {
        new Thread(() -> {
            GameHandler gameHandler = new GameHandler();
            // 将player信息放入threadLocal中
            Player player = getPlayer(id);// 玩家信息对象
            GameHandler.getPlayerThreadLocal().set(player）;
            // 执行方法调用时不再需要将player信息作为参数传入
            gameHandler.getLevel();
            gameHandler.getEquipment();
            gameHandler.getGodAnimal();
            // ...
        }).start();
    }
}
```

从上面的例子可以看出，`ThreadLocal`一般是个static变量，因为`ThreadLocal`本身是在多个线程间共享的，只是它保存的值在每个线程间是独立的。然后可以方便的通过`set`方法进行赋值，通过`get`方法获取线程独有的变量。

### 自己如何实现`ThreadLocal`

如果让我们自己设计`ThreadLocal`的实现的话，可以很容易的想到一个方法。即用一个map来存储数据，线程的id作为key，保存的数据对象作为value。因为可能存在多个线程操作这个map,所以map需要是线程安全的(比如concurrentHashMap)。之前分析ConcurrentMap时知道它是通过加锁来保证线程安全的，但是我们想要保存的信息(上面demo中的玩家信息)其实是线程私有的，不会有多个线程访问同一个变量，本身就是线程安全的，那么每次获取对象时还得经过map的一道锁，是不是不太合适呢？

存储和获取数据前面我们已经很轻松的搞定了，但是还需要考虑一件事，前面的示例代码中是启动了一个线程来处理请求，每个线程都会往threadLocal中放入一个玩家信息对象，线程退出时这些变量其实就没用了，但是因为`ThreadLocal`还在，这些对象还有引用，如何把他们回收呢？是通过在线程退出时调用map的remove操作还是可以通过其他什么办法呢？

带着这些疑问，下面看下jdk的`ThreadLocal`是如何解决上面这些问题的。

### 源码分析

首先看下构造函数

```java
// 支持泛型
public class ThreadLocal<T> {
    // 构造函数啥也不做
    public ThreadLocal() {
    }

    // 还有一种可以设置默认值的构造方法
    // 在get方法为null时返回suppier产生的默认值
    // 具体逻辑后面再看
    public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
        return new SuppliedThreadLocal<>(supplier);
    }
// 一些成员变量用到的时候再介绍
}
```

按照使用步骤来看，我们使用的时候初始化完`ThreadLocal`后，会在线程最开始时使用`set`方法将需要在上下文传递的对象存入，`set`方法如下：

```java
public void set(T value) {
    // 首先获取当前线程
    Thread t = Thread.currentThread();
    // 获取线程t对应的ThreadLocalMap对象
    // ThreadLocalMap是一个map对象(ThreadLocal, value)
    // 因为我们可能需要在线程上下文中存多个对象(玩家信息，session信息，dbsession等)，需要多个ThreadLocal
    // 所以这个map与我们上面猜测的实现方式中的map不是一个东西
    // 我们猜测的是ThreadLocal自己有个<Thread, value>的map
    // ThreadLocalMap具体实现后面分析
    ThreadLocalMap map = getMap(t);
    // 根据ThreadLocalMap是否存在决定如何放入对象
    if (map != null)
        map.set(this, value); // 下面会介绍这个方法
    else
        createMap(t, value);
}

// 看下getMap方法
ThreadLocalMap getMap(Thread t) {
    // 返回的是线程的内部变量threadLocals
    return t.threadLocals;
}
// 看下threadLocals成员变量的定义(在Thread类中)
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
// 源码中的注释清晰的表明这个变量是由ThreadLocal类控制的，如果没有ThreadLocal，
// 这个变量就为null
ThreadLocal.ThreadLocalMap threadLocals = null;
```

这里先回顾下前面我们假设的实现，用map来存储变量，thread作为key。源码的实现与我们的假设有些出入，对象是每个线程的私有变量，这样访问对象时就不需要经过map的一把锁，可以实现更快的访问速度。如果按照我们的假设，源码的get方法可能是这样的：

```java
//ThreadLocal类
// private Map cocurrentMap = new ConcurrentHashMap();
ThreadLocalMap getMap(Thread t) {
    return concurrentMap.get(t.getId());
}
// 我们的假设中很明显的又多了一层map的封装，但其实是没必要的
// 但是JDK的这种会引入另外一个问题，(内存泄漏，后面介绍)
```

#### ThreadLocalMap

上面的set方法用到了`ThreadLocalMap`对象，看下这个对象相关的代码

```java
// 当线程的ThreadLocalMap对象为null时，先创建并设置第一个值
void createMap(Thread t, T firstValue) {
    // ThreadLocalMap是以ThreadLocal作为key的
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
// 构造函数
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    // 初始化大小为16的Entry数组
    table = new Entry[INITIAL_CAPACITY];//INITIAL_CAPACITY=16
    // 计算hash值
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    // 将对象放入对应的下标中
    table[i] = new Entry(firstKey, firstValue);
    // 大小设为1
    size = 1;
    // 扩容阈值为16/2*3
    setThreshold(INITIAL_CAPACITY);
}
// map共性的内容(底层数组，如何扩容，hash冲突等)就不多介绍了，以前的源码介绍中有提到过，可以看一看
// 看下Entry对象
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k); // ThreadLocal作为key是弱引用对象
        value = v;
    }
}
```

#### 内存泄漏问题

前面提到了，将`ThreadLocalMap`放入`Thread`对象中会发生内存泄漏的问题，场景是这样的：假设这个线程一直存在，那么线程内部的map因为`t.threadLocals`这个强引用存在，所以只要线程不消失，map就一直保持着对threadLocal对象的强引用。这时候即使`ThreadLocal`对象在其他地方已经没有强引用了，也会因为线程的存在而无法被回收，从而导致了内存泄漏的问题。

解决这个内存泄漏的奥秘就在Entry的源码上，Entry与HashMap中的Entry不同的地方是，这里的Key是弱引用，弱引用对象在gc发生时如果没有其他强引用指向该对象，该对象就会被回收。gc发生时，如果其他地方还有对ThreadLocal的强引用，ThreadLocal对象不会被回收，但是像上面提到的那种场景下，假设其他地方已经没有对ThreadLocal对象的强引用了，threadLocal对象就会被回收，entry.key就会变为Null，ThreadLocal对象就不会发生内存泄漏。

故事到这还没有结束，虽然ThreadLocal对象不会发生内存泄漏，能够正常被回收，但是别忘了Entry中的value对象，value对象是一个强引用，无法被回收，但是threadlocal被回收后entry.key变成null又会导致value无法被访问，从而又让value产生了内存泄漏。

value的引用关系如下：线程t -> t.threadLocals成员变量(map对象) -> map.table(entry数组) ->Entry对象 -> value. 这些全是强引用，所以线程不消失，value对象就不会被回收(即使value因为key为null访问不到，但是entry对象还在table数组上)。

这个问题将在set方法中被解决，下面一起看下set方法中的另一个分支。

```java
// 回忆一下上面的threadLocal.set()方法
if (map != null)
    map.set(this, value); // 还未分析
else
    createMap(t, value); // 已经分析
```

下面要介绍的就是`map.set(this, value)`方法

```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table; // map底层存储结构，数组
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1); // 计算下标
    for (Entry e = tab[i]; // i对应的下标不为null则进入for循环
        e != null;
        // nextIndex(i+1)用来解决hash冲突，属于map的特性，不过多介绍
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get(); // 弱引用，可能为null
        // 如果key还存在，更新value
        if (k == key) {
            e.value = value;
            return;
        }
        // 因为外层循环保证了e!=null
        // 如果key为null,则说明发生过gc
        // 且该下标处曾经设置过entry对象，只是key被回收了
        if (k == null) {
            // 方法名字为替换旧entry，即上面提到的会发生内存泄漏的value对象
            // 方法后面分析
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    // i对应的下标为null,说明该位置没有赋过值
    // 直接将entry放在下标i处
    tab[i] = new Entry(key, value);
    int sz = ++size; // size+1
    // 重哈希
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}

// replaceStaleEntry方法分析
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    // slotExpunge用来标记需要清理的节点的较小值
    int slotToExpunge = staleSlot;
    // 找到hash冲突的前一个不为null的entry
    // 如果弱引用被gc回收了，用slotToExpunge记录下坐标
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;

    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        if (k == key) {// 如果存在旧的值
            e.value = value;//更新value

            tab[i] = tab[staleSlot];//staleSlot.key==null，是需要被清理的
            tab[staleSlot] = e;// 更新e的下标到staleSlot下标处

            //如果相等，staleSlot已经被更新了，不需要清理
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;//所以slotToExpunge应该指向i,前面把i对于的entry替换了，是需要清理的对象
            // 这个方法进行一些清理操作，后面分析
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }
        if (k == null && slotToExpunge == staleSlot)
        // 如果没找到对应的key，并且staleSlot就是第一个key为null的下标
        // 则staleSlot在后面会被赋值，所以slotToExpunge要指向hash冲突的下一个坐标i
            slotToExpunge = i;
    }
    // 没找到旧值，则设置新的对象
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);
    // 进行一些清理操作
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}

// expungeStaleEntry方法，回收"内存泄漏"的对象，并优化hash分布
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    // 置为null是java中常用的删除引用的方式
    // 因此value没有强引用，entry也没有，都可以被回收
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    Entry e;
    int i;
    // 继续往后寻找可以回收的节点，或者重hash
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        // k==null找到的都是可以应该被回收的节点
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            // 如果不能被回收，重新计算hash值，放入正确的地方
            int h = k.threadLocalHashCode & (len - 1);
            // h!=i说明之前因为hash冲突位置向后移了
            // 现在gc后可能被回收出现了新的空间，尝试向前移动
            if (h != i) {
                tab[i] = null;
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    // i是tab[i]=null的第一个下标（在staleSlot之后）
    return i;
}

// cleanSomeSlots方法
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        // 传入的i是tab[i]==null
        // 所以直接取下一个
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    // 以i = i/2的方式取尝试回收engtry
    return removed;
}
```

上面这段代码有些多了。。。但大概意思就是set方法时会去找key对应的坐标，如果有就更新，如果没有就新建对象，同时还会`尽可能多`的删除entry.key已经被gc过的entry对象，然后还混杂着一些重hash的过程(重hash属于map相关，可以去网上了解下)
简单的说就是即会解决内存泄漏的问题，也会提高清除无用对象的效率，同时还要优化hash分布。

set相关源码及内存泄漏的问题在上面都分析了，那么关于`ThreadLocal`的难点也就都解决了，下面就看些剩余的轻松的代码吧。

#### get()方法

启动线程后首先是往`ThreadLocal`中set上下文中传递的对象，然后在需要的方法中通过get方法来获取对象，来看下get方法的源码吧

```java
public T get() {
    // 获取线程
    Thread t = Thread.currentThread();
    // 获取线程的成员变量ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // 如果map不为空，则返回对应的value
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    // 该线程第一次访问get，map为空，返回生成的初始值
    return setInitialValue();
}

// map.getEntry方法
private Entry getEntry(ThreadLocal<?> key) {
    // 计算下标
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
    // entry存在，且key相等
        return e;
    else
    // entry不存在，或者entry.key被gc回收了
        return getEntryAfterMiss(key, i, e);
}

// getEntryAfterMiss方法
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;
    // 因为可能发生hash冲突，所以向后遍历直到e==null
    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
        // 找到了，这种就是hash冲突导致的向后移动了
            return e;
        if (k == null)
        // 顺便回收一下无用的entry
            expungeStaleEntry(i);
        else
        // 遍历下一个坐标对应的entry
            i = nextIndex(i, len);
        e = tab[i];
    }
    // 找不到就返回null
    return null;
}

//setInitialValue方法，第一次调用线程的threadLocal时调用该方法
private T setInitialValue() {
    // initialValue方法默认返回null
    // 但是创建ThreadLocal对象时可以传入一个可以产生初始值的方法，后面分析
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    // 创建ThreadLocalMap对象，前面提到过，Thread的localMap变量是由ThreadLocal对象来控制的
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```

### 带默认值的ThreadLocal

最开始介绍构造函数时介绍了一种可以产生初始值的构造方法，再来回顾下。

```java
// 返回了一个继承了ThreadLocal的子类SuppliedThreadLocal
public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
    return new SuppliedThreadLocal<>(supplier);
}

// SuppliedThreadLocal源码
static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {

    private final Supplier<? extends T> supplier;

    SuppliedThreadLocal(Supplier<? extends T> supplier) {
        this.supplier = Objects.requireNonNull(supplier);
    }
    // 重写了ThreadLocal默认返回null的initialValue方法
    @Override
    protected T initialValue() {
        return supplier.get();
    }
}
```

### 线程池

前面提到用线程池来处理玩家请求，所以线程是不会被销毁的，那么在处理请求之后，记得删除玩家信息哟

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
```

### 线程退出

文章最开始抛出个问题，如果线程结束了，但是ThreadLocal对象还在会有什么问题吗。其实不会，通过分析，可以知道map信息是放在线程里的，ThreadLocal对象本身不存储任何数据。


### 总结

到这里`ThreadLocal`的源码就已经全部分析完了，小小的总结一下。

- 使用场景
    1. 变量需要在多个线程中隔离(每个线程都需要一个单独的变量)
    2. 变量需要在很多个方法中适用到
- 实现方式
    1. Thread对象存储<ThreadLocal, Objecet>这样的ThreadLocalMap对象
    2. `ThreadLocal.get()`转化为`Thread.getLocalMap.get(threadLocal)`
- 内存泄漏解决办法
    1. ThreadLocalMap底层保存的对象的key是弱引用
    2. ThreadLocal读取和写入时尝试删除key=null的entry对象

### 往期回顾

* [Java动态代理原理剖析(一)](https://cfk1996.github.io/2019/02/28/Java%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86%E5%8E%9F%E7%90%86%E5%89%96%E6%9E%90/)

* [Java集合源码系列-HashMap](https://cfk1996.github.io/2019/01/12/Java%E9%9B%86%E5%90%88%E6%BA%90%E7%A0%81%E7%B3%BB%E5%88%97-HashMap/)


* [jdk序列化与反序列化底层机制](https://cfk1996.github.io/2020/06/13/jdk%E5%BA%8F%E5%88%97%E5%8C%96%E4%B8%8E%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%BA%95%E5%B1%82%E6%9C%BA%E5%88%B6/)

### 欢迎关注我的公众号

欢迎关注我的公众号，会经常分享一些技术文章和生活随笔～

![飞坤吖](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/wechat.jpg)
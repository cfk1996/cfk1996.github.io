---
title: Java集合源码系列-HashMap
date: 2019-01-12 11:05:57
tags:
    - Java
    - Java源码
---

### 前言

Java基础是面试中的重中之重，尤其是对集合类，并发包等源码的相关的问题是面试中常考察的点。但是其实这些点并不难，对于计算机专业的学生来说，如果上过数据结构这门课，对数组，字符串，链表以及字典的大致实现都有一定了解，在阅读JDK源码时也会发现，整体的思路是和所学的数据结构一样的，所以理解源码的整体框架不难，但是不能总是浮于表面，需要理解JDK在一些细节处理上的针对性优化措施。
<!--more-->
网上关于源码解析的博客有很多，也给我提供了很大的帮助。阅读源码时不懂是很正常的情况，可以借助google或baidu来解决遇到的问题。看了那么多博客，自己也想总结出一个源码系列，一是因为基础知识很重要，但是容易遗忘，做一个定期的总结可以让自己将来有机会温习，同时加深自己的理解。二是做一个分享，希望可以收获更多的读者_(:3」∠)_

作为系列的第一篇文章，说一下文章的整体思路。因为是介绍集合类的源码，所以以使用集合的整个生命周期来作为行文路线，即创建对象-->添加元素-->获取元素-->删除元素。 整个系列，如果没有特殊说明，则都是JDK8版源码.

### 概览

Map的底层存储一般都是数组，因为数组可以满足常数时间内进行**get**和**put**操作。而数组的下标一般是由一个哈希函数来获得，在常见的数据结构教材中，一般为对数组长度取模，可以保证哈希函数得到的index范围在[0，len-1]之间，不会数组越界，并且可以利用到数组的所有位置。除了取模这种hash算法，还有很多其他的算法，各有优劣。

除了hash算法，一个Map实现还需要解决index重复的问题，常见的有再hash法，开放地址法，拉链法等；而HashMap可以看做是一个优化过的拉链法。

解决了如何计算hash和解决冲突后，还需要面临的一个问题是，数组的大小都是固定的，那么随着Map存的数据越来越多，同一个下标存储的数据越来越多，存取的性能退化为O(n),数组如何动态的调整其大小来改善性能呢;HashMap通过一个叫做负载因子的变量来控制调整的时机。

带着这几个问题，下面一起来阅读一下HashMap的源码吧～

### 构造函数

使用集合类的第一步就是通过构造函数，HashMap的构造函数有多个，但其实类似，将从最常见的展开，顺带着其余的构造函数。

```java
/**
 * loadFactor即为前面提到的负载因子，默认值为0.75，这是最普通的、最常用的构造函数，所有内部参数都使用默认值，关于一些参数，后面用到时再详细介绍
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f; // 负载因子默认值
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
}
/**
 * 指定构建对象的底层数组大小和负载因子。默认值分别为16和0.75.对传入的参数有一点限制，底层数组必须大于0小于最大容量(2的30次方)，并且会被调整为2的整数幂，原因后续介绍。通过tableSizeFor函数调整。
 */
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
static final int MAXIMUM_CAPACITY = 1 << 30;
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " + 
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}
//通过巧妙的位运算，将指定的cap变为大于等于cap的第一个2的幂次方整数，源码中多次利用位运算来进行速度优化
// n =       000..1XXXXXX假设n的二进制格式，1代表第一个1出现的位置，x为任意的0或1
// n >>> 1 = 000..01xxxxx
// 新n     = 000..11xxxxx 
// n >>> 2 = 000..0011xxx
// 新n或运算 = 000..1111xxx
// 因为cap传进来时最大为2的30次方，所以最多只要16的时候，就可以保证第一次出现1的位置后面全变为1，即n = 000..00111111，那么在返回时通过n+1的操作，使n=000.01000000的形式，保证返回值是2的n次方
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
// 用一个已经存在的map初始化，用的不多，就在代码中稍微写点注释
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) {
        if (table == null) { // 如果数组还没分配，根据m的大小建立合适大小的数组
            float ft = ((float)s / loadFactor) + 1.0F;
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                (int)ft : MAXIMUM_CAPACITY);
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        else if (s > threshold) // 如果m的大小超过table大小，则扩容，resize()后面介绍
            resize();
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```

### 添加元素

介绍完构造函数后现在介绍下如何添加元素，那么在这之前，需要先了解一下HashMap用了什么结构来保存数据

```java
transient Node<K,V>[] table; // HashMap中的table属性用来保存数组，数组元素本身的结构为Node,是一个静态内部类
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash; // 存储hash值，空间换时间，减少重复计算
    final K key;    // 字典的key
    V value;        // 字典的value
    Node<K,V> next; // 前面提到,HashMap使用优化过的拉链法解决Hash冲突，即所有Hash值相同的key通过链表连接，可以看出极端情况get性能会退回为O(N)，所以后面会介绍如何优化，暂时理解为一个简单的链表

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
    // 省略get, set, equal, toString....
}
```

接下来看下put方法的逻辑：

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true); // hash方法求key的index
}
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16); // 充分利用高16位以及低16位求hash，避免连续的key产生的hash值具有规律性
}
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0) // table懒加载，第一次存值时才初始化数组
        n = (tab = resize()).length; // 初始化数组，分配给tab变量，resize()后面介绍，返回值为新分配的数组，n为新数组长度
    if ((p = tab[i = (n - 1) & hash]) == null) // i = (n-1) & hash求出元素下标，前面介绍了字典一般是通过对数组的大小去取模求得数组的下标，但是cpu更擅长的是位运算，而前面提到了数组的大小肯定是2的整数次方，即n的二进制格式为00.100..000，那么n-1即为00.011..111的,对这样的一个数进行与运算的结果集就是0到n-1，效果与除法取模一样，但是速度大幅度提升
        tab[i] = newNode(hash, key, value, null); // 如果指定下标不存在元素，在下标为i的地方插入新值, newNode方法生成一个Node对象并返回
    else { // 如果指定下标已经存在元素，进入该分支
        Node<K,V> e; K k;
        // p为指定下标i位置上的Node元素
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k)))) // 如果p指向的节点的key与新put进来的key一样
            e = p;
        else if (p instanceof TreeNode) // 如果p指向的节点的key与新Put进来的key不一样，且是TreeNode实例
            // 前面介绍过，HashMap的拉链法是优化过的，就体现在这。如果同一个数组跟着的链表长度大于8，会转化为红黑树，红黑树是一种平衡树，平均复杂度为O(logn),优于链表的O(n)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value); // TreeNode版的putVal方法
        else { // key应该在链表中的情况
            for (int binCount = 0; ; ++binCount) { // 遍历链表
                if ((e = p.next) == null) { // 如果遍历完链表还没找到相同的key，则key为新的值，添加到链表尾端
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // 如果节点数大于8，则转换为红黑树结构 TREEIFY_THRESHOLD=8
                        treeifyBin(tab, hash); // 将链表转换为红黑树
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k)))) // put进来的key已经存在，指向Node e
                    break;
                p = e;
            }
        }
        if (e != null) { // put 进来的key已经存在，将value替换
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null) // @param onlyIfAbsent if true, don't change existing value
                e.value = value;
            afterNodeAccess(e); // 方法为空，无操作
            return oldValue;
        }
    }
    // 如果key是新的值，则进入这里，否则直接返回
    ++modCount; // 改变量记录了对map添加和删除的次数
    if (++size > threshold) // threshold为扩容门槛，如果没有指定，在第一次进行resize()时被置为capacity * load factor，即16*0.75
        resize(); // size记录了table包含的key,value对的数量
    afterNodeInsertion(evict); // 改方法为空，无操作
    return null;
}
```

上面就是关于添加元素的部分了，但是对于优化为红黑树的部分没有进行深入介绍。因为红黑树本身的数据结构就比较复杂，不易理解，对红黑树进行插入，平衡，查找的相关操作跟map的关系已经不大，如果有兴趣，可以另外找时间网上搜索一下。需要知道的就是红黑树主要解决的问题是当哈希冲突太多时，链表的查询性能变为O(n),用红黑树替代后可以降低到O(logn)的复杂度，且当链表长度8时才进行这个转变；后面会提到resize()时如果节点数量小于6时，红黑树也会退化为链表。接下来介绍一下前面省略掉的`resize()`方法：

#### resize()方法

```java
final Node<K,V>[] resize() { // size超过capacity * load_factory时进行扩容
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) { // 如果已经达到最大的size：2的30次方，不再扩容，直接返回旧table
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY) // 如果旧的容量*2小于2的30次方且旧的容量大于16时将数组大小翻倍
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // table数组还没创建且threshold>0时，将capacity设为threshold
        newCap = oldThr;
    else {               // table数组还没创建，将capacity设为16，扩容阈值设为16*0.75
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) { // 设置新的扩容阈值
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr; // threshold重新赋值
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap]; // 申请一个新的数组，大小为原来数组的两倍
    table = newTab;  // table变量指向新数组
    if (oldTab != null) { 
        for (int j = 0; j < oldCap; ++j) { // 遍历旧的数组
            Node<K,V> e;
            if ((e = oldTab[j]) != null) { // 旧数组对应的下标j不为空时，将该下标的元素转移到新数组中
                oldTab[j] = null; // 加速GC？猜测
                if (e.next == null) // 如果对应的下标j只有一个元素，将该元素放到新数组 e.hash & (newCap - 1)处
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode) // 如果下标j对应的Node是红黑树节点
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap); // 在split函数中，如果节点数量小于static final int UNTREEIFY_THRESHOLD = 6；退化为链表
                else { // 链表Node
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do { // do-while循环，将链表根据重新e.hash & oldCap是否等于0，分成两个子链表
                        next = e.next;
                        if ((e.hash & oldCap) == 0) { // oldCap二进制为00.100.00的格式，是2的整数次方，只有1个1
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 按照新的table大小来计算数组下标时，算式应该是这样的：e.hash & (oldCap*2-1),假设oldCap为16， 则二进制表示为10,000，oldCap*2=32的二进制为100,000
                    // 那么上面do-while循环中(e.hash & oldCap) == 0的分支，hash值格式为xx..xx0x,xxx(从右往左第5个数是0，其他任意),与（32-1）做&运算的以及与（15-1）做&运算的结果分别是：
                    // （32-1）xx.xx0x,xxx ---- (16-1)xx.xx0x,xxx
                    // （32-1）0000011,111 ---- (16-1)0000001,111
                    // 结果:-- 000000x,xxx ---- (16-1)000000x,xxx;可以看出扩容前后下标的计算值是一样的，所以在新数组同样的下标处插入这个链表
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    } 
                    // 上面do-while循环中(e.hash & oldCap) != 0的分支，hash值格式为xx..xx1x,xxx(oldCap还是16，从右往左第5个数是1，其他任意)，与（32-1）以及（15-1）做&运算的结果分别是：
                    // （32-1）xx.xx1x,xxx ---- (16-1)xx.xx1x,xxx
                    // （32-1）0000011,111 ---- (16-1)0000001,111
                    // 结果:-- 000001x,xxx ---- (16-1)000000x,xxx;可以看出扩容后新的下标值比就的下标值大了16，也就是大了oldCap的值，所以另外一个子链表被放置到新数组j+oldCap下标处，这种巧妙的计算下标方式，可以减少重新计算下标的花销，加快扩容的速度
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

#### TreeNode
`TreeNode`的类大概就是这样，很清楚的表明了是个红黑树结构，它的插入，删除，查找等方法被省略了，关于它如果左旋右旋进行rebalance可以去网上搜索相关资料
```java
 static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
    TreeNode(int hash, K key, V val, Node<K,V> next) {
        super(hash, key, val, next);
    }
```

### 获取元素

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) { // table数组中已经存在元素且hash对应的下标处有元素
        if (first.hash == hash && // 
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first; // table数组中hash对应下标的第一个元素，就是要找的元素时返回first
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key); // 如果hash对应的node是一个红黑树结构，从红黑树中查找key
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e; // 在链表中查找key相等的Node
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

### 删除元素

```java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) { // tab = table; p指向key对应的下标所在的Node，如果tab,p为空，return null
        Node<K,V> node = null, e; K k; V v;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p; // key指向的Node的第一个元素就是要remove的key时，node=p
        else if ((e = p.next) != null) { // 第一个元素存在，但不是所要找的key时，如果p.next存在，进入
            if (p instanceof TreeNode) // 该下标的Node为红黑树结构
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else { // 该下标的Node为链表，找到对应的key时，node指向key对应的节点，否则node=null
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) { // matchValue可选，删除时比较value是否相等，默认不比较
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p) // key对应链表第一个节点
                tab[index] = node.next;
            else // key对应链表非第一个节点
                p.next = node.next;
            ++modCount; // modCount记录添加和删除等操作的次数，不记录修改value的次数
            --size; // size大小减1
            afterNodeRemoval(node); // 方法为空
            return node;
        }
    }
    return null;
}
// 清空map
public void clear() {
    Node<K,V>[] tab;
    modCount++;
    if ((tab = table) != null && size > 0) {
        size = 0;
        for (int i = 0; i < tab.length; ++i)
            tab[i] = null;
    }
}
```

### for-each遍历

```java
@Override
public void forEach(BiConsumer<? super K, ? super V> action) {
    Node<K,V>[] tab;
    if (action == null)
        throw new NullPointerException();
    if (size > 0 && (tab = table) != null) {
        int mc = modCount;
        for (int i = 0; i < tab.length; ++i) { // 按照数组下标从小到大遍历
            for (Node<K,V> e = tab[i]; e != null; e = e.next)
                action.accept(e.key, e.value);
        }
        if (modCount != mc) // 遍历的时候不允许修改数组结构，修改value除外，如果出现，抛出异常
            throw new ConcurrentModificationException();
    }
}
// example
maps.forEach((k, v) -> {
    System.out.println("key: " + k + ", value: " + v); 
})
```

### 总结

上面差不多把HashMap常用的方法的源码都介绍了一下，其实还不到源码的一半的内容，源码中还有很多关于TreeNode的操作，红黑树相关的操作占了将近一半的篇幅，除此之外还有很多Iterator的类和方法，对于这些Iterator的源码和作用，还没有去研究过，准备把大部分集合代码分析完后，专门写一篇关于迭代器的分析博客。(水平有限，如有错误，欢迎提出，可以发邮件或者关注文章底部的微信公众号，谢谢！)

## Technique and Life

欢迎关注我的公众号，会经常分享一些技术文章和生活随笔～

![Technique and Life](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/wechat.jpg)

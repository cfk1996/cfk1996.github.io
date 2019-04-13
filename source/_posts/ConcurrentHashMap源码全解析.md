---
title: ConcurrentHashMap源码全解析
date: 2019-04-11 16:43:00
tags:
    - Java
    - Java源码
---

这篇文章将解析`ConcurrentHashMap`的源码，`ConcurrentHashMap`是一个支持并发检索及高并发性的哈希表实现。包含了与`HashTable`方法相对应的所有实现，但是又比`HashTable`高效。

一般来说同步操作是通过锁来保证线程安全，那么提高并发性有三种优化方式，第一种是对不需要同步的方法不加锁;第二种是缩小加锁的粒度;第三种是通过CAS,即尝试通过不加锁实现同步。对于CAS不理解的，可以查阅相关资料了解下，对阅读本文和`ConcurrentHashMap`有帮助.

<!--more-->


    写完博客回头写下的话：这篇篇幅较长，有耐心的人可以直接把文章仔细读完。我认为比较好的阅读方式是，自己尝试去阅读源码，遇到看不懂的方法，回到我这篇文章，搜索方法的名字，看本文是否提到了该方法，提到了的话，顺着我的思路去理解，当然我可能也是错误的，欢迎指出。如果不幸没提到，那么查阅其他资料或者根据重要程度适当跳过。文章的行文路线是构造函数->添加元素->获取元素->删除元素。
### 类、构造函数

```java
// ConcurrentHashMap继承自AbstractMap并实现了ConcurrentMap接口
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable {}

// 构造函数有多个
// 最常用的无参数构造函数，构造了一个默认table大小为16的空map
public ConcurrentHashMap() {}
//　跟之前分析过的HashMap源码一样，还有传入初始大小;
// 传入一个已经存在的map;传入负载因子等参数的构造函数，就不都放上来了 
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}
// ... 其他构造函数
```

最常用的无参构造函数没有执行任何操作，无法获得更多的信息，那么先来看一下它的部分成员变量。

```java
// Map底层数组，用来存储k,v数据
transient volatile Node<K,V>[] table;
// 内部Node类，与HashMap中的Node类相比，val和next变量加上了volatile关键字
//　同时也增加了几个方法，等用到的时候再提
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;
    Node(int hash, K key, V val, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.val = val;
        this.next = next;
    }    
// 省略一些方法
}
```

### 添加元素

按照我之前写集合类源码分析的思路，集合的生命周期在创建后就到了添加元素这一步，看看`put`方法如何实现。

```java
// put方法需要传入key和value,然后调用putVal方法
public V put(K key, V value) {
    return putVal(key, value, false);
}
// putIfAbsent,只有当key不存在时，才将value写入
// put和putIfAbsent的底层实现都是putVal()
public V putIfAbsent(K key, V value) {
    return putVal(key, value, true);
}
/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) 
        throw new NullPointerException();
    // spread方法计算出一个hash值
    int hash = spread(key.hashCode());
    int binCount = 0;
    //无限循环
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
        //无参数构造函数中的table是懒加载的，在第一次Put时初始化
        // initTable方法在后面介绍
            tab = initTable();
        //tabAt可以通过Unsafe类提供的方法，无锁进行读取table[i]的元素
        //这几个无锁的tab操作，后面介绍，这里明白其作用就行
        //如果tab[i]为null，进入该分支，i为根据hash算出的数组下标
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            //如果tabAt读取的tab[i]为null，再通过cas操作，如果tab[i]为null，
            //则在i下标处插入新的Node(hash, key, value, null)
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;
        }
        //如果tab[i]正在进行resize进入该分支，MOVED=-1
        //进行resize操作时，头节点hash值为-1
        //后面会提到如何变成-1
        else if ((fh = f.hash) == MOVED)
        　　//当前线程帮助table进行扩容操作的转移
            tab = helpTransfer(tab, f);
        //下标i处的头节点为正常Node节点，进入该分支
        else {
            V oldVal = null;
            //对头节点加锁，f=tab[i]
            synchronized (f) {
                //再次判断f仍然是tab[i]处的头节点
                if (tabAt(tab, i) == f) {
                    //头节点hash值>=0，状态正常，后续会介绍不正常时的hash值
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            //如果put进来的Key已经存在，根据onlyIfAbsent决定是否替换
                            //并用oldVal保存旧值，作为方法的返回值
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            //如果当前节点的key与put进来的key不相等
                            //e指向当前节点的Next节点
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                            //如果next节点为空，说明key不存在，插入新结点    
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    //static final int TREEBIN   = -2; // hash for roots of trees
                    //如果头节点hash值为-2,代表其为树的跟节点
                    //与HashMap中一样，如果头节点后的节点数量大于等于8,则优化为红黑树
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        //在红黑树中进行Put操作，具体不分析
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            //上面在链表中遍历节点时，会统计节点数量
            if (binCount != 0) {
                //节点数量大于等于8时，将链表优化为平衡树
                //static final int TREEIFY_THRESHOLD = 8;
                if (binCount >= TREEIFY_THRESHOLD)
                //转换为树的操作不分析了，HashMap之前尝试过了，看不懂，这次看都不想看了~
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    //如果有oldVal会直接返回，没有则意味着插入了一个新的数据
    //addCount检测是否需要扩容等操作，后面介绍
    addCount(1L, binCount);
    //put新的key,value时，put()返回null
    return null;
}
```

putVal()中调用了不少其他方法，选择其中一些有意义的再详细分析.

#### initTable()

首先看下在第一次put操作时，会进行table初始化操作

```java
// 在第一次put操作时初始化table
// sizeCtl用来控制初始化和resize，默认为０,-1表示初始化
//当它大于0时，为下一次需要resize时的阈值
private transient volatile int sizeCtl;
// Unsafe类更多用处自行查阅，可以直接操作内存
//在这里，Unsaft用来进行CAS操作，SIZECTL为sizeCtl变量的内存偏移地址
sun.misc.Unsafe　U = sun.misc.Unsafe.getUnsafe();
Class<?> k = ConcurrentHashMap.class;
SIZECTL = U.objectFieldOffset(k.getDeclaredField("sizeCtl"));
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        // sizeCtl初始为０
        // 如果小于０，则代表另外一个线程已经在初始化
        //则该线程放弃cpu，等待初始化
        if ((sc = sizeCtl) < 0)
            Thread.yield(); 
        //因为sizeCtl为０，进入else if分支
        // U.compareAndSwapInt(this, SIZECTL, sc, -1)进行的操作是
    　　 //比较sc与SIZECTL指向的内存地址的值,如果相等，将SIZECTL指向的内存设为-1
        //cas操作成功则返回true,否则False
    　　 //sizeCtl=-1时，表示正在初始化
    　　 //这些操作整体可以看做是原子的
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    //private static final int DEFAULT_CAPACITY = 16;
                    //table默认大小为16，sc=0
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    //申请node数组，并用table指向该数组
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);//sc = 12
                }
            } finally {
                sizeCtl = sc;//sizeCtl = 12，下一次扩容阈值
            }
            break;
        }
    }
    return tab;
}
```

#### tabAt(),casTabAt()

```java
// ConcurrentHashMap中提供了三个操作table数组的原子方法，这里先介绍出现的两个
ABASE = U.arrayBaseOffset(ak);//根据名字判断，是数组起始偏移地址
int scale = U.arrayIndexScale(ak);
if ((scale & (scale - 1)) != 0)
    throw new Error("data type scale not a power of two");
ASHIFT = 31 - Integer.numberOfLeadingZeros(scale);
//这里的get tab[i]操作没有加锁，但是可以保证tab[i]位置的更新操作的读可见性
//具体如果保证这些可见性的，我也不了解，可以查阅Unsafe相关资料，我暂时没找到-_-!!
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    //((long)i << ASHIFT) + ABASE就是下标i的相对偏移地址
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
//通过CAS操作判断tab[i]的值，并进行赋值，也没有进行加锁，因此并发效率高
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```

#### addCount()

putVal()方法最后，如果是插入新值的情况，会进行`addCount(1L, binCount)`操作来判断是否扩容。

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    //如果CAS修改baseCount失败或者counterCell==null进入该分支
    //CAS修改失败代表有并发的线程在修改baseCount
    //单线程第一次put时，debug发现是不进入该分支的，且b=0,s=1
    //该分支，暂时不理解，不分析
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b+ x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x)) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    //如果是在putVal中调用的，check都会大于等于０
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        //s为baseCount的新值，如果s大于sizeCtl并且
        //table不为空且table容量小于1<<30
        //则进行resize操作
        //private static final int MAXIMUM_CAPACITY = 1 << 30;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                (n = tab.length) < MAXIMUM_CAPACITY) {
            //返回用于调整大小为n的表的标记位,resizeStamp后面介绍
            //这里知道它可以反映n的大小，且n为2的幂次方
            int rs = resizeStamp(n);
            //回想之前的initTable方法把sizeCtl设为了12
            //那么sc怎么会变成小于0的呢，得看else分支
            //看完else分支，再回过头来看sc < 0的情况
            //如果你看完else分支，会发现sc小于0标识当前正在resize
            if (sc < 0) {   
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                //通过cas对进行resize的线程数+1
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            //RESIZE_STAMP_SHIFT＝16
            //通过CAS将sizeCtl设置为(rs<<16)+2
            //那么这里先告诉大家rs值为这样一个形式，为什么是这样在后面提
            //0000 0000 0000 0000 1000 0000 000x xxxx
            //那么再左移16位变成
            //1000 0000 000x xxxx 0000 0000 0000 0000的格式
            //首位为1,则sizeCtl为负数
            //并且高16位能反映出是哪个size的数组在resize
            //别忘记sc=(rs<<16)+2这一步中的+2
            //低16位值为x的话，则反映有下x-1个线程在进行resize
            //sizeCtl不仅是阈值，在小于0时还可以表示正在resize
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
    　　　　　　　//进行resize操作，把数据搬入newTable
                transfer(tab, null);
            //sumCount求出所有累计的Node数量
            s = sumCount();
        }
    }
}
//分析一下int rs = resizeStamp(n);
static final int resizeStamp(int n) {
    //n为table大小，是一个２的整数次幂，是一个形如1000 000...的数
    //Integer.numberOfLeadingZeros(n)返回前导0的个数
    //这个值可以很容易判断出n的位置
    //private static int RESIZE_STAMP_BITS = 16;
    //那么就是一个小于32的数或上1000 0000 0000 0000
　　 //即返回一个0000 0000 0000 0000 1000 0000 000x xxxx形式的数
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```

#### transfer()

`transfer`就是进行resize操作的地方，如果后续的线程需要resize,则会在不超过最大帮助线程等的前提下，一起进行resize操作，现在一起看一下`transfer方法`

```java
static final int NCPU = Runtime.getRuntime().availableProcessors();
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    //根据CPU数量划分每个线程处理桶的个数，最小为16
    //桶大小16意味着一个大小为64的数组，会被划分为[0,15][16,31][32,47][48,64]
    //每个线程每次只能选择其中一块来进行transfer操作
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; 
    //如果新的数组还没申请
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            //数组大小扩容为原来的两倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {//防止OOM错误
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        //赋值给成员变量nextTable，用于其他线程判断是否在resize
        nextTable = nextTab;
        //n为原数组大小，transfer表示需要扩容的数组大小
        transferIndex = n;
    }
    //新数组大小
    int nextn = nextTab.length;
//ForwardingNode是一个特殊的Node,用于resize时放在table头节点，hash值为-1
/**
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        super(MOVED, null, null, null);
        this.nextTable = tab;
}
*/   
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    //标识是否需要向前推进
    boolean advance = true;
    //判断resize是否完成的标志
    boolean finishing = false;
    //无限循环
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {
            int nextIndex, nextBound;
            //虽然从i=0,bound=0开始，但是下面会修改这些值
            //i从上边界开始，bound为桶的下边界
            //ｉ>bound的话说明桶中还有未resize的，需要继续推进
            //通常来说i<bound则说明该线程分配到的桶已经完成
            //在下面的循环中，会继续分配桶给该线程(如果还有桶)
            if (--i >= bound || finishing)
                advance = false;
            //没有桶了，说明resize完成了
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            //i=0,bound=0的第一次循环会进入该分支
            //通过CAS操作修改transferIndex,划分一个大小为stride的桶给当前线程
            //举个例子，扩容前数组大小64,那么transferIndex=64，stride默认最小16
            //CAS成功后，transferIndex=64-16=48
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
            //把[48,63]的resize操作分配给当前线程
            //i赋值为当前线程分配到的桶的上界64-1=63，resize操作从后向前进行
            //bound为桶的下界64-16=48
            //那么在没处理完tab[i]之前，不需要推进，则advance=false                           
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        //前面i在没有桶时会被赋值为-1，说明resize完成
        //i＝n是在该分支被赋值的，用来再次检测是否resize完成
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            //resize完成了
            if (finishing) {
                //清空临时table
                nextTable = null;
                //将新table赋值给table
                table = nextTab;
                //修改新的阈值为新数组大小的3/4
                sizeCtl = (n << 1) - (n >>> 1);
                //返回
                return;
            }
            //通过CAS对帮助resize的线程个数-1
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                //如果是帮助线程，完成resize后直接return
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                //第一个进入resize的线程，设置finishing为true
                //并把i设为n再次检测是否有遗漏的    
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        //如果tab[i]=null,不需要扩容，直接用fwd占位
        //并修改advance为true,向i--进行下一次推进
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
    　　 //tab[i]!=null,但是头节点已经是fwd,表明该下标已经resize完成
    　　//修改advance=true,向i--推进
        else if ((fh = f.hash) == MOVED)
            advance = true;
        //tab[i]!=null,并且还没resize    
        else {
            //锁住头节点，防止putVal操作
            //遗忘了的话回头看下putVal的分析，也会锁住头节点
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    //像HashMap中的一样，会把链表拆分成低位，高位两个链表
                    Node<K,V> ln, hn;
                    //fh>=0表明是链表，fwd的Hash为-1,红黑树的root节点hash为-2
                    if (fh >= 0) {
//这段比较复杂，先分析下fh&n的用处
//n为原数组的大小，是2的整数次幂,肯定形如1000 000..
//＆与运算符，都是1才为1,有一个0就是0
//那么fh&n的结果要么等于0,要么等于n，等于0的被分为低位链表中，其他的划分到高位链表    
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
//这个循环就是为了下一次循环可以少几次不必要的循环，假设现在f节点下有6个节点
//每个节点都会进行hash&n的操作，要么是0，要么是n，因此，我用0和n代表这6个节点
//(1)0-->(2)0-->(3)n-->(4)0-->(5)n-->(6)n--> null;
//经过这次循环，lastRun指向(5)，runBit=n                     
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        //因为runBit=n，所以进入该分支
                        //高位链表ln直接指向lastRun,即举例中的(5)节点
                        else {
                            hn = lastRun;
                            ln = null;
                        }
//该循环将tab[i]下链表中的节点划分为低位节点链表和高位链表
//那么由于上一个循环,可以省去lastRun节点及之后节点的遍历                        
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        //把低位链表插入新数组的下标i处
                        setTabAt(nextTab, i, ln);
                        //高位链表插入新数组的下标i+n处
                        setTabAt(nextTab, i + n, hn);
                        //在旧数组的同节点插入fwd占位，表示已经完成resize
                        setTabAt(tab, i, fwd);
                        //tab[i]resize完成，继续向i--推进
                        advance = true;
                    }
                    //针对树节点的操作，也是分成了两部分，最后还会判断拆分后需不需要回退会链表
                    //针对树节点的具体的就不分析了
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

那么到这里resize步骤都已经介绍完了，putVal方法中就只剩下一个`helpTransfer`还没分析，来简单的看下吧

#### helpTransfer()

```java
//putVal中是这样调用的
//  else if ((fh = f.hash) == MOVED)
//      tab = helpTransfer(tab, f);
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    //tab是传入的旧数组
    //如果tab[i]头节点是fwd，说明正在resize
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        //sizeCtl<0说明正在扩容
        while (nextTab == nextTable && table == tab &&
                (sc = sizeCtl) < 0) {
            //进行一系列合规判断，不重要不管了...        
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            //对sizeCtl+1然后进入transfer.这里的+1就对应了transfer相应的-1操作
            //来判断是否还有帮助线程在resize    
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

### 获取元素

添加元素部分终于结束了...现在来看看如果获取元素，这一部分阅读起来相对比较轻松和简单。

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    //求出key的hash
    int h = spread(key.hashCode());
    //如果table已经初始化，且h对应的下标(n-1)&h处的Node不为空
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        //如果头节点就是需要查找的key,返回其value    
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        //如果头节点的hash<0，那么可能是红黑树，可能是在resize
        else if (eh < 0)
            //后面分析find方法
            return (p = e.find(h, key)) != null ? p.val : null;
        //遍历链表，寻找key    
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    //如果没找到对应的key，返回null
    return null;
}

//get方法比较简单，如果是链表的情况，
//就是根据hash找到下标然后判断key是否相等
//如果是hash小于0的情况，有两种，一种是FWD节点表示在resize
//一种是-2，表示是树节点
//对于是负数的情况，Node类中有find方法，FWD node和TreeNode都重写了该方法
// find() in ForwardingNode
Node<K,V> find(int h, Object k) {
    //最外层的无限循环
    outer: for (Node<K,V>[] tab = nextTable;;) {
        Node<K,V> e; int n;
        //如果key为null，或者nextTable还未申请
        //或者nextTable[i]为null，返回null
        //这里可以看出，resize过程中，是在新table中查找
        //这是forWardingNode的find方法，当旧节点头节点为fwd时
        //新table的i和n+i处都应该赋值完成
        if (k == null || tab == null || (n = tab.length) == 0 ||
            (e = tabAt(tab, (n - 1) & h)) == null)
            return null;
        //无限循环    
        for (;;) {
            int eh; K ek;
            //判断新table的(n-1)&hash处是否与Key相等
            if ((eh = e.hash) == h &&
                ((ek = e.key) == k || (ek != null && k.equals(ek))))
                return e;
            //如果头节点hash<0
            if (eh < 0) {
                //头节点是fwd
                if (e instanceof ForwardingNode) {
                    tab = ((ForwardingNode<K,V>)e).nextTable;
                    continue outer;
                }
                else//头节点是树节点，或者-3,-3是什么情况还没遇到过
                //static final int RESERVED  = -3; // hash for transient reservations
                //调用对应类重写的find方法
                    return e.find(h, k);
            }
            //链表从头向后遍历
            if ((e = e.next) == null)
                return null;
        }
    }
}
//treeNode中的find方法就不介绍了，有兴趣自己看下
```

从get方法的实现来看，读操作是没有加锁的，因为Node类中的next变量用`volatile`修饰，可以保证对其他线程的写可见

### 删除元素

获取元素相对来说比较轻松，现在来看一下删除元素的操作是如何进行的。

```java
public V remove(Object key) {
    return replaceNode(key, null, null);
}
//key和value都满足时才删除元素
public boolean remove(Object key, Object value) {
    if (key == null)
        throw new NullPointerException();
    return value != null && replaceNode(key, null, value) != null;
}
//当key和value都正确时，用newValue替换OldValue
public boolean replace(K key, V oldValue, V newValue) {
    if (key == null || oldValue == null || newValue == null)
        throw new NullPointerException();
    return replaceNode(key, newValue, oldValue) != null;
}
//三个方法都调用了内部方法replaceNode()
final V replaceNode(Object key, V value, Object cv) {
    //得到key对应的hash值
    int hash = spread(key.hashCode());
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        //如果table还未初始化，或者tab[i]处头节点为Null,则不存在
        if (tab == null || (n = tab.length) == 0 ||
            (f = tabAt(tab, i = (n - 1) & hash)) == null)
            break;
        //如果头节点是fwd,Hash值为-1，帮助resize    
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        //头节点存在，且是正常Node节点    
        else {
            V oldVal = null;
            boolean validated = false;
            //对头节点加锁
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    //tab[i]后面为链表
                    if (fh >= 0) {
                        validated = true;
                        for (Node<K,V> e = f, pred = null;;) {
                            K ek;
                            //当前节点e的key与remove的key相等
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                (ek != null && key.equals(ek)))) {
                                //ev记录key的旧value    
                                V ev = e.val;
                                //如果cv为空
                                //或者cv不为空，且与旧value相等
                                if (cv == null || cv == ev ||
                                    (ev != null && cv.equals(ev))) {
                                    oldVal = ev;
                                    //如果传入的value不会空，替换
                                    if (value != null)
                                        e.val = value;
                                    //删除操作的话，修改链表指针    
                                    else if (pred != null)
                                        pred.next = e.next;
                                    //如果删除的是头节点，更新tab[i]头节点    
                                    else
                                        setTabAt(tab, i, e.next);
                                }
                                break;
                            }
                            pred = e;
                            if ((e = e.next) == null)
                                break;
                        }
                    }
                    //如果头节点是树的根节点，不分析了
                    else if (f instanceof TreeBin) {
                        validated = true;
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> r, p;
                        if ((r = t.root) != null &&
                            (p = r.findTreeNode(hash, key, null)) != null) {
                            V pv = p.val;
                            if (cv == null || cv == pv ||
                                (pv != null && cv.equals(pv))) {
                                oldVal = pv;
                                if (value != null)
                                    p.val = value;
                                else if (t.removeTreeNode(p))
                                    setTabAt(tab, i, untreeify(t.first));
                            }
                        }
                    }
                }
            }
            if (validated) {
                if (oldVal != null) {
                    if (value == null)
                    //删除后，baseCount-1
                        addCount(-1L, -1);
                    return oldVal;
                }
                break;
            }
        }
    }
    return null;
}
```

### 总结

那么到这里呢，这篇文章的分析就结束了，篇幅还是比较大的，尽管如此，`ConcurrentHashMap`中还有很多的方法和类还没有分析，等以后能力提升了还需要回头再看一看。

`ConcurrentHashMap`通过大量的CAS操作和synchronized来支持同步和提高并发度，虽然分析完了代码，但是还没理解其并发过程的巧妙设计，比如何时需要加锁，何时不需要加锁，何时可以用CAS来减少锁的开销都值得去推敲。也希望大家看完了可以多看看这方面的资料和分析，加深理解。

### 往期回顾

[Java集合源码系列-HashMap](https://cfk1996.github.io/2019/01/12/Java%E9%9B%86%E5%90%88%E6%BA%90%E7%A0%81%E7%B3%BB%E5%88%97-HashMap/)

[Java集合源码系列-ArrayList](https://cfk1996.github.io/2019/01/22/Java%E9%9B%86%E5%90%88%E6%BA%90%E7%A0%81%E7%B3%BB%E5%88%97-ArrayList/)

[Java动态代理原理剖析(一)](https://cfk1996.github.io/2019/02/28/Java%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86%E5%8E%9F%E7%90%86%E5%89%96%E6%9E%90/)

### 欢迎关注我的公众号

这篇文章花费了我好几天的时间，如果觉得对你有一些帮助的话，可以点个赞和分享一下

欢迎关注我的公众号，会经常分享一些技术文章和生活随笔～

![技术旅途](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/wechat.jpg)
---
title: 跳表-ConcurrentSkipListMap走马观花
date: 2019-09-08 15:58:57
tags:
    - Java
    - Java源码
---


跳表（跳跃表）是一种数据结构，改进自链表，用于存储有序的数据，跳跃表通过空间换时间的方法来提高数据检索的速度。早些在学校的数据结构课程中并没有接触过跳表。第一次接触是在了解Redis的有序集合的底层实现的时候，毕竟Redis的五种数据结构是面试中的常见题，即使没有实际使用过，也需要提前去了解一下。

但是最近在工作中不一样了，由于是从事游戏开发，经常会有排行榜的需求需要实现，而这大多数通过跳表来实现，所以有必要来深入了解一下跳表并简略的阅读一下`ConcurrentSkipListMap`的源码。前面也提到了Redis中的有序集合底层就是跳表，所以其也是开发排行榜功能时可以考虑使用的。有时候呢，我们需要排序的东西不需要持久化或者为了更高的效率，是直接在内存中进行排序，毕竟访问redis是有网络io开销的，那么使用到的数据结构就是jdk中的`ConcurrentSkipListMap`（支持同步）或者`TreeMap`（不支持同步，性能更好）.

不过这次源码重点在于跳跃表，不在于并发的控制。

<!--more-->

### 跳跃表

通过上面的介绍，我们了解到跳跃表的应用场景大概是这样的:
1. 有序
1. 频繁插入删除
1. 频繁的查找

对于有序来说，数组和普通链表都可以通过排序算法来实现，在排序复杂度上不相上下。链表在插入和删除上性能较好，数组在查找上性能较好，所以都不能满足我们的要求。跳跃表则是在插入删除和查找的性能上做了折中，复杂度为log(n)。

跳表结构如下所示：

![跳跃表](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2019/skiplist.png)

为了更好的支持插入和删除，所以采用链表的形式，可以看到图片中最下面一行是一个有序的链表。但如果只是一个单一的链表，查找时复杂度为O(n)，性能太差。如何优化呢？

在有序数组中，我们查找时用的是二分查找，一次查找可以排除一半元素的遍历。在数组中之所以可以用二分查找，是因为我们能够快速的以O(1)的复杂度定位到中间的位置，但是链表只能是O(N)。所以跳跃表采取空间换时间的方式，既然你找不到中间点，或者三分之一点等中间位置，那么我可以通过多增加一个节点来指向中间位置，这样你也能够快速的定位到中间的位置，然后一定程度的减少你遍历元素的个数，提高效率。图中有多层，相邻的两层，采用的都是这样的思想。

这个图一目了然，很容易就可以让大家了解跳表的思想。至于我们应该添加多少层额外的链表，给什么位置的节点添加索引才能更好的优化检索和插入的效率，就是我希望通过阅读源码找寻的.


### 节点

通过上图，可以发现有两种节点类型，第一种是最下层的，与普通链表相似，第二种是除了最后一层以外的其他索引层节点，有两个指针。
但是在jdk源码中还存在一种节点，是索引层的头节点，还维护了其层数信息，下面先给出源码注释中的跳表样例。

```
* Head nodes          Index nodes
* +-+    right        +-+                      +-+
* |2|---------------->| |--------------------->| |->null
* +-+                 +-+                      +-+
*  | down              |                        |
*  v                   v                        v
* +-+            +-+  +-+       +-+            +-+       +-+
* |1|----------->| |->| |------>| |----------->| |------>| |->null
* +-+            +-+  +-+       +-+            +-+       +-+
*  v              |    |         |              |         |
* Nodes  next     v    v         v              v         v
* +-+  +-+  +-+  +-+  +-+  +-+  +-+  +-+  +-+  +-+  +-+  +-+
* | |->|A|->|B|->|C|->|D|->|E|->|F|->|G|->|H|->|I|->|J|->|K|->null
* +-+  +-+  +-+  +-+  +-+  +-+  +-+  +-+  +-+  +-+  +-+  +-+
*
```
接着我们来分别看下它们在源码中是如何定义的。

```java
// 记住了，只关注跳表，不去关注并发！！！水平有限，不误导大家
// Node是最底层的链表的节点，包括键值对和指向下一个节点的指针
static final class Node<K,V> {
    final K key;
    volatile Object value;
    volatile Node<K,V> next;
// 至于为什么需要两个构造函数，后面源码会有解释
    Node(K key, Object value, Node<K,V> next) {
        this.key = key;
        this.value = value;
        this.next = next;
    }
    Node(Node<K,V> next) {
        this.key = null;
        this.value = this;
        this.next = next;
    }
    // ...配套method
}

// 索引节点结构
// 存储了两个指针，分别指向右边和下边的节点
// 索引节点的value为链表节点
static class Index<K,V> {
    final Node<K,V> node;
    final Index<K,V> down;
    volatile Index<K,V> right;

    Index(Node<K,V> node, Index<K,V> down, Index<K,V> right) {
        this.node = node;
        this.down = down;
        this.right = right;
    }
    // ...配套method
}

// 索引层的头节点结构
// 在索引节点的基础上添加了表示层数的level变量
static final class HeadIndex<K,V> extends Index<K,V> {
    final int level;
    HeadIndex(Node<K,V> node, Index<K,V> down, Index<K,V> right, int level) {
        super(node, down, right);
        this.level = level;
    }
}
```

### 源码阅读小技巧

JDK中集合包下的代码，一个集合差不多有几千行，想要从上往下逐行看完是不现实的，需要能快速的定位到我们想看的方法。如果是`HashMap`这种常用的数据结构，我们经常使用，对它有哪些方法非常的了解，就可以通过搜索方法名字来定位代码的位置。但是像今天这种`ConcurrentSkipListMap`比较不常用的类，在不了解它大多数方法的时候，可以通过idea提供的功能帮我们快速定位。打开下图中的`show members`就可以让类现实它所有的方法和内部类，就可以靠猜名字快速定位到源码的位置。

![idea技巧](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2019/screen1.png)

### 构造函数

了解完内部链表节点的实现情况，现在按照老规矩，从构造函数开始阅读源码。

```java
// Compartor接口用来指定key的排序规则
public ConcurrentSkipListMap() {
    this.comparator = null;
    initialize();
}
public ConcurrentSkipListMap(Comparator<? super K> comparator) {
    this.comparator = comparator;
    initialize();
}
// 还有两个传入map和sortedMap的构造函数
private void initialize() {
    keySet = null;// 内部类
    entrySet = null;// 内部类
    values = null;// 内部类
    descendingMap = null;// 内部类
// private static final Object BASE_HEADER = new Object();
// 从注释给出的图来看，这个head应该是一直处于第一层的头节点
    head = new HeadIndex<K,V>(new Node<K,V>(null, BASE_HEADER, null),
                              null, null, 1);
}
```

构造函数好像没啥，那么接着往下看吧

### put()

```java
// 与一般的map一样，通过put插入键值对，，key，value不能为空
public V put(K key, V value) {
    if (value == null)
        throw new NullPointerException();
    return doPut(key, value, false);
}

// doPut
private V doPut(K key, V value, boolean onlyIfAbsent) {
    Node<K,V> z;   // 要被添加的Node
    if (key == null)
        throw new NullPointerException();
    // key的比较方法
    Comparator<? super K> cmp = comparator;
    outer: for (;;) { // 因为cas操作可能失败，套了个无限循环
    // findPrecessor返回小于key但最接近key的节点，不存在则为链表的头节点，下面也会介绍
        for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
            if (n != null) {
                Object v; int c;
                Node<K,V> f = n.next;
                // b--->n--->f
                if (n != b.next)
                    break;
                if ((v = n.value) == null) {   // n is deleted
                // 简单来说就是b.next = f，但是考虑了cas
                    n.helpDelete(b, f);
                    break;
                }
                if (b.value == null || v == n) // b is deleted
                    break;
                // 如果key大于n.key
                if ((c = cpr(cmp, key, n.key)) > 0) {
                // 说明有并发插入，继续向后遍历一个节点
                    b = n;
                    n = f;
                    continue;
                }
                // 如果key相等
                if (c == 0) {
                    // 没有指定putIfAbsent的话，通过cas替换value并返回新的value
                    // 指定了putIfabsent则返回原有value值
                    if (onlyIfAbsent || n.casValue(v, value)) {
                        @SuppressWarnings("unchecked") V vv = (V)v;
                        return vv;
                    }
                    break; //cas失败，重试
                }
                // else c < 0;则位置正确，插入就行
            }
            // 创建新的Node节点
            z = new Node<K,V>(key, value, n);
            // 更新链表b->n->f ==> b->z->n->f
            if (!b.casNext(n, z))
                break;     
            break outer;
        }
    }

    int rnd = ThreadLocalRandom.nextSecondarySeed();
    // 0x80000001转换为二进制1000...0001
    // 看起来似乎添加层数是有一定随机性的
    // rnd为0xxx...xxx0时可以进入
    if ((rnd & 0x80000001) == 0) { 
        int level = 1, max;
        // 有多少个1，level递增多少次
        while (((rnd >>>= 1) & 1) != 0)
            ++level;
        Index<K,V> idx = null;
        HeadIndex<K,V> h = head;
        if (level <= (max = h.level)) {
        // 如果level比现在的层数小，则在新增加的节点z上
        // 建立level个索引节点，忘记了可以回上面看看索引节点和其他节点区别
            for (int i = 1; i <= level; ++i)
                idx = new Index<K,V>(z, idx, null);
        }
        else {
        // 如果level大于层数，则level设为层数+1
            level = max + 1; 
            // 构造索引节点数组
            @SuppressWarnings("unchecked")Index<K,V>[] idxs =
                (Index<K,V>[])new Index<?,?>[level+1];
            //为新建的节点z创造level个索引节点
            //下标从1开始
            for (int i = 1; i <= level; ++i)
                idxs[i] = idx = new Index<K,V>(z, idx, null);
            for (;;) {
                h = head;
                int oldLevel = h.level;
            //用于判断并发修改，不考虑
            //正确情况下该分支的level>当前层数
                if (level <= oldLevel) // lost race to add level
                    break;
                HeadIndex<K,V> newh = h;
                Node<K,V> oldbase = h.node;
                for (int j = oldLevel+1; j <= level; ++j)
                //为每层生成一个headIndex
                    newh = new HeadIndex<K,V>(oldbase, newh, idxs[j], j);
                //更新最上层的headIndex指针
                if (casHead(h, newh)) {
                    h = newh;
                    idx = idxs[level = oldLevel];
                    break;
                }
            }
        }
        
        splice: for (int insertionLevel = level;;) {
            int j = h.level;
            // h为最新的头节点
            for (Index<K,V> q = h, r = q.right, t = idx;;) {
                if (q == null || t == null)
                    break splice;
                // r为前面的idxs[x]
                if (r != null) {
                    Node<K,V> n = r.node;
        // compare before deletion check avoids needing recheck
                    int c = cpr(cmp, key, n.key);
                    if (n.value == null) {
                        if (!q.unlink(r))
                            break;
                        r = q.right;
                        continue;
                    }
                // 没并发的情况下，idxs数组里都是新增的key
                // c应该=0
                    if (c > 0) {
                        q = r;
                        r = r.right;
                        continue;
                    }
                }
            // j开始时为新跳表层数
            //insertionLevel为旧跳表，经过后面的几个j--才会进入该分支
                if (j == insertionLevel) {
                    //将t插入q,r之间
                    if (!q.link(r, t))
                        break; // restart
                    if (t.node.value == null) {
                        findNode(key);
                        break splice;
                    }
                    if (--insertionLevel == 0)
                        break splice;
                }

                if (--j >= insertionLevel && j < level)
                    t = t.down;
            //向下一层
                q = q.down;
                r = q.right;
            }
        }
    }
    return null;
}

// findPredecessor
// 回想一下前面那个跳表的结构，该方法就是根据key，先从head往右找，然后往下找
//然后再往右再往下，知道找到比key小的节点
private Node<K,V> findPredecessor(Object key, Comparator<? super K> cmp) {
    if (key == null)
        throw new NullPointerException();
    for (;;) {
        for (Index<K,V> q = head, r = q.right, d;;) {
        // 从head头节点点向后遍历
            if (r != null) {
                Node<K,V> n = r.node;
                K k = n.key;
                //说明被删除了，更新q.next
                if (n.value == null) {
                    if (!q.unlink(r))
                        break;           // restart
                    r = q.right;         // reread r
                    continue;
                }
                if (cpr(cmp, key, k) > 0) {
                    q = r;
                    r = r.right;
                    continue;
                }
            }
            // 失败了就从下一层开始
            if ((d = q.down) == null)
                return q.node;
            q = d;
            r = d.right;
        }
    }
}
```

put函数介绍完了，最后一段更新跳表的操作还是有些乱和难以理解...

### get()

```java
public V get(Object key) {
    return doGet(key);
}

private V doGet(Object key) {
    if (key == null)
        throw new NullPointerException();
    Comparator<? super K> cmp = comparator;
    outer: for (;;) {
        //findPredecessor找到离key最近的，小于key的node
        // 没有并发的情况下，要么是n，要么没有要找的key
        for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
            Object v; int c;
            if (n == null)
                break outer;
            Node<K,V> f = n.next;
            if (n != b.next)                // inconsistent read
                break;
            if ((v = n.value) == null) {    // n is deleted
                n.helpDelete(b, f);
                break;
            }
            if (b.value == null || v == n)  // b is deleted
                break;
            //key与n.key相同
            if ((c = cpr(cmp, key, n.key)) == 0) {
                @SuppressWarnings("unchecked") V vv = (V)v;
                return vv;
            }
            if (c < 0)
                break outer;
            //并发导致key>n.key，说明findPredecessor找的位置不对
            //往后位移一个节点，重新开始
            b = n;
            n = f;
        }
    }
    return null;
}
```

get方法还是比较简单的，`findPrecessor`方法找到的就是小于key但是最接近key的节点，所以key如果存在，必然是`findPrecessor`找到的节点的下一个节点，而代码中为了考虑并发带来的修改，还要做很多其他的判断。

### remove()

接下来看下删除的代码。

```java
public V remove(Object key) {
    return doRemove(key, null);
}

final V doRemove(Object key, Object value) {
    if (key == null)
        throw new NullPointerException();
    Comparator<? super K> cmp = comparator;
    outer: for (;;) {
    // findPrecessor找到比key小但是最近的node
    //不考虑并发的话，如果存在key那就是n=b.next
        for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
            Object v; int c;
            if (n == null)
                break outer;
            Node<K,V> f = n.next;
            //考虑一些并发修改的问题
            if (n != b.next)                    // inconsistent read
                break;
            if ((v = n.value) == null) {        // n is deleted
                n.helpDelete(b, f);
                break;
            }
            if (b.value == null || v == n)      // b is deleted
                break;
            // 如果key<n，说明key不存在，退出最外层循环
            if ((c = cpr(cmp, key, n.key)) < 0)
                break outer;
            //c>0说明findPre方法找到的节点过期了，重新找
            if (c > 0) {
                b = n;
                n = f;
                continue;
            }
            // c==0
            if (value != null && !value.equals(v))
                break outer;
            //通过cas将节点n的value设为null，就是key对应的节点
            //findPrecessor方法会在读到value为null的值时进行删除
            if (!n.casValue(v, null))
                break;
            //n添加删除标志并且更新b.next指针
            //appendMarker创建一个key为null,value为自己的节点
            if (!n.appendMarker(f) || !b.casNext(n, f))
                findNode(key);                  // retry via findNode
            else {
            //删除成功并且更新b.next后进入该分支
            //findPredecessor会在value==null时，更新next指针，实现删除
                findPredecessor(key, cmp);      // clean index
                if (head.right == null)
                    tryReduceLevel();
            }
            @SuppressWarnings("unchecked") V vv = (V)v;
            return vv;
        }
    }
    return null;
}
```

### findPredecessor()

到这里插入、读取和删除都介绍完了，这三个方法都调用了`findPredecessor`，可以说这个方法是跳表的核心了。所以这里拿出来再看一下该方法。

```java
private Node<K,V> findPredecessor(Object key, Comparator<? super K> cmp) {
    if (key == null)
        throw new NullPointerException(); 
    for (;;) {
        //q为第一层的head节点
        //r=q.right说明先向右遍历
        for (Index<K,V> q = head, r = q.right, d;;) {
            //向右遍历的时候，右边的节点不为空
            if (r != null) {
                Node<K,V> n = r.node;
                K k = n.key;
                //在remove方法中，n.casValue(v, null)将节点的值置空
                //置空后，在这里进行索引节点的删除
                if (n.value == null) {
            //unlink操作为将原本的q->r->r.next
            //转换为q->r.next
                    if (!q.unlink(r))
                        break;           // restart
                    r = q.right;         // reread r
                    continue;
                }
                //如果r的节点没被删除，就执行到这里
                //如果key大于当前的k，则向后移一个节点
                //head索引节点也向后移一个节点
                if (cpr(cmp, key, k) > 0) {
                    q = r;
                    r = r.right;
                    continue;
                }
            }
            //如果索引节点的右边没有节点了，则向下移动
            //向下移动后如果还是索引层，则继续向后
            //如果是节点，直接返回了
            if ((d = q.down) == null)
                return q.node;
            q = d;
            r = d.right;
        }
    }
}
```


### 总结

本文介绍了跳表的特点和使用场景，并对`ConcurrentSkipListMap`的源码进行了简略的分析，但由于并发的存在，使得在理解跳表上增加了一些不必要的难度，同时其生成索引层的代码也比较晦涩难懂，可以通过维基百科等更简单的途径来了解其原理和源码。
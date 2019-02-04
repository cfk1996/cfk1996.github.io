---
title: Java集合源码系列-LinkedList
date: 2019-02-03 15:22:04
tags:
    - Java
    - Java源码
---

### 前言

这是Java集合源码系列的第三篇了，准备来介绍一下`LinkedList`这个类,`LinkedList`常与`ArrayList`放在一起比较，都是`List`接口的实现类，但是底层的数据结构不同，导致它们在一些操作上各有优缺点，需要根据应用场景灵活选用`List`。`ArrayList`的细节上篇文章已经介绍过了，不了解的可以回过去看一下。`LinkedList`的底层实现是一个双向链表，同时也实现了`Deque`接口，因此也可以作为双向队列的一个实现，这是`LinkedList`相比`ArrayList`的另外一个重大特性。话不多说，直接进入源码分析阶段吧,按照惯例，依然是从使用集合的整个生命周期来作为行文路线，即创建-->插入-->获取-->删除。

<!--more-->
### 创建

#### 继承关系

![LinkedList](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2019/LinkedList.png)

#### 底层数据结构--双向链表

前面介绍到`LinkedList`的底层数据结构是双向链表，那么先来看一下其实现。

```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

非常的简单，与普通的双向链表完全一致，一个存储数据的**item**,以及两个分别指向前一个节点和后一个节点的指针。

#### 构造函数
```java
transient int size = 0;　// number of elements in list
// 类中有两个指针变量，一个指向双向链表的头，一个指向尾
transient Node<E> first;
transient Node<E> last;
// 创建一个空的list
public LinkedList() {
}
// 从一个已有集合创建
public LinkedList(Collection<? extends E> c) {
    this();
// addAll就是将collection中所有元素添加到list中，具体在添加元素部分介绍
    addAll(c);
}
```

`LinkedList`的构造函数非常简单，而且成员变量的数目最少，只有3个，那么现在来看下如何添加元素吧。

### 添加元素

首先介绍下上面构造函数中提到的的**addAll**方法

```java
public boolean addAll(Collection<? extends E> c) {
    return addAll(size, c); //　size初始值为0
}
//　从指定Index处插入给定集合中的所有元素
public boolean addAll(int index, Collection<? extends E> c) {
//　check index>=0 && index <= size
    checkPositionIndex(index);
    Object[] a = c.toArray();
    int numNew = a.length;
    if (numNew == 0)
        return false;
    Node<E> pred, succ;
    if (index == size) {
    //　index为最后一个元素的后面一个时
        succ = null;
        pred = last;//pred指向List中最后一个节点
    } else {
    //　index是中间元素时
    //　node方法返回下标为index的节点,将在获取元素部分详细介绍
        succ = node(index);
    // pred指向succ的前一个节点
        pred = succ.prev;
    }
    // 新插入的元素在pred指针之后
    // 遍历数组a，将元素逐个插入，并指定新的first和last指针
    for (Object o : a) {
        E e = (E) o;
        Node<E> newNode = new Node<>(pred, e, null);
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        pred = newNode;
    }
    if (succ == null) {
        last = pred;
    } else {
        pred.next = succ;
        succ.prev = pred;
    }
    //　修改size的数值
　  size += numNew;
　　//　对于add, delete这样结构性的修改方式，modCount记录修改次数，用于迭代时检测list是否被修改，被修改则抛出异常，但是不保证一定抛出异常。需要用其他方式检测并发操作的正确性
    modCount++;
    return true;
}
private void checkPositionIndex(int index) {
    if (!isPositionIndex(index))
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
//　生成错误信息
private String outOfBoundsMsg(int index) {
    return "Index: "+index+", Size: "+size;
}
private boolean isPositionIndex(int index) {
    return index >= 0 && index <= size;
}
```

上面的是addAll方法，应该是不常用的(我猜的)。那么现在来看下`LinkedList`作为**list**实现的常用添加方法**add**

```java
public boolean add(E e) {
    linkLast(e);
    return true;
}
//　在双向链表末尾插入新元素
void linkLast(E e) {
// last指针指向双向链表最后一个元素，l指向last
    final Node<E> l = last;
// 新创建一个节点，其prev指针指向链表末尾
    final Node<E> newNode = new Node<>(l, e, null);
// 更新last指针
    last = newNode;
    if (l == null)
//　如果当前没有元素，插入的元素即为第一个元素，将first指针指向该node
        first = newNode;
    else
// 如果双向链表已经存在，将添加之前的链表的最后一个几点的next指针更新
        l.next = newNode;
// 修改size和modCount
    size++;
    modCount++;
}
// *******************************************************
//　指定下标处插入元素的add方法
public void add(int index, E element) {
// check index>=0 && index<=size
    checkPositionIndex(index);
    if (index == size)
// 等价于末尾插入
        linkLast(element);
    else
// 在链表前端节点插入，node方法返回index处的节点，后面详细介绍
        linkBefore(element, node(index));
}
void linkBefore(E e, Node<E> succ) {
//　succ为index所在位置的节点，原链表A->succ->B
// 插入后链表A->newNode->succ->B
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
//　因为是双向链表，succ.prev指针也需要更新
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
// *******************************************************
// 接下来看看LinkedList作为deque的一些方法
public void addLast(E e) {
    linkLast(e);
}
// 入栈操作
public void push(E e) {
    addFirst(e);
}
public void addFirst(E e) {
    linkFirst(e);
}
//　将元素添加到链表的第一个节点
private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}
```

插入操作总的来说是先找到插入的位置，生成一个新的双向链表的Node节点，修改前后节点相应的指针。时间开销只需要一个遍历的开销，相比于`ArrayList`,`ArrayList`需要将index之后的元素进行位移，花费的开销更大。

### 获取元素

```java
//　先介绍下前面出现过几次的node()方法
// node方法返回指定Index处的节点，根据index与first及last的距离，选择较近的一端进行遍历，总的复杂度为O(n),而ArrayList根据index获取元素的时间复杂度为O(1)
Node<E> node(int index) {
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
//根据指定下标获取元素
public E get(int index) {
//check index>=0 && index<size
    checkElementIndex(index);
    return node(index).item;
}
private void checkElementIndex(int index) {
    if (!isElementIndex(index))
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
private boolean isElementIndex(int index) {
    return index >= 0 && index < size;
}
//　作为队列，有getFirst和getLasr两个方法，直接返回内部成员变量first和last指针的值，非空才返回，为空抛异常
public E getFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return f.item;
}
public E getLast() {
    final Node<E> l = last;
    if (l == null)
        throw new NoSuchElementException();
    return l.item;
}
//　判断是否包含某个元素
public boolean contains(Object o) {
    return indexOf(o) != -1;
}
//　返回第一个相等对象的index
public int indexOf(Object o) {
    int index = 0;
//　从first节点开始遍历，检验是否存在对象o
    if (o == null) {
//　支持查找null对象
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null)
                return index;
            index++;
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item))
                return index;
            index++;
        }
    }
// 查找不到return -1
    return -1;
}
//还有一个lastIndexOf,从后向前查找，实现原理相同
```

### 删除元素

```java
//删除指定下标处的元素
public E remove(int index) {
//check index>=0 && index<size
    checkElementIndex(index);
    return unlink(node(index));
}
//删除指定对象，与indexOf的实现雷同
public boolean remove(Object o) {
// 从first向后遍历，支持null对象，找到该对象后调用unlink方法删除，删除成功返回true，否则false
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
E unlink(Node<E> x) {
    final E element = x.item;
// 获取即将删除节点的前一个节点和后一个节点
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;
    if (prev == null) {
// 被删除节点为first指向的节点，更新first指针为next
        first = next;
    } else {
// 被删除节点不是first指向的节点，将前一个节点的next指针更新，并把被删除节点的prev节点置空
        prev.next = next;
        x.prev = null;
    }
    if (next == null) {
// 被删除节点为last指向的节点，更新last指针为prev
        last = prev;
    } else {
// 被删除的节点不是last指向的节点，更新被删除节点的后一个节点的prev指针
        next.prev = prev;
        x.next = null;
    }
//　将x的各个成员变量置为null，方便GC,同时对size和modCount进行修改后返回被删除元素的值
    x.item = null;
    size--;
    modCount++;
    return element;
}
//修改指定下标处的值
public E set(int index, E element) {
// check index>=0 && index<size
    checkElementIndex(index);
//通过node方法获取index下标对应的节点，将其item变量更新
    Node<E> x = node(index);
    E oldVal = x.item;
    x.item = element;
    return oldVal;
}
//删除所有元素
public void clear() {
//从first向后遍历，将当前节点的成员变量都置空
    for (Node<E> x = first; x != null; ) {
        Node<E> next = x.next;
        x.item = null;
        x.next = null;
        x.prev = null;
        x = next;
    }
//将first,last,size恢复为空值
    first = last = null;
    size = 0;
    modCount++;
}
// *****************************************************
// 作为deque的一些删除方法
public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}
private E unlinkFirst(Node<E> f) {
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC
//更新first指针，并判断是否还存在元素
    first = next;
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}
//removeLast与removeFirst实现相似，不再重复
public E removeLast() {
    final Node<E> l = last;
    if (l == null)
        throw new NoSuchElementException();
    return unlinkLast(l);
}
private E unlinkLast(Node<E> l) {
    final E element = l.item;
    final Node<E> prev = l.prev;
    l.item = null;
    l.prev = null; // help GC
    last = prev;
    if (prev == null)
        first = null;
    else
        prev.next = null;
    size--;
    modCount++;
    return element;
}
```

### deque方法介绍

上面的分析差不多已经介绍完大部分代码了，其实可以发现，有很多代码的逻辑都是相似的。`LinkedList`作为队列的实现类，还有很多针对队列的实现方法，但其实只是原有代码的再封装，我把之前没提到的一些方法放在这一部分一起介绍了。

```java
//返回队列第一个元素的值
public E peek() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}
//返回队列第一个元素的值，与peek的区别在于，element方法在第一个元素不存在时会抛出异常
public E element() {
    return getFirst();
}
//返回队列第一个元素，并删除
public E poll() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}
//返回队列第一个元素，并删除，为空抛异常
public E remove() {
    return removeFirst();
}
//末尾添加
public boolean offer(E e) {
    return add(e);
}
//头节点添加
public boolean offerFirst(E e) {
    addFirst(e);
    return true;
}
//末尾添加
public boolean offerLast(E e) {
    addLast(e);
    return true;
}
//弹出list的第一个元素
public E pop() {
    return removeFirst();
}
```

### 总结

`LinkedList`的大部分代码都介绍完了，可以看出来比较简单。`LinkedList`底层实现为双向链表，除了可以作为**List**外，还可以作为队列和栈的实现类。因为是双向链表，所以对频繁的插入和删除操作较为友好，但是对频繁的根据下标获取的场景不够友好，需要权衡一下。才疏学浅，如有错误，欢迎指出

### 往期回顾

* [Java集合源码系列-HashMap](https://cfk1996.github.io/2019/01/12/Java%E9%9B%86%E5%90%88%E6%BA%90%E7%A0%81%E7%B3%BB%E5%88%97-HashMap/)

* [Java集合源码系列-ArrayList](https://cfk1996.github.io/2019/01/22/Java%E9%9B%86%E5%90%88%E6%BA%90%E7%A0%81%E7%B3%BB%E5%88%97-ArrayList/)

### 欢迎关注我的公众号

欢迎关注我的公众号，会经常分享一些技术文章和生活随笔～

![技术旅途](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/wechat.jpg)

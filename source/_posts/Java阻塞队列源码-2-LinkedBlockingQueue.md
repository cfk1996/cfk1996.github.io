---
title: Java阻塞队列源码(2)-LinkedBlockingQueue
date: 2019-07-03 17:11:53
tags:
    - Java
    - Java源码
---

在系列的第一篇中介绍了`ArrayBlockingQueue`的源码实现，用数组实现了阻塞队列，作为系列的第二篇，将分析`LinkedBlockingQueue`的源码。基于链表的队列与基于数组的队列相比，基于链表的数据结构将提供更大的容量，但是随机读写的性能会低于按下标访问的速度，这些由于底层存储元素的数据结构不同导致的性能差异在两种`List`的实现中也体现了出来。

### 构造函数与成员变量

首先看下`LinkedBlockingQueue`类的继承关系,与`ArrayBlockingQueue`的继承关系一致,`AbstractQueue`是一个抽象类，用模板方法设计模式封装了几个简单的方法，具体的方法实现在实现类中。`BlockingQueue`接口则定义了阻塞队列的通用方法，在系列第一篇文章中介绍过，不再分析。

<!--more-->

```java
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {...}
```

#### 构造函数

接下来看一下其构造函数，构造函数共有三个，与`ArrayBlockingQueue`必须指定一个容量不同的是，`LinkedBlockingQueue`可以不需要指定队列大小，默认为`Integer.MAX_VALUE`，相当于一个无界队列。而之前分析过的线程池源码中提到了`Executors`的多数工厂方法，都使用了无界队列作为存储任务的数据结构，不加注意会导致溢出。

```java
// 无参构造函数，则队列大小默认设为Integer.MAX_VALUE
public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE);
}

// capacity用于指定阻塞队列的大小，该构造方法是三个构造方法中的最根本 
public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    //用于是基于链表的队列，初始化头结点和尾结点
    //Node类的介绍在后面
    last = head = new Node<E>(null);
}

//该构造函数初始化无界的阻塞队列，然后将集合c中元素依次插入
public LinkedBlockingQueue(Collection<? extends E> c) {
    this(Integer.MAX_VALUE);
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        int n = 0;
        for (E e : c) {
            if (e == null)
                throw new NullPointerException();
            if (n == capacity)
                throw new IllegalStateException("Queue full");
            enqueue(new Node<E>(e));
            ++n;
        }
        count.set(n);
    } finally {
        putLock.unlock();
    }
}
```

#### Node结点

`LinkedBlockingQueue`底层数据结构为链表，由一个`Node`内部类实现，先看下`Node`类的源码。

```java
//成员变量可直接访问，不通过get,set方法访问
static class Node<E> {
    E item;//保存元素
    Node<E> next;//单向链表，指向下一个结点的指针
    //构造函数非常简单，用item指向放入队列的元素
    Node(E x) { item = x; }
}
```

#### 成员变量

之前的源码分析文章中没有将成员变量单独拿出来分析，只是在代码出现的位置提及一下，觉得不够清晰，这篇文章尝试将成员变量单独提炼出来。`LinkedBlockingQueue`共有如下9个成员变量(包括一个序列化id)：

![Fields](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2019/LinkedBlockingQueueFields.PNG)

```java
// 队列容量，为空时默认为Integer.MAX_VALUE
private final int capacity;
//队列中当前元素个数，在ArrayBlockingQueue中count变量不是原子变量，修改操作在锁中完成，LinkedBlocking如何修改count变量在后面介绍
private final AtomicInteger count = new AtomicInteger();
//链表头结点和尾结点
//需要注意的是head结点为一个空节点
transient Node<E> head;
private transient Node<E> last;
//读操作的锁
private final ReentrantLock takeLock = new ReentrantLock();
//非空标志，用于唤醒阻塞的读取方法
private final Condition notEmpty = takeLock.newCondition();
//写操作的锁
private final ReentrantLock putLock = new ReentrantLock();
//非满标志，用于唤醒阻塞的写入方法
private final Condition notFull = putLock.newCondition();
```

### 写方法

#### offer()方法

`offer`方法将元素插入队列的尾部，其有两种重载方式，首先介绍第一种，不指定阻塞时间

```java
public boolean offer(E e) {
    //禁止插入null元素
    if (e == null) throw new NullPointerException();
    final AtomicInteger count = this.count;
    //判断是否超过容量限制，超过了直接返回false
    //因为count是原子变量，不需要在锁里进行读操作
    if (count.get() == capacity)
        return false;
    int c = -1;
    //将需要插入的元素封装进Node结点
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    // 为插入操作加锁
    putLock.lock();
    try {
        //距离上一次读取count有一段不是同步的代码
        //重新判断是否超过容量
        if (count.get() < capacity) {
        //插入node结点，细节后面介绍    
            enqueue(node);
            c = count.getAndIncrement();
            //队列没有满
            if (c + 1 < capacity)
            //唤醒阻塞的notFull.await()
                notFull.signal();
        }
    } finally {
        putLock.unlock();
    }
    //如果插入的是队列中第一个，唤醒读取线程
    if (c == 0)
    //下面分析
        signalNotEmpty();
    return c >= 0;
}

// enqueue方法
// 与ArrayBlockingQueue中的相比，简单的令人发指
//首先将node节点放在链表尾端即last.next=node
//然后更新last节点到末尾，即last=last.next
private void enqueue(Node<E> node) {
    last = last.next = node;
}

// signalNotEmpty方法
private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
}
```

`offer`方法还提供了指定阻塞时间的重载方法,除了多了一个循环等待，其他逻辑相似

```java
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {
//禁止插入null
    if (e == null) throw new NullPointerException();
    //获取阻塞时间
    long nanos = unit.toNanos(timeout);
    int c = -1;
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
        //如果队列达到容量限制但没超时，阻塞等待
        while (count.get() == capacity) {
            if (nanos <= 0)
                return false;
            nanos = notFull.awaitNanos(nanos);
        }
        //插入元素Node(e)
        enqueue(new Node<E>(e));
        c = count.getAndIncrement();
        //判断是否达到容量限制
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return true;
}
```

#### put()方法

除了`offer`方法，`put`方法同样可以向队列中插入元素。但与`offer`不同的是`put`方法在队列满时，会无限阻塞等待，不会返回`false`。

```java
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();

    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
        //容量满时，无限阻塞等待，与offer不同
        while (count.get() == capacity) {
            notFull.await();
        }
        //插入node
        enqueue(node);
        //更新count
        c = count.getAndIncrement();
        //判断是否超过容量限制
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
}
```

### 读方法

#### poll()方法

`poll`方法与`offer`应该是一组相对的方法，同样有两种重载形式

```java
public E poll() {
    final AtomicInteger count = this.count;
    //队列为空，直接返回null，所以不允许插入null
    //null用于标志队列为空
    if (count.get() == 0)
        return null;
    E x = null;
    int c = -1;
    final ReentrantLock takeLock = this.takeLock;
    //读锁
    takeLock.lock();
    try {
        //双重检查
        if (count.get() > 0) {
            //读取元素，dequeue下面介绍
            x = dequeue();
            //更新count
            c = count.getAndDecrement();
            //非空则唤醒notEmpty.await
            if (c > 1)
                notEmpty.signal();
        }
    } finally {
        takeLock.unlock();
    }
    //c是读取前的元素数量
    //因此c如果等于capacity说明队列从满的变成不是满的
    //因此唤醒notFull.await()
    if (c == capacity)
        signalNotFull();
    return x;
}

// dequeue方法分析
// dequeue方法从头结点处移除一个结点
private E dequeue() {
    //head为空节点，在头部增加一个空节点可以方便对链表的操作
    Node<E> h = head;
    //first为真正保存了元素的第一个结点
    Node<E> first = h.next;
    //头结点next指向自己，帮助gc
    h.next = h;
    //将head结点更新到第一个有值的结点
    head = first;
    //获取第一个结点的元素，作为方法返回结果
    E x = first.item;
    //head结点又置空，注意头结点始终为一个空的占位结点
    first.item = null;
    return x;
}
```

现在看下`poll`的第二种形式，指定阻塞时间，但大多数逻辑与前一个`poll`相似

```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    E x = null;
    int c = -1;
    long nanos = unit.toNanos(timeout);
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        //没有元素可取时，阻塞指定的时间
        //超时后返回Null
        while (count.get() == 0) {
            if (nanos <= 0)
                return null;
            nanos = notEmpty.awaitNanos(nanos);
        }
    // ...逻辑与前面几乎一样
    return x;
}
```

#### take()方法

除了`poll`以外，还有`take`方法可以用来读取元素.其与`poll`的区别在于队列为空时的表现不同，`poll`方法在队列为空时会返回`null`，而`take`方法会无限阻塞除非发生异常。

```java
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        //如果队列为空，take方法会一直阻塞
        while (count.get() == 0) {
            notEmpty.await(); 
        }
        //后面逻辑与前面几个get方法一致
        x = dequeue();
        c = count.getAndDecrement();
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}
```

### 其他方法汇总

介绍完上面的构造方法、写入和读取方法后，类中还有很多的其他辅助方法，统一归到这里分析。

#### remove()方法与contains()方法

`remove`方法在队列中删除指定的一个元素，还有一个`contains`方法的逻辑与`remove`类似，都是遍历找到相等的元素，但找到后的处理逻辑略有不同。

```java
public boolean remove(Object o) {
    if (o == null) return false;
    //给读锁和写锁都加上锁
    fullyLock();
    try {
        //从头向后遍历链表
        for (Node<E> trail = head, p = trail.next;
            p != null;
            trail = p, p = p.next) {
            //找到与指定元素相等的元素
            if (o.equals(p.item)) {
                //删除该结点后返回true
                unlink(p, trail);
                return true;
            }
        }
        //没有找到则返回false
        return false;
    } finally {
        //释放读锁和写锁
        fullyUnlock();
    }
}
// fullyLock()与fullyUnlock()
// 全部加锁和全部释放锁
void fullyLock() {
    putLock.lock();
    takeLock.lock();
}
void fullyUnlock() {
    takeLock.unlock();
    putLock.unlock();
}

// unlink()方法用于链表的删除操作
//根据remove中的for循环来看，trail.next=p
//p是需要删除的结点
void unlink(Node<E> p, Node<E> trail) {
// 将需要删除的结点置为null，help gc
    p.item = null;
    // 更新链表的指向关系，跳过p结点
    trail.next = p.next;
    //判断p结点是否是尾结点
    if (last == p)
    //如果是，p结点的前继结点trail则变为尾结点
        last = trail;
    //告知其他线程队列元素没满
    if (count.getAndDecrement() == capacity)
        notFull.signal();
}

// contains方法
public boolean contains(Object o) {
    if (o == null) return false;
    fullyLock();
    try {
    //遍历判断是否找到相等的元素，找到返回true，反之false
        for (Node<E> p = head.next; p != null; p = p.next)
            if (o.equals(p.item))
                return true;
        return false;
    } finally {
        fullyUnlock();
    }
}
```

#### peek()方法

获取队列的第一个元素但不删除

```java
public E peek() {
    //队列为空时直接返回Null
    if (count.get() == 0)
        return null;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    // 读锁
    try {
        //读取第一个元素，判断是否为空
        Node<E> first = head.next;
        if (first == null)
            return null;
        else
            return first.item;
    } finally {
        takeLock.unlock();
    }
}
```

#### clear()方法

原子的删除队列中所有元素

```java
public void clear() {
// 获取读锁、写锁
    fullyLock();
    try {
        //遍历链表，将每个结点的item置null
        for (Node<E> p, h = head; (p = h.next) != null; h = p) {
            h.next = h;
            p.item = null;
        }
        //更新head以及last指针为初始状态
        head = last;
        //通知队列未满
        if (count.getAndSet(0) == capacity)
            notFull.signal();
    } finally {
        //释放读锁和写锁
        fullyUnlock();
    }
}
```

### 总结

到这里大部分方法的源码都已经分析完了，依然很简单。`LinkedBlockingQueue`用链表作为底层的存储结构，支持有界和无界两种阻塞队列形式，靠两把锁--读锁和写锁以及两个条件来进行并发控制。

### 往期回顾

* [Java阻塞队列源码1-ArrayBlockingQueue](https://cfk1996.github.io/2019/07/01/Java%E9%98%BB%E5%A1%9E%E9%98%9F%E5%88%97%E6%BA%90%E7%A0%81%281%29-ArrayBlockingQueue/)

* [Java线程池源码](https://cfk1996.github.io/2019/05/02/Java%E7%BA%BF%E7%A8%8B%E6%B1%A0%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)

## 欢迎关注我的公众号

欢迎关注我的公众号，会经常分享一些技术文章和生活随笔～

![飞坤吖](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/wechat.jpg)
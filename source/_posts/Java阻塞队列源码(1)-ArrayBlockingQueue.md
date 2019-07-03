---
title: Java阻塞队列源码(1)-ArrayBlockingQueue
date: 2019-07-01 17:41:17
tags:
    - Java
    - Java源码
---

Java并发包下有个`BlockingQueue`接口，并提供了多种阻塞队列的实现方式。阻塞队列通常被用于生产者消费者模型、消息队列、并行任务等并发场景，并通过内部的锁和并发控制实现线程安全。这个系列将分析其中多种实现的源码，了解阻塞队列的实现细节，从而能够根据使用场景的不同选择最适合的阻塞队列实现类。

并发包下关于阻塞队列的接口和实现如下所示：

![接口与实现类](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2019/BlockingQueue.PNG)

<!--more-->

### BlockingQueue接口

从上面的关系图可以看出，所有的实现类或者接口都派生自`BlockingQueue`接口，该接口共定义了11个方法，除此之外，它还继承自接口`Queue`。

```java
public interface BlockingQueue<E> extends Queue<E> {
    // 非阻塞添加指定元素到队列中，失败抛出异常
    boolean add(E e);
    // 非阻塞添加指定元素，失败返回false
    boolean offer(E e);
    // 阻塞添加
    void put(E e) throws InterruptedException;
    // 阻塞读取并删除第一个元素
    E take() throws InterruptedException;
    // 阻塞一段时间内添加
    boolean offer(E e, long timeout, TimeUnit unit) throws InterruptedException;
    // 阻塞一段时间读取并删除第一个元素
    E poll(long timeout, TimeUnit unit) throws InterruptedException;
    // 队列剩余容量
    int remainingCapacity();
    // 删除指定元素(if exist)
    boolean remove(Object o);
    public boolean contains(Object o);
    // 删除当前队列所有元素，并添加到新集合中
    int drainTo(Collection<? super E> c);
    int drainTo(Collection<? super E> c, int maxElements);
}
```

### ArrayBlockingQueue类

系列的第一篇文章将分析`ArrayBlockingQueue`的源码，首先看下其构造函数。

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

// 构造函数共有三个，最终都会调用该两参数的构造函数
    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
    // final Object[] items;
    // 保存队列元素的数组
        this.items = new Object[capacity];
    // 初始化内部锁，根据fair决定是否是公平锁，会影响多个线程读写顺序   
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }

    public ArrayBlockingQueue(int capacity) {
        this(capacity, false);
    }
// 初始化阻塞队列，并将传入的集合添加到队列
    public ArrayBlockingQueue(int capacity, boolean fair,
                              Collection<? extends E> c) {
        this(capacity, fair);
        // ...省略
    }
}
```

#### 生产者消费者

阻塞队列的一个使用场景是生产者消费者模式，那么现在举一个生产者消费者的代码demo来展示如何使用阻塞队列，然后进一步分析其中用到的方法的源码。

```java
class Producer implements Runnable {
    private final BlockingQueue queue;
    Producer(BlockingQueue q) { queue = q; }
    public void run() {
        try {
        // 生产者通过put方法不断向队列添加元素
            while (true) { queue.put(produce()); }
        } catch (InterruptedException ex) { ... handle ...}
   }
   Object produce() { ... }
 }

class Consumer implements Runnable {
    private final BlockingQueue queue;
    Consumer(BlockingQueue q) { queue = q; }
    public void run() {
        try {
        // 消费者通过take方法不断向队列读取元素
            while (true) { consume(queue.take()); }
        } catch (InterruptedException ex) { ... handle ...}
   }
   void consume(Object x) { ... }
}

class Setup {
// demo中有多个生产者和消费者线程，阻塞队列能够保证线程安全    
    void main() {
        BlockingQueue q = new SomeQueueImplementation();
        Producer p = new Producer(q);
        Consumer c1 = new Consumer(q);
        Consumer c2 = new Consumer(q);
        new Thread(p).start();
        new Thread(c1).start();
        new Thread(c2).start();
    }
}
```

#### put方法与take方法

生产者消费者模型中，首先需要生产者添加元素，否则队列中没有元素，消费者无法执行，添加元素的操作通过`put`方法来执行，消费元素则通过`take`方法，下面一一分析。

```java
public void put(E e) throws InterruptedException {
//禁止插入null值，null值用于无可读元素时的读取失败标志    
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
//该获取锁的方法，即使在阻塞获取锁时(即还没拿到锁)也会响应其他线程调用该线程的interrupt方法，抛出InterruptedException异常
    lock.lockInterruptibly();
    try {
// int count;队列中元素个数
// 如果队列满了，通过await方法阻塞等待，即使不往下看也能想到take或其他读取元素的方法中必然有个地方会调用notFull.signal()方法来唤醒当前线程
        while (count == items.length)
            notFull.await();
        //插入元素e    
        enqueue(e);
    } finally {
        lock.unlock();
    }
}
//该方法用于在putIndex处插入元素，只在拥有锁的情况下被调用
private void enqueue(E x) {
    //获取存储元素的数组
    final Object[] items = this.items;
//int putIndex;下一个待添加元素的下标，初始为0
    items[putIndex] = x;
    //防止越界，效果类似于循环数组
    if (++putIndex == items.length)
        putIndex = 0;
    //队列中元素个数加一    
    count++;
    //唤醒notEmpty.await()
    notEmpty.signal();
}
```

`put`方法通过锁来实现线程安全，同时通过`notFull`和`notEmpty`两个方法来进行线程间控制，`take`方法也大致如此，下面看下`take`方法

```java
public E take() throws InterruptedException {
// 获取锁，保证读取的安全性    
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
    //判断队列是否为空，为空阻塞等待
    //在上面分析的enqueue方法的最后会调用notEmpty.signal来唤醒    
        while (count == 0)
            notEmpty.await();
    //如果队列不为空，退出循环，通过dequeue()返回读取的元素
        return dequeue();
    } finally {
        lock.unlock();
    }
}
// dequeue方法取出takeIndex下标处的元素，只在锁内被调用
private E dequeue() {
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
// items index for next take, poll, peek or remove
// int takeIndex;初始化为0
//读取takeIndex处的元素，并删除数组中对其引用    
    E x = (E) items[takeIndex];
    items[takeIndex] = null;
    //防止越界
    if (++takeIndex == items.length)
        takeIndex = 0;
    //队列元素数量减一
    count--;
//itrs同于与当前正活跃的迭代器共享状态，如果存在，通知其一起删除该元素
    if (itrs != null)
        itrs.elementDequeued();
    //唤醒阻塞的put方法
    notFull.signal();
    return x;
}
```

#### offer()与poll()方法

上面介绍的一对读写方法是无限阻塞的，除非有其他线程调用signal方法来唤醒当前线程。阻塞队列中还提供了几个可以指定阻塞时间的读写方法

```java
//offer方法可以指定阻塞时间，除此之外与put不同的是offer方法会返回true或者false
//而put方法正常情况下一直阻塞，二者都会抛出中断异常
public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {
    checkNotNull(e);//禁止插入null元素
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
    //逻辑与Put方法几乎一样，除了增加了阻塞时间    
        while (count == items.length) {
            if (nanos <= 0)
                return false;
            nanos = notFull.awaitNanos(nanos);
        }
        enqueue(e);
        return true;
    } finally {
        lock.unlock();
    }
}
//poll方法的逻辑也非常简单，不过多介绍，与take相似
//不同点在与在阻塞时间到达后，poll方法会返回null值
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0) {
            if (nanos <= 0)
                return null;
            nanos = notEmpty.awaitNanos(nanos);
        }
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```

同时poll方法和offer方法还有一种重载实现，提供了立即返回结果的功能，代码如下

```java
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
    //在阻塞的方式中，会通过while循环和await方法的配合来实现阻塞
    //但在该方法中，会立刻返回结果    
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();
    }
}
//逻辑相似，理解返回结果，不阻塞
public boolean offer(E e) {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        if (count == items.length)
            return false;
        else {
            enqueue(e);
            return true;
        }
    } finally {
        lock.unlock();
    }
}
```

现在已经分析了6个常用方法了，接下来把前面接口中出现的方法依次分析一下，可以看出整个该实现类的源码比较简单

#### add()方法

```java
//add方法直接调用了继承的抽象类中的add方法
public boolean add(E e) {
    return super.add(e);
}
//AbstractQueue中定义的add方法
public boolean add(E e) {
//借用了offer方法的实现，offer方法的实现在具体实现类中，上面介绍过
//add方法与另外两个添加元素方法的不同点在于，添加失败时抛出IllegalStateException
    if (offer(e))
        return true;
    else
        throw new IllegalStateException("Queue full");
}
```

#### remove()方法

`remove`方法用于在队列中删除指定元素，因为是随机的，与队列的FIFO的特性不符，性能较差。

```java
public boolean remove(Object o) {
    if (o == null) return false;
    final Object[] items = this.items;
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //判断队列中是否有元素
        if (count > 0) {
            final int putIndex = this.putIndex;
            int i = takeIndex;
        //遍历判断是否相等，然后调用removeAt删除    
            do {
                if (o.equals(items[i])) {
                    removeAt(i);
                    return true;
                }
                if (++i == items.length)
                    i = 0;
            } while (i != putIndex);
        }
        return false;
    } finally {
        lock.unlock();
    }
}

void removeAt(final int removeIndex) {
    final Object[] items = this.items;
    //如果要删除的元素正好是正常情况下的那个元素，即takeIndex
    //删除后修改takeIndex并将count减一
    if (removeIndex == takeIndex) {
        items[takeIndex] = null;
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
    } else {//要删除的不是takeIndex处的元素
        final int putIndex = this.putIndex;
        //将i+1到putIndex处元素向前移动一个位置，并更新putIndex
        for (int i = removeIndex;;) {
            int next = i + 1;
            if (next == items.length)
                next = 0;
            if (next != putIndex) {
                items[i] = items[next];
                i = next;
            } else {
                items[i] = null;
                this.putIndex = i;
                break;
            }
        }
        //队列元素数量减一，并删除迭代器中的该元素
        count--;
        if (itrs != null)
            itrs.removedAt(removeIndex);
    }
    //唤醒阻塞的notFull.await
    notFull.signal();
}
```

#### 其他方法汇总

除了上面介绍的几个方法，还有一些比较简单的方法，统一放在这一小节介绍，相互之间关联较小，可以看做是多个独立的方法。

```java
//读取队列的第一个元素，但不删除
//加锁后读取数组中takeIndex的下标处
//在多线程中，读取的操作也都需要通过加锁操作进行，否则会读取到不正确的值
public E peek() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return itemAt(takeIndex); // null when queue is empty
    } finally {
        lock.unlock();
    }
}
final E itemAt(int i) {
    return (E) items[i];
}

//查看队列剩余容量
//count变量维护着队列中元素的个数
public int remainingCapacity() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return items.length - count;
    } finally {
        lock.unlock();
    }
}
//还有比如size(),clear()等方法，都比较简单，把代码粘贴进来就能看懂，不浪费篇幅了
```

#### Itr与Itrs介绍

在上面的代码分析中，多次出现了如下所示的代码片段
```java
if (itrs != null)
    itrs.elementDequeued();
```

这一节就介绍下这个对象是什么，起到了什么作用。`Itrs`是`ArrayBlockingQueue`中的一个内部类，`itrs`则为其一个成员变量。初始化时为null，`transient Itrs itrs = null;`

源码中关于`Itrs`的描述截取如下

* Shared data between iterators and their queue, allowing queue modifications to update iterators when elements are removed.

该对象在迭代器即阻塞队列之间共享了数据，在队列删除元素时会更新迭代器。

构造迭代器的方法如下：

```java
public Iterator<E> iterator() {
    //这里是Itr不是Itrs
    return new Itr();
}
```

然后进入Itr的构造函数中看下：

```java
Itr() {
    lastRet = NONE;
    final ReentrantLock lock = ArrayBlockingQueue.this.lock;
    lock.lock();
    try {
        //队列中没有元素
        if (count == 0) {
            cursor = NONE;
            nextIndex = NONE;
            prevTakeIndex = DETACHED;
        } else {
            final int takeIndex = ArrayBlockingQueue.this.takeIndex;
            prevTakeIndex = takeIndex;
            nextItem = itemAt(nextIndex = takeIndex);
            cursor = incCursor(takeIndex);
            if (itrs == null) {
        //在这里会构造一个Itrs对象，并赋值给ArrayBlockingQueue中的itrs对象
        //而Itrs内部类似个链表，用于迭代   
                itrs = new Itrs(this);
            } else {
                itrs.register(this); // in this order
                itrs.doSomeSweeping(false);
            }
            prevCycles = itrs.cycles;
        }
    } finally {
        lock.unlock();
    }
}
```

### 总结

`ArrayBlockingQueue`的源码还是比较简单的，所有的读写操作通过内部的同一把`ReentrantLock`锁来控制，在队列满或者队列空时通过两个`Condition`来进行通信。队列通过一个FIFO的环形数组来实现，维护了`takeIndex`和`putIndex`等变量来决定插入和读取的元素位置。

### 瞎聊

过去几个月因为是毕业季，时间都被我玩掉了，没有好好的看源码，也没有写博客，今天放假没事做终于是补了一篇。写阻塞队列是因为上一篇介绍线程池原理的源码中用到了阻塞队列，所以顺着这个思路准备把阻塞队列的多个实现类的源码都看一遍。

写之前其实没看过`ArrayBlockingQueue`的源码，是一边看一边写出来的。写着写着发现比之前几篇源码分析简单多了，但是开弓没有回头箭，既然写了，内容不多也发出来吧。

现在写博客、公众号的人很多，大家的标题都起的很夸张、奇特来试图吸引人，让读者有点进去的欲望，增加更多的粉丝。忘记哪一天我突然觉得这不是一个很恰当的理由，有些太功利了，我写博客应该是为了积累，为了记录，也许以后自己可以复习；写博客不是为了写给别人看，当然有人因此得到帮助或者因此与我交流是一件顺带的好事，没有也不用强求。所以不必费尽心思去想些新颖的题目，不用关心语言是否幽默生动，想写啥写啥~

## 欢迎关注我的公众号

欢迎关注我的公众号，会经常分享一些技术文章和生活随笔～

今天公众号改名啦，改成**飞坤呀**

![飞坤呀](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/wechat.jpg)
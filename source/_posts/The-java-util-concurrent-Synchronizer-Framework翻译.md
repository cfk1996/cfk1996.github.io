---
title: The java.util.concurrent Synchronizer Framework翻译
date: 2020-03-05 16:23:40
tags:
    - Java
    - Java并发
---

最近在看AQS相关的源码，比较晦涩，阅读之前本打算自己也写一篇博客，看完后自知水平不够，写出来估计也不能很好的阐述AQS的理念。我也看了不少讲解AQS的文章，大多数讲解还是不够清晰。在美团技术博客中一篇讲解AQS的文章的引用中发现AQS作者自己写过相关论文，所以打算翻译一下，做一个小小搬运工。(不全部翻译，想看原版文末有链接)

读之前还是需要对AQS有一定的理解，可以看美团的这篇文章。
[从ReentrantLock的实现看AQS的原理及应用](https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html)

`The java.util.concurrent Synchronizer Framework`是java并发容器作者Doug Lea关于并发框架AQS的论文，阐述了AQS框架的原理、设计、实现、使用以及性能。

<!--more-->

## Introduction

JUC提供了不少同步器，比如`locks, read-write locks,
semaphores, barriers, futures, event indicators, and handoff
queues` 等。并且这些同步器可以很容易的被用来实现另一个同步器，比如用`ReentrantLock`来实现`Semaphore`.但是这样带来了很多的复杂性和不灵活性，且开发者想要实现自己的同步器时会犹豫选择哪一个，所以推出了`AbstractQueuedSynchronizer`，它提供了juc包里大多数同步器通用的机制，并且开发者可以基于它实现自己的同步类。

## Requirements

既然要设计一个通用的同步框架，先来思考一下这个框架要满足哪些需求。

* 功能上的需求

    >一个同步器至少需要实现两个方法：`acquire`方法来阻塞线程，直到同步状态允许它继续执行。`release`方法来修改同步状态，从而释放一个或多个正在阻塞的线程

    >但是JUC包现在并没有一个包含了`acquire`和`release`方法的接口，有些同步器实现了Lock接口，但是更多同步器都有自己的特殊的方法名。比如`Lock.lock`,`Semaphore.acquire`, `CountDownLatch.await`和`FutureTask.get`都可以看作是`acquire`。

    >如果有需要，同步器也可以实现trylock这种非阻塞的或者带timeout的同步方法，有的甚至需要实现中断逻辑

    >另外还需要区别是独占的还是共享的。大多数锁是独占的，但是信号量这种在count允许的范围内可以让多个线程acquire，同步框架需要支持两种模式。

* 性能上的需求

    >最重要的性能指标是`scalability`，甚至在发生同步冲突时保证有效率的。理想化的状态时，不管有多少个线程在竞争，都能以常数的消耗来通过同步点，所以这里的主要目标是以最小的时间消耗来决定哪些线程可以通过，哪些不能。但是这又需要考虑到资源消耗(cpu time, memory traffic,线程调度时间消耗)的问题，取得平衡。 比如自旋锁通常比阻塞锁需要更短的获取锁时间，但是浪费了cpu的周期和内存争用。

    >大多数应用需要最大化吞吐量，并容忍有可能出现的饥饿情况(一个线程一直阻塞，被其他线程抢占)。但是有些系统，比如说资源控制就希望以一种公平的方式来把资源分配给线程，容忍吞吐量的降低。框架不能决定同步器是公平的还是非公平的，所以需要支持不同的公平策略。

    >无论框架内部多精心的设计，同步竞争总会成为一些应用的瓶颈，所以框架还需要提供监控和检查基础操作的能力，让使用者可以发现并优化瓶颈。最少(也是最重要的)需要提供一个方法来说明有多少个线程block了。

## Design And Implementation

实现同步器最基本的想法是很直接的，伪代码如下：

```java
// acquire
    while (synchronization state does not allow acquire) {
        enqueue current thread if not already queued;
        possibly block current thread;
    }
    dequeue current thread if it was queued;

// release
    update synchronization state;
    if (state may permit a blocked thread to acquire)
        unblock one or more queued threads;
```

实现这两个方法，需要三个基础功能：

+ 原子的管理同步状态synchronization state
+ 阻塞线程，解除阻塞线程
+ 管理等待队列

实现一个框架让这三种功能独立的变化是有可能的，但这是低效且无用的。举个例子说明：队列节点保存的信息是unblock线程时所需要的，且这些操作都需要同步状态的参与。

所以AQS的核心设计思想是设计出这三个功能的一个具体实现，但在如何使用上提供了很多选择。我们有意的限制了同步框架的适用性，但是它足够有效以至于你不会有任何理由想要自己从头实现一个类似的框架。

### Synchronization State

AQS通过一个`volatile int`值来维护同步状态。并且暴露出`getState`,`setState`,`compareAndSetState`三个方法来获取和更新状态。

尽管JSR166规范提供了64位long类型的原子操作，还需要测试，未来可能会有64位的状态量，但目前没必要。

继承了AQS的具体类必须定义`tryAcquire`和`tryRelease`来控制获取锁和释放锁的操作。`tryAcquire`必须在线程acquire时返回true。`tryRelease`必须在同步状态可以在未来被acquire时返回true.同时这两个方法可以传入一个int参数，用来修改同步状态。

### Blocking

一直到JAR166,Java中都没有不依赖于内置monitor的API用来block和unblock线程。唯一的候选者Thread.suspend和Thread.resume还有一个没解决的问题（不管他了）。

现在JUC包里有一个`LockSupport`类解决了这个问题。`LockSupport.park`方法阻塞当前线程直到`LockSupport.unpark`被调用。

### Queues

框架的核心是管理存放阻塞线程的队列，AQS使用了FIFO队列。因此，框架不支持有优先级的同步。

目前主要有两种数据结构可以作为队列的候选者，AQS采用了`Craig, Landin, and Hagersten (CLH)`的变体，因为它更容易的被用于处理cancel和timeout。

CLH队列有两个原子更新的字段`head`和`tail`，初始化时指向一个虚节点。

一个新节点，通过一个原子操作入队
```java
do {
    pred = tail;
} while(!tail.compareAndSet(pred, node));
```
节点的release状态存放在前继节点，出队的过程如下：
```java
while (pred.status != RELEASED); // 自旋
head = node; // head指向自己，拿到锁
```

从上面可以看出，CLH变体的优点是：

+ 入队和出队很快、无锁、非阻塞的
+ 判断是否有线程在等待也是很快速的(head == tail)
+ release status保存在前一个节点，是去中心化的，防止内存竞争

在最原始的CLH锁中，节点没有前后指针。但是在AQS中将它改造了，因为拥有pred指针可以很好的处理cancel和timeout.当一个节点的pred节点cancel时，该节点可以连接到更前面一个节点，从而使用更前面节点的状态。

AQS也拥有一个next指针，但是因为没有一个可用的技术来无锁化的、原子的插入一个双向链表节点，所以next指针的赋值不是一个cas原子操作，而是通过`pred.next = node`。所以当node.next为空或者node.next是cancel状态时，可以从tail节点向前遍历，pred字段是原子赋值的。

AQS的第二个改动是使用节点内部的status变量来控制阻塞，而不是通过自旋。在这个同步框架中，一个排队的线程只有在它通过了AQS具体子类定义的`tryAcquire`方法时才能从`acquire`操作中返回，单单一个`released`状态是不满足的。
同时节点的status字段也可以用来减少不必要的`park`和`unpark`调用。在调用`park`之前，线程将status设置为`signal`,然后重新检查同步状态和节点状态。一个`release`线程会清除status，这两者配合就可以减少一些不必要的线程block。
第三个改动是gc相关，还有一些比较微小的如懒加载虚节点等
**(这部分关于CLH的太复杂了，看的不是很懂，想深入的可以看原文)**

抛开上面所有的细节不谈，最终最基础的`acquire`（独占模式，不支持中断，不支持timeout）方法实现是下面这样的：

```java
if (!tryAcquire(arg)) { // 尝试获取锁失败
    node = create and enqueue new code; // 创建新节点，并入队
    // 取有效的前继节点，signal 为cancel的是无效的，跳过
    pred = node‘s effective predecessor; 
    // 当前继节点不是head时，说明自己不在队列第一个
    // 不是第一个，尝试继续获取锁，获取失败进入while方法
    while (pred is not head node || !tryAcquire(arg)) {
        // 如果前继节点的状态已经被设置成signal
        // signal意味着，节点release时会去通知后面的节点
        // 所以后面的节点可以安心的进入阻塞状态，不用害怕无限阻塞
        if (pred‘s signal bit is set)
            park();
        // 如果前继节点的node status不是signal，通过cas设置
        // 设置失败也没事，可以进入下一次循环
        else
            compareAndSet pred‘s signal bit to true;
        // 更新有效的前继节点
        pred = node‘s effective predecessor;
    }
    // 当前节点是队列第一个，可以获取锁，将head指向自己
    head = node;
}
```

`release`实现如下：
```java
// 如果tryRelease成功并且当前节点状态为signal进入方法
if (tryRelease(arg) && head node‘s signal bit is set) {
    // 设置成false
    compareAndSet head‘s signal bit to false;
    // 唤醒后面的线程
    unpark head‘s successor, if one exists；
}
```

### Condition Queues

不多介绍。

## Useage

AQS使用了模版方法设计模式，子类只需要定义用于检查和更新状态的方法来控制`acquire`和`release`。JUC包中的所有同步器都把AQS的子类定义为内部私有类，因为内部控制`acquire`和`release`的策略不应该被使用者看到。

下面举个最简单的例子`Mutex`类，当同步状态为0时表示未上锁，1表示上锁。因为不需要arg参数，所以用0代替，可以忽略(AQS定义的方法有arg参数)。

```java
class Mutex {
    class Sync extends AbstractQueuedSynchronizer {
        public boolean tryAcquire(int ignore) {
            return compareAndSetState(0, 1);
        }
        public boolean tryRelease(int ignore) {
            setState(0);
            return true;
        }
    }

    private final Sync sync = new Sync();

    public void lock() {
        sync.acquire(0);
    }
    public void unlock() {
        sync.release(0);
    }
}
```
这个例子的完成版本和很多其他使用方法可以在J2SE文档中找到。

AQS通用提供类一系列方法来支持同步器实现不同的策略控制，比如它包括支持timeout和interrupt版本的`acquire`方法；又比如到目前为止讨论的都是独占锁，AQS也包括类一系列类似于`acquireShared `方法来支持共享锁。

尽管序列化一个同步器不是一件常见的事，但是同步器经常被用在线程安全的集合类中，而这些容器需要被序列化，所以AQS也支持序列化同步状态。

### Controlling Fairness

尽管AQS是基于FIFO队列，但是同步器不一定是公平的。可以看下前面最基础的`acquire`伪代码(一堆中文注释的那个)，它先尝试了`tryAcquire`，如果失败了才是去排队，所以一个新的`acquire`操作是有可能插队的,也就是非公平的。

这种抢先插入式的FIFO队列有更好的吞吐性能，如果是FIFO队列，当一个竞争的锁释放时，它会有一段时间内没有线程去持有锁，因为队列的第一个节点从线程阻塞中恢复需要消耗一定的时间；同时这种改良提高了并发度，因为不只是队列的第一个线程可以去竞争锁。开发者开发自己的同步器时，如果锁的持有时间都比较短的话，就需要更加关注插入的性能，通过多次tryAcquire来减少线程block的可能。但这种方式也有一个缺点，当插队的线程来的非常快和多的时候，队列里第一个线程将会一直抢不到锁，从而队列后续节点也都一直block住，造成饥饿现象。

如果需要较高的公平性，也非常的简单。只需要在`tryAcquire`方法中判断当前线程是不是head节点，不是的话返回false，然后进入排队逻辑。

除此之外，还有一个更多折中的方法，`tryAcquire`只在队列为空时可以进行插入，这样多个线程竞争时，可以省去入队的步骤。

这些权利都是留给开发者去考虑的。

### Synchronizers

下面看看JUC中的同步器如何使用AQS框架。

* ReentrantLock

    >使用同步状态来保存持有锁的数量(可重入)，当一把锁被获取时，它也记录了持有锁的线程用来判断重入以及检测其他线程调用unlock的异常。ReentrantLock支持可选的公平策略(内部实现来两个AQS子类)，根据构造时的参数，选择合适的那个子类。

* ReentrantReadWriteLock

    >之前提到同步状态用32位的int值存储，读写锁使用其中16位存储写锁的个数，剩下16位存储读锁的个数。写锁的结构与ReentrantLock相似，读锁是通过实现`acquireShared`方法来支持多个读者。

* Semaphore

    >信号量使用同步状态来保存count，它定义了`acquireShared`方法来减少count，当count非正的时候block住。定义了`tryRelease`来增加count，如果count变正了，去unblock其他线程。

* CountDownLatch

    >用同步状态来保存count，所有的acquire请求在count变为0时通过

* FutureTask

    >用同步状态代表future的运行状态(initial, running, cancelled,done)。

* SynchronousQueue

    >通过同步状态允许producer在cousumer取走东西后生产东西，反过来也一样。

## 总结

看完这些可以了解AQS的大致框架，后面我可能还会对照着去看源码来进一步的理解AQS.

## 引用

[The java.util.concurrent Synchronizer Framework](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)
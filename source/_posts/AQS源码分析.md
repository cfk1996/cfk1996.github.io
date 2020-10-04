---
title: AQS源码分析
date: 2020-10-04 11:46:33
tags:
    - Java
    - Java源码
---

`AbstractQueuedSynchronizer`是jdk中的中大多数同步控制器的底层并发框架，通过模版方法模式定义了一系列方法，子类只需要实现少数几个`protected`方法(tryAcquire, tryRelease)就能实现并发控制。 并发控制器一般会有一个继承自AQS的内部类，不同的控制器有不同的对外API,最终都会转换为对AQS内部类的操作，进而操作AQS底层数据结构。下面将从`ReentrantLock`和`Seamphore`等同步类的实现源码来分析AQS的作用。

<!-- more-->

之前还翻译过JUC作者关于AQS介绍的文章，不熟悉AQS的可以简单翻阅一下。[The java.util.concurrent Synchronizer Framework翻译](https://cfk1996.github.io/2020/03/05/The-java-util-concurrent-Synchronizer-Framework%E7%BF%BB%E8%AF%91/)



### 从ReentrantLock看AQS

`ReentrantLock`内部实现了两个继承于AQS的内部类，分别对应着公平策略和非公平策略。(后续的分析为了简单起见，不会去分析中断，timeout等机制，只会分析阻塞模式的方法)

```java
// Sync是内部继承了AQS的抽象类
private final Sync sync;
public ReentrantLock() {
    sync = new NonfairSync();
}
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

可重入锁的一个简单使用方式如下，后续将只分析非公平锁模式(没有硬性的公平性要求，建议非公平锁模式，效率更高)

```java
ReentrantLock lock = new ReentrantLock();
// ReentrantLock lock = new ReentrantLock(true); 公平模式
lock.lock();
try {
    // do something
} finally {
    lock.unlock();
}

// lock方法简单的代理了内部的Sync类的
public void lock() {
    sync.lock();
}
// 非公平锁策略下，sync=new NonfairSync()
// NonfairSync.lock()
final void lock() {
    // 非公平性体现在这里，首先会通过一次cas操作state变量
    // state后续介绍，是AQS中的成员变量
    // 0代表没有占用，大于1则代表有资源被占用
    if (compareAndSetState(0, 1))
        // 如果cas成功，说明该线程抢到资源，把线程设为独占线程
        // ReentrantLock是独占锁，一个线程抢到以后，其他线程必须等待
        setExclusiveOwnerThread(Thread.currentThread());
    else
        // 如果没有插队成功，则通过acquire进入后续操作
        acquire(1);
}
```

需要注意的是，上面的`compareAndSetState`,`setExclusiveOwnerThread`和`acquire`都是AQS中的方法，不需要同步器实现。至于同步器要实现什么，继续往下看吧。

#### AbstractQueuedSynchronizer.acquire

`compareAndSetState`,`setExclusiveOwnerThread`两个方法没有什么值得分析的，就是上面代码中注释的内容；`acquire`是一个模版方法模式，定义了内部的执行过程，下面分析一下`acquire`方法

```java
public final void acquire(int arg) {
    // tryAcquire是AQS未实现的方法，需要子类实现
    // 如果获取锁失败就入队等待
    // addWaiter将线程加入队列
    // acquireQueued尝试获取锁直到成功
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        // 中断相关，不知何用
        selfInterrupt();
}

// tryAcquire需要子类实现，我们先去ReentrantLock中找一下它的实现方法
// ReentrantLock.NonfairSync.tryAcquire()
protected final boolean tryAcquire(int acquires) {
    //调用NonfairSync父类Sync的方法
    return nonfairTryAcquire(acquires);
}

final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    // 如果c=0说明资源未被抢占
    if (c == 0) {
        // 通过cas执行修改state值
        if (compareAndSetState(0, acquires)) {
            // cas成功则代表抢锁成功，将当前线程设为独占线程
            setExclusiveOwnerThread(current);
            // 返回true则!tryAcquire直接结束，lock成功
            return true;
        }
    }
    // 如果c!=0并且当前线程就是独占线程，可重入
    else if (current == getExclusiveOwnerThread()) {
        // 增加state值
        int nextc = c + acquires;
        // 这里为什么可以在先读再写的过程中不用cas也不用加锁呢？
        // 因为就是该线程在占有锁，不会出现并发修改的情况
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        // lock成功
        return true;
    }
    // lock失败，进行后续的入队等待操作
    return false;
}
```

接下来回到`acquire`方法的剩余部分，执行一些入队操作。

AQS的队列结构在源码注释中有提到，示意图如下

          +------+  prev +-----+  next +-----+
     head |      | <---- |     | ----> |     |  tail
          +------+       +-----+       +-----+

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
// 如果tryAcquire失败，则先执行addWaiter再执行acquireQueued
// AQS.addWaiter
private Node addWaiter(Node mode) {
    // Node是最终放入队列的对象，封装了线程和节点状态
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    // 如果尾节点不为空，继续往队尾加
    if (pred != null) {
        // 先把当前node指向pred
        // 这步有点奇怪，如果cas失败了，node.prev的值很可能指向错误的节点
        // cas失败的解决方案，在后面的enq方法中
        node.prev = pred;
        // 通过cas将新创建的node节点设为tail
        if (compareAndSetTail(pred, node)) {
            // 成功则将更新前面节点的next指针
            // 从这可以看出是个双向队列
            pred.next = node;
            // 入队成功
            return node;
        }
    }
    // 现在没有尾节点，执行enq
    enq(node);
    // 返回当前节点
    return node;
}
// enq方法
private Node enq(final Node node) {
    // cas操作一般都是在死循环中
    for (;;) {
        Node t = tail;
        // 如果tail为null，则还未初始化
        if (t == null) {
            // 进行初始化
            // 通过cas设置head节点为一个空的Node
            if (compareAndSetHead(new Node()))
                // 初始化时tail和head指向同一个空节点
                tail = head;
        } else {// 队列初始化过
            // 将新节点的prev指针指向tail
            // 这里的node.prev=t与addWaiter中的不同之处在于
            // 这里是在死循环中，如果cas失败，会将t更新为最新的tail
            // 所以prev指向的最终会是正确的前置节点
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                // 更新前继节点的next指针
                t.next = node;
                入队成功
                return t;
            }
        }
    }
}
// addWaiter执行完后返回入队的节点Node，作为acquireQueued的参数
// acquireQueued方法
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        // 死循环
        for (;;) {
            // 拿到前继节点，node.prev
            // node.prev不应该为null
            final Node p = node.predecessor();
            // p==head说明当前node是队列的第一个
            // 通过tryAcquire尝试抢占资源(如果是公平锁，则一定成功)
            // 非公平锁入队前会先执行tryAcquire，可能存在竞争
            if (p == head && tryAcquire(arg)) {
                // 抢占资源成功后
                // 把node设为head节点
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // node不是队列中的第一个节点，则需要等待
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
// 在介绍shouldParkAfterFailedAcquire方法前插播一点Node类的介绍
// Node类除了prev, next, Thread外还有一个int型的状态量waitStatus
// 它只有0，1，-1，-2，-3几种取值
// 1表示线程cancel了
static final int CANCELLED =  1;
// -1表示当前节点的next节点需要unpark唤醒
static final int SIGNAL    = -1;
// -2表示当前线程在等待condition
static final int CONDITION = -2;
// 不知所云the next acquireShared should unconditionally propagate
static final int PROPAGATE = -3;
// 创建Node时，waitStatus未赋值，默认0，何时修改后面分析

// shouldParkAfterFailedAcquire方法
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        // 前置节点会唤醒自己的，可以安心的park(阻塞)了
        return true;
    if (ws > 0) {
        // 大于0说明前置节点cancel了，需要更新自己的前继节点到一个不大于0的节点
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        //更新前继节点的指针
        pred.next = node;
    } else {
        // 通过cas设置前继节点的waitStatus为-1
        // 如果设置成功则下次进入会在ws == Node.SIGNAL处return true
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    // 还不能阻塞，return后进入下一个循环
    return false;
}
//parkAndCheckInterrupt方法
private final boolean parkAndCheckInterrupt() {
    // 阻塞线程
    LockSupport.park(this);
    return Thread.interrupted();
}
```

到这里lock方法就已经介绍完了，也深入lock方法的内部，分析了AQS的相关代码(中断相关没有分析)，接下来以同样的方式看一下unlock的过程，进而继续解析AQS的源码

```java
// ReentrantLock.unlock调用了sync类的release方法
public void unlock() {
    sync.release(1);
}
// 与lock方法中的结构一样， release也是AQS中的模版方法
// AQS.release方法
public final boolean release(int arg) {
    // tryRelease需要子类实现
    if (tryRelease(arg)) {// 如果state=0了，资源完全释放了
        Node h = head;
        // 如果h!=null说明已经初始化过
        // waitStatus不等于0，说明后面节点已经设置过前继节点进入阻塞状态，需要唤醒
        // waitStatus等于0则说明，后面节点还在执行流程，不需要唤醒
        if (h != null && h.waitStatus != 0)
            // 唤醒后面阻塞的线程
            unparkSuccessor(h);
        return true;
    }
    //因为是可重入的，当前线程可能还持有锁
    return false;
}
// ReentrantLock.Sync.tryRelease方法
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    // state==0时意味着当前线程资源全部释放
    if (c == 0) {
        free = true;
        // 清除独占线程
        setExclusiveOwnerThread(null);
    }
    // 设置state
    setState(c);
    // return是否完全释放
    return free;
}
// tryRelease因为是完成线性的，不需要考虑并发，所以很简单
// 下面回到AQS的release方法中，还差一个unparkSuccessor方法没有介绍
// unparkSuccessor
private void unparkSuccessor(Node node) {
    // node为头节点
    int ws = node.waitStatus;
    if (ws < 0)
        // 将当前节点的status设为0
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;
    // 如果next为null或者waitStatus是取消状态
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 从尾节点向前找到最前面的waitStatus<=0的节点
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    // 如果s不为空，唤醒它对应的线程
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

到这里AQS的大部分代码都分析完了，AQS提供了并发框架的大部分基础功能，只需要子类实现tryAcquire和tryRelease方法来实现具体的并发逻辑，接下来再看看Semaphore中的这两个方法的实现细节来体会下其用处。

### 从Semaphore看AQS

`Semaphore`又叫做信号量，用来控制同时访问资源的线程个数。当信号量的个数限制为1时，可以当作锁来使用。当信号量个数大于1时，可以认为是共享模式的锁。

`Semaphore`也分为公平和非公平，其实只是不同的`tryAcquire`实现方式的差异。下面放一个简单的使用示例。

```java
public class Test {
    private int threadNum;
    private Semaphore semaphore;

    public Test(int permits,int threadNum, boolean fair) {
        this.threadNum = threadNum;
        semaphore = new Semaphore(permits,fair);
    }

    private void println(String msg){
        SimpleDateFormat sdf = new SimpleDateFormat("[YYYY-MM-dd HH:mm:ss.SSS] ");
        System.out.println(sdf.format(new Date()) + msg);
    }

    public void test(){
        for(int i =  0; i < threadNum; i ++){
            new Thread(() -> {
                try {
                    semaphore.acquire();
                    println(Thread.currentThread().getName() + " acquire");
                    Thread.sleep(5000);//模拟业务耗时
                    println(Thread.currentThread().getName() + " release");
                    semaphore.release();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }

    public static void main(String[] args) {
        // 10个线程抢占3个资源
        new Test(3, 10, true).test();
    }
}
```

下面来看看`semaphore.acquire()`和`semaphore.release()`吧，同时为了方便起见，分析不带中断版本的（非公平）

```java
public void acquireUninterruptibly() {
    // 共享模式
    sync.acquireShared(1);
}
// AQS.acquireShared
public final void acquireShared(int arg) {
    // tryAcquireShared由子类实现
    // 小于0意味着抢占资源失败，进行下一步等待操作
    if (tryAcquireShared(arg) < 0)
        // 获取资源
        doAcquireShared(arg);
}
// Semaphore.NonfairSync.tryAcquireShared
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}
// Semaphore.Sync.nonfairTryAcquireShared
final int nonfairTryAcquireShared(int acquires) {
    // 死循环
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        // 如果remain小于0则超出最大资源限制，return
        // 否则通过cas设置state
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
// 如果nonfairTryAcquireShared返回小于0的值，意味着抢占资源失败
// 需要进入doAcquireShared,结构与之前几乎一样，不再分析
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

Semaphore通过`tryAcquireShared`来维护state变量的值，当获取资源树木小于state时，则成功获取，反之失败，进入AQS的排队流程。下面看下release操作吧。

```java
public void release() {
    sync.releaseShared(1);
}
// AQS.releaseShared模版方法
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        // AQS的释放操作，不再分析，过程与前面出队操作类似
        doReleaseShared();
        return true;
    }
    return false;
}
//tryReleaseShared
protected final boolean tryReleaseShared(int releases) {
    //通过cas将state的值加上释放的资源数量
    for (;;) {
        int current = getState();
        int next = current + releases;
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next))
            return true;
    }
}
```

### 从CountDownLatch看AQS

CountDownLatch的使用场景如下：让数个线程一起执行某批任务，当这批任务全部完成时，才可以继续执行。简单的看下`tryAcquire`和`tryRelease`

```java
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
// 根据上面的分析，如果state！=0则return -1
// return -1时则会进入等待队列，然后阻塞
//与使用场景一致，主线程需要阻塞等待直到前面所有线程执行完毕

protected boolean tryReleaseShared(int releases) {
    // 线程执行完成后，将state-1
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```

### 欢迎关注我的公众号

欢迎关注我的公众号，会经常分享一些技术文章和生活随笔～

![飞坤呀](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/wechat.jpg)









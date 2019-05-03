---
title: Java线程池源码分析
date: 2019-05-02 16:49:29
tags:
    - Java
    - Java源码
---

多线程是进行并发编程时运用到的最主要的技术，由于频繁的创建销毁线程会带来较大的开销，因此又引申出线程池的概念。在需要一个新的线程来执行任务时，从池子中找出一个空闲的线程来执行任务，减少了线程重复创建的次数，减少CPU等资源的消耗。

经过JDK很好的封装，线程池使用起来非常的简单，只需要简单的参照一个示例代码，就能够使用上线程池，那么对于这种并发中最重要的技能，仅仅会用是不够的，必须深入理解其实现原理，那么一起从源码中一探究竟吧。

<!--more-->

首先在`Intellij IDEA`中打开`java.util.concurrent`目录，通过IDE可以生成concurrent包整体的层次结构图，对线程池技术的宏观了解有助于理解其底层实现，那么我从中截取了线程池相关的层次结构图如下：

![线程池类层次结构图](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2019/executor.png)

那么今天的分析将从按照`Executor->ExecutorService->AbsExeService->ThreadPoolExecutor`的路线展开，但是其中前三个是接口和抽象类，分析的重点还是在`ThreadPoolExecutor`上。`ForkJoinPool`和`Scheduled`相关的留到以后分析。

## Executor & ExecutorService

`Executor`是一个接口，它的实例对象是用来执行被提交的Runnable任务，这个接口解耦了任务的提交和任务如何被执行(选择线程，执行时间等)。

```java
// 接口非常简单，只定义了一个方法
public interface Executor {
    void execute(Runnable command);
}
```

`ExecutorService`接口拓展了`Executor`接口，提供了更多操纵线程的方法，比如管理线程的终止。`Executor`中定义的方法，无法获取线程执行的结果，无法了解线程执行状况，使用上有些许不便。因此`ExecutorService`还提供了可以返回Future对象的方法，Future对象可以理解为异步结果的占位符，可以通过它在之后某个时间获取异步操作的结果。

```java
public interface ExecutorService extends Executor {
    // 线程终止
    void shutdown();

    List<Runnable> shutdownNow();

    boolean isShutdown();

    boolean isTerminated();

    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
    // submit方法拓展了execute方法，提供了Future对象用于管理任务的执行
    <T> Future<T> submit(Callable<T> task);

    <T> Future<T> submit(Runnable task, T result);

    Future<?> submit(Runnable task);
    // 批量执行任务
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

## AbstractExecutorService分析

`ExecutorService`只是一个接口，定义了很多方法。`AbstractExecutorService`则对其中`submit`开始的方法提供了默认实现。但是把`execute`和终止相关的方法的实现留给了`ThreadPoolExecutor`。

简单介绍下`Future`,其对象可以看做是一个异步执行任务的占位符，提供了`get`,`cancel`等方法，可以获取异步任务的结果，也可以取消异步任务的执行。比如在A线程中，开启了线程B去执行一个任务，那么A线程就无法了解B的状态，但是通过`Future f = B.doAsync()`,就可以通过Future绑定到执行结果。线程A就能从而得到B的执行结果。

```java
public abstract class AbstractExecutorService implements ExecutorService {
    // submit方法拓展了execute方法，使其可以返回一个Future对象
    // submit方法都会检测任务不为空
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
    // 绑定Future和task    
        RunnableFuture<Void> ftask = newTaskFor(task, null);
    // 真正的执行逻辑还是exectue,在具体类中实现    
        execute(ftask);
    // 返回future对象    
        return ftask;
    }
    // 逻辑类似，不重复分析
    public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task, result);
        execute(ftask);
        return ftask;
    }
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }
    // 绑定task和future对象
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }
    // ...
    // invokeAll, invokeAny不介绍
```

## Executors类

介绍完`AbsExecutorService`，应该到了`ThreadPoolExecutor`，但是这里还需要提到一个结构图中没有出现的类`Executors`，它提供了大量的`Executor`和`EexcutorService`的工厂函数和工具方法，简化了线程池的创建和使用，使开发者在不了解线程池的原理的情况下也可以很快上手。线程池最佳实践中很少直接由开发者创建一个`ThreadPoolExecutor`对象，而是通过Executors提供的几个常用的工厂函数来创建线程池对象。`Executors`中提供了很多的静态工厂方法，那么我选出最常用的以及跟这篇文章息息相关的几个。

```java
// 创建一个线程数量固定的线程池，线程池数量为nThreads
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
        0L, TimeUnit.MILLISECONDS,
        new LinkedBlockingQueue<Runnable>());
}
// 通过指定的threadFactory来创建线程
// 可以设置线程名字，优先级等
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(nThreads, nThreads,
        0L, TimeUnit.MILLISECONDS,
        new LinkedBlockingQueue<Runnable>(),
        threadFactory);
}
// 创建一个可无限增长的线程池(注意场景，放止创建过多线程，内存溢出)
// 省略需要传入threadFactory的工厂函数
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
        60L, TimeUnit.SECONDS,
        new SynchronousQueue<Runnable>());
}
// 创建只有一个线程的线程池
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService(
        new ThreadPoolExecutor(1, 1,
        0L, TimeUnit.MILLISECONDS,
        new LinkedBlockingQueue<Runnable>()));
}
```

这里只列出了`ThreadPoolExecutor`相关的工厂方法，`Executors`中还包括了`ForkJoinPool`和`Scheduled`的工厂方法。

列出的工厂函数最终都是调用了`ThreadPoolExecutor`的构造函数，那么根据几个方法的作用及参数的组合，聪明的读者已经可以大概猜出构造函数中每个变量的意义。还没有注意到的可以回头比较一下几个工厂方法参数的差异，相信会有一些收获。

## ThreadPoolExecutor分析

绕了这么多，终于来到今天的主角，线程池的实现类`ThreadPoolExecutor`.

在没分析之前，首先思考一下如果自己来实现一个线程池，需要考虑哪些问题？

+ 如何创建线程，何时创建线程(初始化时全部创建？按需创建？)
+ 创建多少个线程？
+ 任务多于线程数量时，如何处理任务？
+ 线程池状态获取，管理
+ 空闲线程如何处理？

### 构造函数

```java
// ThreadPoolExecutor继承了AbsExecutorService
public class ThreadPoolExecutor extends AbstractExecutorService {
    ...
}
// 进入主题，从构造函数开始分析
// 构造函数的参数非常多，所以有多个构造函数
// 那么无论哪一个构造函数，最终都会调用该构造函数
/* @param corePoolSize 核心线程数，即使处于空闲状态也会保存在线程池中，
 *        除非设置了allowCoreThreadTimeOut
 * @param maximumPoolSize 线程池最大可以拥有的线程数，超过核心线程数的
 *        线程会在空闲超过指定时间后销毁
 * @param keepAliveTime 超过核心线程数的线程的最大空闲时间
 * @param unit 时间单位
 * @param workQueue 用于保存还未执行的任务
 * @param threadFactory 线程工厂，用于创建新的线程
 * @param handler 当线程池大小边界和任务队列的边界到达时执行的拒绝策略
 */
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    // 对几个变量的值进行合法性校验                          
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    // 变量合法性校验    
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
        null : AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

构造函数中只是进行了简单的赋值操作，没有创建线程等相关操作，那么就继续从线程池的使用方法着手分析。

线程池通过`ExecutorService`的`submit`方法执行任务，而在前面的分析中得到`AbsExeService`中的默认实现最终调用了`ThreadPoolExecutor`中的`execute`方法。

### execute()方法分析

```java
// 内部使用一个原子型整数保存两个状态，runState和workerCount
// 整数的高3为表示运行状态，包括RUNNING,STOP,SHUTDOWN等5个状态
// 其余位数表示线程的数量
// 因为将两个变量放在一个整数里，所以对ctl变量提供了打包、解包方法
// ctlof方法用于将runState和workerCount打包成一个整数
// 初始化时runState=RUNNING, workerCount=0
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
// runState详细信息
// 接收任务并且处理排队的任务     
private static final int RUNNING    = -1 << COUNT_BITS;
// 不接受新任务，处理队列中的任务
private static final int SHUTDOWN   =  0 << COUNT_BITS;
// 不接受新任务，不处理排队任务，中断正在处理的任务
private static final int STOP       =  1 << COUNT_BITS;
// 所有任务已经终止，线程转换为TIDYING状态会调用钩子方法terminated()
private static final int TIDYING    =  2 << COUNT_BITS;
// terminated()方法执行完成
private static final int TERMINATED =  3 << COUNT_BITS;

public void execute(Runnable command) {
    // 判断command飞空
    if (command == null)
        throw new NullPointerException();
    // 读取ctl当前的值    
    int c = ctl.get();
    // workerCountOf方法解包ctl,读取其中线程数量的值
    // 如果当前工作线程小于核心线程数，调用addWorker()添加一个核心线程
    if (workerCountOf(c) < corePoolSize) {
        // 调用addWorker成功的话直接返回
        if (addWorker(command, true))
            return;
        // addWorker失败，则重新读取ctl，可能的失败原因后面分析    
        c = ctl.get();
    }
    //线程数大于核心线程数时，尝试将任务放在队列中
    // 如果runState=RUNNING并且任务队列添加command成功
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 如果队列添加成功，仍然需要执行双重检查
        // 避免在上次检查到目前的时间内，一个线程销毁了
        // 或者线程池ShutDown了
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    } 
    // 如果任务队列不能添加任务，尝试创建新的线程
    // 这里的addWorker线程添加的线程是核心线程之外的线程
    else if (!addWorker(command, false))
        // 如果创建失败，则拒绝改任务
        reject(command);
}

// 下面分析下execute需要用到的addWorker()方法
/* @param firstTask 新线程的第一个任务
 * @param core 为true时以corePoolSize当做边界，否则maxPoolSize
 */   
// 注意这个core的不同取值，区分是否添加核心线程
// 可以回到上面的execute方法调用addWorker时的参数，体会下 
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);
        // 根据上面分析的runState方法，在某些状态需要检查队列是否为空
        //正常情况runState为RUNNING，除非调用shutDown等方法
// 之前看过一篇博客，介绍如何分析源码。讲到一定要关注重点，这种异常情况如果
// 想要研究清楚，比较耗时，且意义不大。跟着正常的逻辑阅读下去，会更容易理解
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
                firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            //根据core的取值，选择以corePoolSize还是maximumPoolSize作边界
            // 如果线程池数量达到边界，则拒绝创建新线程，返回false
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // 通过cas操作增加线程数量，添加成功跳出retry循环    
            if (compareAndIncrementWorkerCount(c))
                break retry;    
            c = ctl.get();
            // 如果runState发生变化，重新进入外层retry循环  
            if (runStateOf(c) != rs)
                continue retry;
            // 否则是cas修改workerCount失败，重新进行cas操作
        }
    }
    // 通过cas将workerCount自增后进入这
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        //Worker的构造函数通过threadFactory的newThread()创建一个新线程
        w = new Worker(firstTask);
        // 获取worker创建的线程
        final Thread t = w.thread;
        // threadFactory创建线程成功
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                int rs = runStateOf(ctl.get());
                //重新检查runState状态，RUNNING < SHUTDOWN
                //也就是说正常情况会进入该分支
                //或者处于shutdown状态，但是队列中还有任务
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive())
                        throw new IllegalThreadStateException();
            // workers = new HashSet<Worker>();
            //将新的worker加入集合          
                    workers.add(w);
                    int s = workers.size();
                    //更新最大线程数
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                //修改workerAdded状态        
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                //执行线程
                t.start();
                workerStarted = true;
            }
        }
    // 返回addWorker的结果，成功为true    
    } finally {
        if (! workerStarted)
        //添加worker失败调用addWorkerFailed方法
        //addWorkerFailed方法在workers集合中删除w,并将workerCount-1
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

## Worker分析

看完`execute()`和`addWorker()`，已经对线程池如何工作有一定了解了，现在唯一的困惑可能就是Worker封装了线程后做了些什么，一起来看看。

```java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable {
    final Thread thread;
    Runnable firstTask;
    //worker已经完成任务的个数
    volatile long completedTasks;

    Worker(Runnable firstTask) {
    // AbstractQueuedSynchronizer方法，不分析了    
        setState(-1);
        //赋值firstTask,可能为空    
        this.firstTask = firstTask;
        //通过threadFactory的newThread方法创建新的线程
        //并把当前worker实例传入，Worker实现了Runnable接口，符合线程创建规则
        this.thread = getThreadFactory().newThread(this);
    }
    // 当调用thread.start()方法后执行run方法
    // 将任务委派给ThreadPoolExecutor的runWorker方法
    public void run() {
        runWorker(this);
    }
    //...省略一些方法
}
```

### runWorker()方法

`runWorker()`封装了线程的执行逻辑。

```java
final void runWorker(Worker w) {
    //获取到当前线程
    Thread wt = Thread.currentThread();
    //获取到需要执行的任务
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // 允许中断
    boolean completedAbruptly = true;
    try {
        // 如果firstTask不为空或者队列中还有任务
        //getTask()不仅仅是获取任务这么简单，后面介绍
        while (task != null || (task = getTask()) != null) {
            w.lock();
            //当线程池停止时，对线程进行中断
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                //执行钩子方法，ThreadPoolExecutor默认是无操作
                //可以通过继承ThreadPoolExecutor来修改其实现
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    //执行任务逻辑
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    //执行钩子函数
                    afterExecute(task, thrown);
                }
            } finally {
                //执行完成后，回收task,将已完成任务数量加1
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}

// 分析下runWorker中调用的getTask()方法
private Runnable getTask() {
    // 标志是否超时
    boolean timedOut = false; 

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);
        // 如果runState状态为shutdown之后，忽略队列中的任务
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }
        int wc = workerCountOf(c);
        //allowCoreThreadTimeOut为false(默认)时，核心线程即使空闲也不会回收
        //如果线程数大于核心线程数，timed=true
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
// 如果线程数大于最大线程数或者有多余的空闲线程超时了
// 并且线程数大于1或者队列为空时，将ctl减1
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }
        try {
        // 有多于核心线程数的线程时，timed为true
        // 则从任务队列中用poll()方法取任务，阻塞keepAliveTime秒的时间
        // 如果线程数小于等于核心线程数，则无限期的阻塞等待队列中的第一个元素   
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            // 如果队列中有任务，返回该任务并执行    
            if (r != null)
                return r;
            // 队列中没有任务，标记超时，则在下一次循环中会将ctl减1
            // 并退出循环，然后进入runWorker方法中的processWorkerExit()销毁线程
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

这里稍微对上面总结下，一个`Worker`就对应了线程池中的一个线程，通过`runWorker()`方法执行任务，线程工作时首先执行传入`Worker`对象的`firstTask`,之后从`workQueue`中获取任务并执行。当线程数小于等于核心线程数时，从队列中获取任务时是无限期等待的，即没有任务时，会一直阻塞在获取任务的状态；但是当线程数多于核心线程数时，从阻塞队列中获取元素的阻塞时间为`keepAliveTime`, 超时后退出`runWorker`中的**while**循环，进入`processWorkerExit()`方法执行线程的销毁动作。

### processWorkerExit()方法

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    // completedAbruptly用来判断ctl当前的值是否已经是应该存活线程的数量
    // 因为getTask方法中已经对ctl减1，所以runWorker方法传入的应该是false
    // 即不需要再次对ctl的值减1
    if (completedAbruptly) 
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 整个线程池已经完成的任务数
        completedTaskCount += w.completedTasks;
        // 从Worker集合中删除Worker w
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }
    // tryTerminate()方法在RUNNING状态下不进行任何操作，直接返回
    // 其他runState情况不过多介绍
    tryTerminate();
    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return;
        }
        addWorker(null, false);
    }
}
```

## 拒绝策略

在`execute`方法中，如果`addWorker()`失败，则会执行拒绝策略。

```java
final void reject(Runnable command) {
    handler.rejectedExecution(command, this);
}
// 方法非常简单，调用我们传入的RejectedExecutionHandler对象的方法执行拒绝策略
// 下面介绍几个简单的拒绝策略
// (1)抛出异常
public static class AbortPolicy implements RejectedExecutionHandler {

    public AbortPolicy() { }
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException("Task " + r.toString() +
            " rejected from " + e.toString());
    }
}
// (2)直接忽略新的任务
public static class DiscardPolicy implements RejectedExecutionHandler {

    public DiscardPolicy() { }
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {}
}
// (3)忽略队列中最后一个任务
public static class DiscardOldestPolicy implements RejectedExecutionHandler {

    public DiscardOldestPolicy() { }
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            e.getQueue().poll();
            e.execute(r);
        }
    }
}
```

## 总结

看到这里，关于线程池的核心部分都已经知其然且知其所以然，还有一些修改内部状态，终止线程的方法就不过多介绍了。

再用一张图总结一下线程池的工作流程

![线程池工作流程](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/2019/threadPool.jpg)

除了图中的流程，还需要记得线程池通过阻塞队列的poll()方法超时，来将空闲的多余线程(大于核心线程数的线程)销毁。

## 往期回顾

[Java集合源码系列-HashMap](https://cfk1996.github.io/2019/01/12/Java%E9%9B%86%E5%90%88%E6%BA%90%E7%A0%81%E7%B3%BB%E5%88%97-HashMap/)

[Java集合源码系列-ArrayList](https://cfk1996.github.io/2019/01/22/Java%E9%9B%86%E5%90%88%E6%BA%90%E7%A0%81%E7%B3%BB%E5%88%97-ArrayList/)

[Java动态代理原理剖析(一)](https://cfk1996.github.io/2019/02/28/Java%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86%E5%8E%9F%E7%90%86%E5%89%96%E6%9E%90/)

## 欢迎关注我的公众号

欢迎关注我的公众号，会经常分享一些技术文章和生活随笔～

![技术旅途](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/wechat.jpg)
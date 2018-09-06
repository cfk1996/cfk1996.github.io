---
title: Python多线程
date: 2018-05-29 19:12:18
tags:
    - Python
---
最近在学习并发编程，打算写一个并发编程系列的文章。之前也看过很多Python多线程多进程的教程、博客等，但收益不大。因为大多数文章上来就是写几个对照代码，不去解释那些库方法的作用，然后得出多线程多进程确实快。这种教程相当于让我死记硬背，这是我无法接受的。

经过长期摸索，我发现了一个神奇的网址，那就是[The Python Standard Library](https://docs.python.org/3/library/index.html).这在我刚学Python时就了解过，但当时看到整页整页的英文我就头大，现在慢慢适应了，发现它确实很好，对标准库的介绍非常详细。如果可以接受直接去看它就行，我的文章只是在它之上的一个翻译和总结。

## 基础概念

学习并发编程首先要了解一些基本概念，线程，进程，协程，IO等概念，不了解的可以学习一下操作系统。

## GIL

Python有个历史遗留问题，那就是全局解释器锁，一个进程同一个时刻只有一个线程在执行，如果将多线程用于CPU计算密集行工作可能效果不如单线程，但是在I/O这种耗时操作方面还是有用的，这方面可以查阅GIL相关资料。
<!--more-->
## threading

threading是Python标准的多线程接口，是对底层的_thread模块的封装， 使多线程用起来更加方便。

这个模块定义了如下几个函数(列举部分用到的)：

1. threading.active_count(): 返回当前存活的线程数量

1. threading.current_thread(): 返回当前线程对象

1. threading.enumerate(): 返回一个列表，列表包含所有存活的线程对象

1. threading.main_thread()： 返回主线程对象

### 创建线程

有两种方法可以新创建一个线程对象，都基于Thread类。第一种是传一个可调用的对象（一般是函数）给Thread的构造器，第二种是继承Thread类，重写它的run方法。

一旦thread对象被创建，需要调用对象的start()方法让他在一个单独的线程中运行，即运行run方法。

### class threading.Thread

class threading.Thread(group=None, target=None, name=None, args=(), kwargs={}, *, daemon=None)。

构造Thread类时传入的参数必须是关键字参数。

group可忽略，target为需要执行的函数func，name为线程的名字（没实际意义，可忽略），args为func需要的参数元祖(注意是元祖，元祖的单个形式为`(arg,)`)，kwargs为func需要的关键字参数，daemon设置是否为守护进程。

**start()**： 启动线程

**run()**： 线程运行后执行的函数

**join()**: This blocks the calling thread until the thread whose join() method is called is terminated.比如在A线程中调用B.join()，A就会阻塞，直到B运行结束，看个例子。

```python
from threading import Thread

def hello(num):
    print(num)

if __name__ == '__main__':
    threads = []
    for i in range(10):
        t = Thread(target=hello, args=(i,))
        threads.append(t)
    for t in threads:
        t.start()
        t.join()
    print('1111111')
# output: 节省篇幅，用，代替换行
# 0， 1, 2, 3, 4, 5, 6, 7, 8, 9, 1111111
if __name__ == '__main__':
    ...
    for t in threads:
        t.start()
    print('22222')
# output:
# 0, 1, 2, 3, 4, 6, 5, 22222, 8, 7, 9
```

注意Print('22222')的位置。the calling thread在这里就是主线程，the thread whose join is called在这里是threads列表里的对象。所以这句话意思就是主线程会一直阻塞，直到所有threads里的thread执行完毕。如果不调用join()方法，主线程就不会等待子线程的执行。

## 同步机制

不同的线程可能需要对同一个资源进行修改，如果不加控制，会出现意想不到的结果，来看个神奇的例子。

```python
count = 0
def add():
    global count
    temp = count + 1
    time.sleep(0.0001)
    count = temp

threads = []

for i in range(10):
    t = Thread(target=add)
    threads.append(t)
for t in threads:
    t.start()
for t in threads:
    t.join()

print('count={}'.format(count))
# output: 运行三次的结果
# 3, 4， 4
```

正常来说count的值应该是10,因为我调用了10个线程。但结果却是3或4或其他数，原因在于time.sleep()和多线程并发，time的作用只是让线程有机会交叉执行。考虑两个线程并发执行。


| A线程       | B线程    |
| :------: | :-----:  |
|     global count(count=0)   |      |
|     temp = count+1（temp=1)    |      |
|      time.sleep()  |       |
| | global count(count=0) |
|  | temp = count+1 (temp=1) |
|  | time.sleep() |
| count = temp(count=1) |  |
|  | count = temp(count=1) |


如果不加控制，并发操作会导致意料不到的后果，所以需要采取一些手段来控制并发的执行顺序。

### Lock 锁

锁有两个状态，上锁了(locked)和没上锁(unlocked)。一个锁对象创建时是没上锁的状态。锁有两个基本函数，acquire()用来上锁，release()用来释放锁。如果锁处于上锁状态，不能调用acquire();属于未上锁状态，不能调用release()，否则会引起错误。对上面一个例子进行加锁处理。

```python
count = 0

def add(lock):
    global count
    lock.acquire()
    temp = count + 1
    time.sleep(0.0001)
    lock.release()
    count = temp

threads = []
lock = Lock()

for i in range(10):
    t = Thread(target=add, args=(lock,))
    threads.append(t)
for t in threads:
    t.start()
for t in threads:
    t.join()

print('count={}'.format(count))
# output:
# 10
```

锁如果不恰当控制会出现死锁的情况，关于死锁的原因和解决办法在操作系统中也有涉及，不了解的学习一下操作系统，我这篇文章只介绍多线程模块的基本用法。

### RLock 可重入锁

对于这个锁我有点困惑，想不到它的实际应用场景，可重入锁建立在锁的基础上，获得锁A的线程可以对锁A进行再次加锁而不会陷入死锁，但释放操作也需要相同次数，但是其他线程无法在锁住的情况下获得锁，同样对上面例子进行修改。还没找到一个很有说服力的需要加多次锁的例子。

```python
# ...不变
def add(lock):
    global count
    lock.acquire()
    temp = count + 1
    lock.acquire()
    time.sleep(0.0001)
    lock.release()
    count = temp
    lock.release()
# ...不变
```

### Condition 条件

条件也是建立在锁的基础上，在创建条件对象时可以传入一个锁或可重入锁，不传入参数则默认生成一个可重入锁。一些线程A等待条件而阻塞，一些线程B发出条件满足的信号，则等待的线程A可以继续运行。线程A首先acquire()加锁，在wait()处释放锁并阻塞，当其他线程发出条件满足信号，发出条件满足信号的方法有两个，一个是notify()，唤醒一个等待线程，另一个是notify_all()唤醒所有等待线程;A线程的wait()重新加锁并返回。经典的场景可能就是消费者和生产者模式。

```python
def consumer(con):
    con.acquire()
    print('{} is waiting'.format(threading.currentThread().name))
    con.wait()
    print('{} cousumes one time.'.format(threading.currentThread().name))
    con.release()

def producer(con):
    print('prodece and notify')
    con.acquire()
    con.notify_all()
    con.release()

threads, condition = [], Condition()

for i in range(5):
    t = Thread(target=consumer, args=(condition,))
    threads.append(t)

for t in threads:
    t.start()

produce = Thread(target=producer, args=(condition,))
produce.start()
produce.join()
for t in threads:
    t.join()
print('end')
# output:
# Thread-1 is waiting
# Thread-2 is waiting
# Thread-3 is waiting
# Thread-4 is waiting
# Thread-5 is waiting
# prodece and notify
# Thread-1 cousumes one time.
# Thread-4 cousumes one time.
# Thread-5 cousumes one time.
# Thread-2 cousumes one time.
# Thread-3 cousumes one time.
# end
```

### Semaphore 信号量

方法与Lock()一样，但是可以在创建信号量时可以传入一个大于０的整数，当传入的整数为１时与Lock作用相同。信号量一般用于限制并发的数量，如连接数据库服务器时需要在连接数量控制在一定范围内。

```python
def hello(sem):
    sem.acquire()
    time.sleep(1)
    print('{} is running'.format(threading.currentThread().name))
    sem.release()

threads, sem = [], Semaphore(5)

for i in range(100):
    t = Thread(target=hello, args=(sem,))
    threads.append(t)
    t.start()

for t in threads:
    t.join()
```

可以在自己的电脑上运行一下，会看到print结果是5个5个的出现，sleep时间长点效果更明显。可以看到虽然开了100个线程，但是同时在运行只有5个。

### Event 事件

一个线程标识一个事件，其他线程等待它。一个事件对象管理一个内部标志，这个标志可以用set()置为true,用clear()置为false。wait()方法会一直阻塞直到标志为true。这个看起来似乎跟条件有点像。

```python
def consumer(event):
    print('{} is waiting for an event.'.format(threading.currentThread().name))
    event.wait()
    print('{} finish.'.format(threading.currentThread().name))

def producer(event):
    print('producer')
    event.set()

threads, event = [], Event()

for i in range(3):
    t = Thread(target=consumer, args=(event,))
    threads.append(t)
    t.start()

produce = Thread(target=producer, args=(event,))
produce.start()
for t in threads:
    t.join()
produce.join()
print('end')
# output:
# Thread-1 is waiting for an event.
# Thread-2 is waiting for an event.
# Thread-3 is waiting for an event.
# producer
# Thread-2 finish.
# Thread-3 finish.
# Thread-1 finish.
# end
```

### 上下文管理器

上下文管理协议是Python的一个语法糖吧，用来方便资源的申请与释放。实现了上下文管理协议的类就可以用with语句包裹，简化代码的书写。具体实现查阅资料。

上面介绍的那么多同步机制大多数都有一个申请锁和释放锁的步骤，可以用with语句简化这些操作，支持with语法的有lock, Rlock, conditions和semaphore.官网示例如下：

```python
with some_lock:
    # do something
```

等价于

```python
some_lock.acquire()
try:
    # do something
finally:
    some_lock.release()
```

## 最后

如果还有疑问，标准库是你最好的选择！刚开始看也许看不下去，有这么几个建议。

1. 看简短一点的模块
2. 积累专业词汇量
3. 找自己熟悉的模块入手，这样你就能大概的猜出它的意思
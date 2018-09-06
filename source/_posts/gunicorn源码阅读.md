---
title: gunicorn源码阅读
date: 2018-08-13 17:03:59
tags:
    - Python
---
## 入口

首先程序的入口为`gunicorn/app/wsgiapp`这个模块。

```python
def run():
    """\
    The ``gunicorn`` command line runner for launching Gunicorn with
    generic WSGI applications.
    """
    from gunicorn.app.wsgiapp import WSGIApplication
    WSGIApplication("%(prog)s [OPTIONS] [APP_MODULE]").run()


if __name__ == '__main__':
    run()
```

`WSGIApplication`这个类继承于`Application`,然后继承于`BaseApplication`.而且这三个类只有`BaseApplication`是有构造函数的。

```python
def __init__(self, usage=None, prog=None):
    self.usage = usage
    self.cfg = None
    self.callable = None
    self.prog = prog
    self.logger = None
    self.do_load_config()
```

这里面`useage, prog`就是两个字符串，忽略，其他的下面分析。赋值完后进入`do_load_config`方法。这个方法做了两件事，第一件是将一个Config对象赋值`给self.cfg`参数。这个对象可以从命令行中解析参数，将一些配置绑定。第二件是调用一个在`Application`中才实现的方法`load_config`。这个方法通过各种途径将参数绑定到cfg对象中,其中包括调用一次`WSGIApplicagion`的init方法，同样也是绑定相关参数。

<!--more-->
但这里有个比较神奇的技巧，关于cfg的，一开始没看懂，看到后来发现cfg中包含了很多可以使用的方法，却不知道是什么时候偷偷绑定上来的。现在来仔细看一下，之前说过了，cfg就是一个`Config`对象。

```python
KNOWN_SETTINGS = []

def make_settings(ignore=None):
    settings = {}
    ignore = ignore or ()
    for s in KNOWN_SETTINGS:
        setting = s()
        if setting.name in ignore:
            continue
        settings[setting.name] = setting.copy()
    return settings

class Config(object):

    def __init__(self, usage=None, prog=None):
        self.settings = make_settings()
        ...
```
目前来看，`KNOWN_SETTINGS`是一个空列表，所以`self.settings`也应该是一个空字典。但其实不然。

```python
class SettingMeta(type):
    def __new__(cls, name, bases, attrs):
        super_new = super(SettingMeta, cls).__new__
        parents = [b for b in bases if isinstance(b, SettingMeta)]
        if not parents:
            return super_new(cls, name, bases, attrs)

        attrs["order"] = len(KNOWN_SETTINGS)
        attrs["validator"] = staticmethod(attrs["validator"])

        new_class = super_new(cls, name, bases, attrs)
        new_class.fmt_desc(attrs.get("desc", ""))
        KNOWN_SETTINGS.append(new_class)
        return new_class

class Setting(object):
    pass

Setting = SettingMeta('Setting', (Setting,), {})

class Workers(Setting):
    name = 'xxx'
    ...
    validator = xxx
    pass
```

config.py这个模块中还有很多个类似`Workers`一样的类，结构都是差不多的，首先都是继承`Setting`类，而`Setting`类是一个由`SettingMeta`创造出来的类，大家应该都知道创造类是__new__这个方法来完成的，这里也不例外，在__new__方法中，通过type这个元类来生成一个新的类，并通过`attrs["validator"] = staticmethod(attrs["validator"])`来给类绑定一个方法。同时将新的`Setting`类加入`KNOWN_SETTINGS`中，这样后续定义的类似`Workers`的类都会被加入列表中，从而绑定到cfg这个对象上。

简单的说，在调用run方法之前，初始化了一些参数，主要是给cfg这个对象绑定了很多熟悉和方法。

## run方法

`run`方法最终调用的是`Arbiter`对象的run方法，创建`Arbiter`对象时传入`Application`对象作为参数。根据类注释，可以很清楚的了解这个类的主要作用。

```python
class Arbiter(object):
    """
    Arbiter maintain the workers processes alive. It launches or
    kills them if needed. It also manages application reloading
    via SIGHUP/USR2.
    """
    ...
        def run(self):
        "Main master loop."
        self.start()
        ...
        try:
            self.manage_workers()

            while True:
                self.maybe_promote_master()

                ...
        ...
        except Exception:
            ...
            sys.exit(-1)
```
在`Arbiter`的run方法中，先调用start()来创建socket监听,然后通过manage_workers()来控制worker的数量，现在来看下`manage_workers`的代码。

```python
    def manage_workers(self):
        """\
        Maintain the number of workers by spawning or killing
        as required.
        """
        if len(self.WORKERS.keys()) < self.num_workers:
            self.spawn_workers()
        workers = self.WORKERS.items()
        workers = sorted(workers, key=lambda w: w[1].age)
        while len(workers) > self.num_workers:
            (pid, _) = workers.pop(0)
            self.kill_worker(pid, signal.SIGTERM)
        ...

    def spawn_workers(self):
        """\
        Spawn new workers as needed.

        This is where a worker process leaves the main loop
        of the master process.
        """

        for _ in range(self.num_workers - len(self.WORKERS.keys())):
            self.spawn_worker()
            time.sleep(0.1 * random.random())

    def spawn_worker(self):
        self.worker_age += 1
        worker = self.worker_class(self.worker_age, self.pid, self.LISTENERS,
                                   self.app, self.timeout / 2.0,
                                   self.cfg, self.log)
        self.cfg.pre_fork(self, worker)
        pid = os.fork()
        if pid != 0:
            worker.pid = pid
            self.WORKERS[pid] = worker
            return pid

        # Do not inherit the temporary files of other workers
        for sibling in self.WORKERS.values():
            sibling.tmp.close()

        # Process Child
        worker.pid = os.getpid()
        try:
            util._setproctitle("worker [%s]" % self.proc_name)
            self.log.info("Booting worker with pid: %s", worker.pid)
            self.cfg.post_fork(self, worker)
            worker.init_process()
            sys.exit(0)
        except SystemExit:
            raise
        ...
```

如果worker少于cfg.num_workers，调用spawn_workers方法增加worker数量，增加的方法就是os.fork()。
如果数量大于cfg.num_workers，根据worker.age的属性排序后kill一个worker。

我们主要看下增加worker的过程，增加worker是通过调用os.fork来实现的，调用os.fork的进程称为主进程，生成的进程称为子进程，对于这两个进程，os.fork的返回值是不一样的，子进程的返回值是0，父进程返回的是子进程的进程id。所以如果是主进程则记录子进程id后返回到run里的无限循环。如果是子进程，则成为一个worker进程，执行worker.init_process()。正常情况不会执行`sys.exit(0)`语句。

我们现在回到刚才os.fork的主进程，他执行完os.fork后就返回到run里的无限循环.
```python
        try:
            self.manage_workers()

            while True:
                self.maybe_promote_master()

                sig = self.SIG_QUEUE.pop(0) if self.SIG_QUEUE else None
                if sig is None:
                    self.sleep()
                    self.murder_workers()
                    self.manage_workers()
                    continue

                if sig not in self.SIG_NAMES:
                    self.log.info("Ignoring unknown signal: %s", sig)
                    continue

                signame = self.SIG_NAMES.get(sig)
                handler = getattr(self, "handle_%s" % signame, None)
                if not handler:
                    self.log.error("Unhandled signal: %s", signame)
                    continue
                self.log.info("Handling signal: %s", signame)
                handler()
                self.wakeup()
        except StopIteration:
            self.halt()
        except KeyboardInterrupt:
            self.halt()
        except HaltServer as inst:
            self.halt(reason=inst.reason, exit_status=inst.exit_status)
        except SystemExit:
            raise
        except Exception:
            self.log.info("Unhandled exception in main loop",
                          exc_info=True)
            self.stop(False)
            if self.pidfile is not None:
                self.pidfile.unlink()
            sys.exit(-1)
```

主进程在执行`maybe_promote_master`方法，将自己标识为master进程，然后根据信号量来进行一些控制进程的操作。如果信号量为空，则通过sleep方法进入睡眠状态，sleep的代码是这样的：

```python
    def sleep(self):
        """\
        Sleep until PIPE is readable or we timeout.
        A readable PIPE means a signal occurred.
        """
        try:
            ready = select.select([self.PIPE[0]], [], [], 1.0)
            if not ready[0]:
                return
            while os.read(self.PIPE[0], 1):
                pass
        except (select.error, OSError) as e:
            # TODO: select.error is a subclass of OSError since Python 3.3.
            error_number = getattr(e, 'errno', e.args[0])
            if error_number not in [errno.EAGAIN, errno.EINTR]:
                raise
        except KeyboardInterrupt:
            sys.exit()
```
循环的监听管道，如果有信号量就退出循环，关于select这一块我也不是很清楚。退出循环后回到上一段的循环中,首先保持worker的数量为配置信息里的值，然后读取信号量的名字，根据不同的名字调用不同的hander方法。之后不断的重复，master进程大概就是这样。


## Worker进程

通过上面的分析，可以看出来worker进程才是真正用来处理请求的进程，入口是`worker.init_process()`.这个worker的来历大概是这样的，`worker -> self.work_class(*args) -> self.cfg.worker_class() -> util.load_class()`。util.load_class接受一个字符串参数，是配置中的`worker_class`变量，默认为SyncWorker。但是也能变成gevent, threadworker等更高效的worker.我们先看下默认的SyncWorker的逻辑是怎么样的。

所有的worker模块都在gunicorn/workers包中。`SyncWorker`继承自base.Worker.`SyncWorker`的init_process()方法来自于父类。

```python
    def init_process(self):
        """\
        If you override this method in a subclass, the last statement
        in the function should be to call this method with
        super(MyWorkerClass, self).init_process() so that the ``run()``
        loop is initiated.
        """

        # set environment' variables
        if self.cfg.env:
            for k, v in self.cfg.env.items():
                os.environ[k] = v

        util.set_owner_process(self.cfg.uid, self.cfg.gid,
                               initgroups=self.cfg.initgroups)

        # Reseed the random number generator
        util.seed()

        # For waking ourselves up
        self.PIPE = os.pipe()
        for p in self.PIPE:
            util.set_non_blocking(p)
            util.close_on_exec(p)

        # Prevent fd inheritance
        for s in self.sockets:
            util.close_on_exec(s)
        util.close_on_exec(self.tmp.fileno())

        self.wait_fds = self.sockets + [self.PIPE[0]]

        self.log.close_on_exec()

        self.init_signals()

        # start the reloader
        if self.cfg.reload:
            def changed(fname):
                self.log.info("Worker reloading: %s modified", fname)
                self.alive = False
                self.cfg.worker_int(self)
                time.sleep(0.1)
                sys.exit(0)

            reloader_cls = reloader_engines[self.cfg.reload_engine]
            self.reloader = reloader_cls(extra_files=self.cfg.reload_extra_files,
                                         callback=changed)
            self.reloader.start()

        self.load_wsgi()
        self.cfg.post_worker_init(self)

        # Enter main run loop
        self.booted = True
        self.run()
```

1. init_signals()注册信号量
2. load_wsgi()： self.wsgi = self.app.wsgi()，一般就是python框架里起的app，比如Flask里的`app = Flask(__name__)`.
3. run(). 现在我们到syncworker的run方法看一看。

```python
    def run(self):
        timeout = self.timeout or 0.5

        for s in self.sockets:
            s.setblocking(0)

        if len(self.sockets) > 1:
            self.run_for_multiple(timeout)
        else:
            self.run_for_one(timeout)
    
    def run_for_one(self, timeout):
        listener = self.sockets[0]
        while self.alive:
            self.notify()
            try:
                self.accept(listener)
                continue
            except EnvironmentError as e:
                if e.errno not in (errno.EAGAIN, errno.ECONNABORTED,
                        errno.EWOULDBLOCK):
                    raise

            if not self.is_parent_alive():
                return

            try:
                self.wait(timeout)
            except StopWaiting:
                return

    def run_for_multiple(self, timeout):
        while self.alive:
            self.notify()
            try:
                ready = self.wait(timeout)
            except StopWaiting:
                return

            if ready is not None:
                for listener in ready:
                    if listener == self.PIPE[0]:
                        continue

                    try:
                        self.accept(listener)
                    except EnvironmentError as e:
                        if e.errno not in (errno.EAGAIN, errno.ECONNABORTED,
                                errno.EWOULDBLOCK):
                            raise

            if not self.is_parent_alive():
                return
```

我把一些注释删了，run方法之后进入的两个方法同样也都是无限循环，不断的接收socket。`accept`方法很简洁，就是在建立连接的socket上获取client端的地址等信息，并设置socket为阻塞的，也就是同一时间只能处理一个请求。然后调用handle方法处理请求，handle方法如下：

```python
    def handle(self, listener, client, addr):
        req = None
        try:
            if self.cfg.is_ssl:
                client = ssl.wrap_socket(client, server_side=True,
                    **self.cfg.ssl_options)

            parser = http.RequestParser(self.cfg, client)
            req = six.next(parser)
            self.handle_request(listener, req, client, addr)
        except http.errors.NoMoreData as e:
            self.log.debug("Ignored premature client disconnection. %s", e)
        except StopIteration as e:
            self.log.debug("Closing connection. %s", e)
        except ssl.SSLError as e:
            if e.args[0] == ssl.SSL_ERROR_EOF:
                self.log.debug("ssl connection closed")
                client.close()
            else:
                self.log.debug("Error processing SSL request.")
                self.handle_error(req, client, addr, e)
        except EnvironmentError as e:
            if e.errno not in (errno.EPIPE, errno.ECONNRESET):
                self.log.exception("Socket error processing request.")
            else:
                if e.errno == errno.ECONNRESET:
                    self.log.debug("Ignoring connection reset")
                else:
                    self.log.debug("Ignoring EPIPE")
        except Exception as e:
            self.handle_error(req, client, addr, e)
        finally:
            util.close(client)

    def handle_request(self, listener, req, client, addr):
        environ = {}
        resp = None
        try:
            self.cfg.pre_request(self, req)
            request_start = datetime.now()
            resp, environ = wsgi.create(req, client, addr,
                    listener.getsockname(), self.cfg)
            # Force the connection closed until someone shows
            # a buffering proxy that supports Keep-Alive to
            # the backend.
            resp.force_close()
            self.nr += 1
            if self.nr >= self.max_requests:
                self.log.info("Autorestarting worker after current request.")
                self.alive = False
            respiter = self.wsgi(environ, resp.start_response)
            try:
                if isinstance(respiter, environ['wsgi.file_wrapper']):
                    resp.write_file(respiter)
                else:
                    for item in respiter:
                        resp.write(item)
                resp.close()
                request_time = datetime.now() - request_start
                self.log.access(resp, req, environ, request_time)
            finally:
                if hasattr(respiter, "close"):
                    respiter.close()
        except EnvironmentError:
            # pass to next try-except level
            six.reraise(*sys.exc_info())
        except Exception:
            if resp and resp.headers_sent:
                # If the requests have already been sent, we should close the
                # connection to indicate the error.
                self.log.exception("Error handling request")
                try:
                    client.shutdown(socket.SHUT_RDWR)
                    client.close()
                except EnvironmentError:
                    pass
                raise StopIteration()
            raise
        finally:
            try:
                self.cfg.post_request(self, req, environ, resp)
            except Exception:
                self.log.exception("Exception in post_request hook")
```
了解过wsgi协议的应该知道，服务器是如何跟框架交互的。简单的说就是服务器会调用一个方法并传入两个参数，第一个参数为environ,这个参数包含了所有请求有关的信息，比如headers, body等等。第二个参数是一个回调函数，后台服务处理完业务后调用这个函数将response传给服务器，服务器再传给客户端。但是这里还有很多细节，水平有限，看不大明白，但是整体的流程应该还是很清楚。所以这里先parser了http请求的相关信息，保存在environ中，然后生成回调函数resp.strt_response，然后调用wsgi(environ, resp.start_response)。这里的wsgi就是框架中的app.

## GeventWorker

我最近接触到的是配合gevent起一个服务，所以我也分析一下geventworker的逻辑。首先geventworker继承自asyncworker，asyncworker继承自base.worker。上面提到了，默认的worker是一个阻塞的模型，同一时间只能处理一个请求，所以效率比较低，生产环境一般不会使用。

### AsyncWorker

`AsyncWorker`的构造函数先是调用了父类的构造函数，然后又添加了一个额外的参数`worker_connections`，这个参数也是在cfg中设置的，且只在`eventlet`和`gevent`两种模式下起作用，作用是限制最大的同时的客户端连接数。

前面的SyncWorker的init_process是继承自worker。但是GeventWorker重写了这个方法。用过gevent的应该知道，gevent底层实现的方法叫做猴子补丁-monkey_patch。修改了大多数的底层库，将一些阻塞的底层实现，重新换成非阻塞的。所以GeventWorker先是打上补丁，然后调用worker的init_process方法，最终进入GeventWorker的run方法开始执行处理请求任务。run方法代码如下：
```python
    def run(self):
        ...
        for s in self.sockets:
            s.setblocking(1)
            pool = Pool(self.worker_connections)
            if self.server_class is not None:
                environ = base_environ(self.cfg)
                environ.update({
                    "wsgi.multithread": True,
                    "SERVER_SOFTWARE": VERSION,
                })
                server = self.server_class(
                    s, application=self.wsgi, spawn=pool, log=self.log,
                    handler_class=self.wsgi_handler, environ=environ,
                    **ssl_args)
            else:
                hfun = partial(self.handle, s)
                server = StreamServer(s, handle=hfun, spawn=pool, **ssl_args)

            server.start()
            servers.append(server)

        while self.alive:
            self.notify()
            gevent.sleep(1.0)

        try:
            # Stop accepting requests
            for server in servers:
                if hasattr(server, 'close'):  # gevent 1.0
                    server.close()
                if hasattr(server, 'kill'):  # gevent < 1.0
                    server.kill()

            # Handle current requests until graceful_timeout
            ts = time.time()
            while time.time() - ts <= self.cfg.graceful_timeout:
                accepting = 0
                for server in servers:
                    if server.pool.free_count() != server.pool.size:
                        accepting += 1

                # if no server is accepting a connection, we can exit
                if not accepting:
                    return

                self.notify()
                gevent.sleep(1.0)

            # Force kill all active the handlers
            self.log.warning("Worker graceful timeout (pid:%s)" % self.pid)
            for server in servers:
                server.stop(timeout=1)
        except:
            pass
```
1. 创建tcpServer。并用pool限制了最大连接数。这个server的实现在gevent中，没看懂。
2. hfun这个方法，是一个绑定了参数的handle，是asyncWorker的handle。过程跟前面的同步的差不多。但是遇到阻塞是gevent会帮助切换，所以提高了并发量。
3. 创建完server进入无限循环，notify网上查了一下说是给Arbiter发信号的，这里我不大懂。


## 总结

gunicorn代码比较多，且有很多底层的东西。很多地方不懂，都跳过了，分析可能也有很多错误，看到可以指出。

相比于之前看过的flask、request、tornado等等。gunicorn显然难很多，也没有那么清晰，有很多方法，参数来的不明不白；而且跟gevent牵扯很大，gevent的代码更加难懂。

但应该还是有点收获吧，虽然暂时没察觉到～

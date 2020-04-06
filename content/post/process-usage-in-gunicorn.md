---
title: "gunicorn中的进程管理"
date: 2020-04-04T20:13:41+08:00
lastmod: 2020-04-04T20:13:41+08:00
draft: false
keywords: ["gunicorn", "进程"]
description: ""
tags: []
categories: []
author: ""

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
autoCollapseToc: false
postMetaInFooter: false
hiddenFromHomePage: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false

# You unlisted posts you might want not want the header or footer to show
hideHeaderAndFooter: false

# You can enable or disable out-of-date content warning for individual post.
# Comment this out to use the global config.
#enableOutdatedInfoWarning: false

flowchartDiagrams:
  enable: true 
  options: ""

sequenceDiagrams: 
  enable: true
  options: ""

---

gunicorn是python中一个主流的wsgi server，本文只会关注其中关于进程管理的部分，顺便也会涉及其他关于进程的东西。

gunicorn基于pre-fork工作模型，也就意味着它有一个核心进程，管理一组工作进程，核心进程不关心也不知道具体每个工作进程都做了写什么，所有的请求也只会被工作进程处理。

总结来说，核心进程管理工作进程的生命周期，工作进程处理具体的请求，干活。

## # 主进程都做了些什么

先删掉不必要的细节，看一下代码：

```python
    def run(self):
        # 系统自检，注册signal handler等
        self.start()

        try:
            self.manage_workers()

            while True:
                sig = self.SIG_QUEUE.pop(0) if self.SIG_QUEUE else None
                if sig is None:

                    # Sleep until PIPE is readable or we timeout.
                    # A readable PIPE means a signal occurred.
                    self.sleep()

                    # kill unused/idle workers
                    self.murder_workers()

                    # Maintain the number of workers by spawning or killing
                    # as required.
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
            # 主进程退出
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
            sys.exit(-1)
```

可以看到主进程主要做了几件事情：

- 服务起来时候，管理启动指定数量对我工作进程
- 进入`main loop`循环，维护工作进程

    1. 检查有无signal产生
    2. 如果没有信号产生，休眠指定时间或者被信号唤醒，然后检查各个工作进程的心跳，如果有超出心跳时间的进程，kill之, 之后检查工作进程数量是否合法
    3. 如果有信号根据不同的信号，调用对应的handler

### # 处理系统信号

主进程中，处理系统信号的逻辑如下：

```python
    SIG_QUEUE = []
    SIGNALS = [getattr(signal, "SIG%s" % x)
               for x in "HUP QUIT INT TERM TTIN TTOU USR1 USR2 WINCH".split()]
    SIG_NAMES = dict(
        (getattr(signal, name), name[3:].lower()) for name in dir(signal)
        if name[:3] == "SIG" and name[3] != "_"
    
        )

    def start(self):
		...
        self.init_signals()


    def init_signals(self):
        """\
        Initialize master signal handling. Most of the signals
        are queued. Child signals only wake up the master.
        """
        # close old PIPE
        for p in self.PIPE:
            os.close(p)

        # initialize the pipe
        self.PIPE = pair = os.pipe()
        for p in pair:
            util.set_non_blocking(p)
            util.close_on_exec(p)

        self.log.close_on_exec()

        # initialize all signals
        for s in self.SIGNALS:
            signal.signal(s, self.signal)
        signal.signal(signal.SIGCHLD, self.handle_chld)

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

    def signal(self, sig, frame):
        if len(self.SIG_QUEUE) < 5:
            self.SIG_QUEUE.append(sig)
            self.wakeup()

    def wakeup(self):
        """\
        Wake up the arbiter by writing to the PIPE
        """
		# 唤醒主循环
        try:
            os.write(self.PIPE[1], b'.')
        except IOError as e:
            if e.errno not in [errno.EAGAIN, errno.EINTR]:
                raise
```

可以看到gunicorn处理系统信号的方式非常trick:

- 首先会生成一个管道`pipe`
- 把非`SIGCHLD`的信号注册到同一个handler, 这个handler在信号产生时候，就直接把该信号扔进队列，同时往pipe里面发消息，唤醒`main loop`
- 把`SIGCHLD`注册到对应的handler，如果有信号发生，直接处理
- 主进程的`main loop`的`sleep`会不停的在`pipe`上监听，直到有信号发生或者timeout（这个方法真的有点帅）

### # 主进程对不同信号的处理

主进程对不同信号的反应当然是不同的，gunicorn中对系统信号的使用也非常值得学习。

#### `SIGCHLD`

首先看一下这个信号代表什么含义：

> This signal is sent to a parent process whenever one of its child processes terminates or stops.

当一个进程的一个子进程`terminate`或者被`stop`时候，它自己会收到一个`SIGCHLD`的信号。

在gunicorn里面，对应的handler为`handle_chld`

```python
    def handle_chld(self, sig, frame):
        "SIGCHLD handling"
        self.reap_workers()
        self.wakeup()

    def reap_workers(self):
        """\
        Reap workers to avoid zombie processes
        """
        try:
            while True:
				# 检查所有结束的子进程，立刻返回
                wpid, status = os.waitpid(-1, os.WNOHANG)
                if not wpid:
                    break
                if self.reexec_pid == wpid:
                    self.reexec_pid = 0
                else:
                    # A worker was terminated. If the termination reason was
                    # that it could not boot, we'll shut it down to avoid
                    # infinite start/stop cycles.
                    exitcode = status >> 8
                    if exitcode == self.WORKER_BOOT_ERROR:
                        reason = "Worker failed to boot."
                        raise HaltServer(reason, self.WORKER_BOOT_ERROR)
                    if exitcode == self.APP_LOAD_ERROR:
                        reason = "App failed to load."
                        raise HaltServer(reason, self.APP_LOAD_ERROR)

                    worker = self.WORKERS.pop(wpid, None)
                    if not worker:
                        continue
                    worker.tmp.close()
                    self.cfg.child_exit(self, worker)
        except OSError as e:
            if e.errno != errno.ECHILD:
                raise
```

- 利用os.waitpid清除已经退出的进程占用的资源，比如进程号
- 如果子进程退出的原因为不能启动，整个服务down掉
- 子进程退出时候，主进程会轮询查看是哪个子进程退出了，并从注册列表`self.WORKERS`里面清理掉
- 如果有子进程退出时候的钩子函数，执行之

#### `SIGHUP`

> The SIGHUP (“hang-up”) signal is used to report that the user’s terminal is disconnected, perhaps because a network or telephone connection was broken. For more information about this, see Control Modes.

SIGHUP一般表示terminal挂断。在一个terminal启动一个前台或者后台进程，当和这终端断开链接时候，终端中启动的所有进程都会收到这个信号，默认会退出。但是daemon进程在启动时候已经与终端分离了（通过两次fork）,所以收不到这个信号。

生产环境的gunicorn当然是daemon进程，所以这个信号本着节约的原则，被服务用为了重启。

```python
    def handle_hup(self):
        """\
        HUP handling.
        - Reload configuration
        - Start the new worker processes with a new configuration
        - Gracefully shutdown the old worker processes
        """
        self.log.info("Hang up: %s", self.master_name)
        self.reload()
```

需要注意，gunicorn在重载配置之后，会新开同样数目的新子进程，然后再把老的干掉。

#### `SIGTERM`

> The SIGTERM signal is a generic signal used to cause program termination. Unlike SIGKILL, this signal can be blocked, handled, and ignored. It is the normal way to politely ask a program to terminate.
> The shell command kill generates SIGTERM by default.

handler会直接抛`StopIteration`，`main loop`里面会进行拦截，然后给所有的子进程发送`SIGTERM`信号

#### `SIGINT`

> The SIGINT (“program interrupt”) signal is sent when the user types the INTR character (normally C-c). See Special Characters, for information about terminal driver support for C-c.

这个信号等同`CTRL+C`
给所有子进程发SIGQUIT信号之后，主进程抛`StopIteration`退出

#### `SIGQUIT`

> Ctrl-\ sends a QUIT signal (SIGQUIT); by default, this causes the process to terminate and dump core.

给所有子进程发SIGQUIT信号之后，主进程抛`StopIteration`退出

#### `SIGTTIN`和`SIGTTOU`

> The SIGTTIN and SIGTTOU signals are sent to a process when it attempts to read in or write out respectively from the tty while in the background. Typically, these signals are received only by processes under job control; daemons do not have controlling terminals and, therefore, should never receive these signals.

当后台运行的进程试图从tty中读或者写的时候，就会分别触发这两个信号，所有在job control之下的进程都会受到这个信号。后台进程已经离开terminal，所以永远收不到这两个信号。

生产环境的gunicorn当然收不到这两个信号，所以可以用作它用。

在gunicorn中，这两个信号分别对应增加一个工作进程或者减少一个工作进程。

```python
    # self.num_workers += 1 # ttin
    self.num_workers -= 1 # ttou
    self.manage_workers()
```

#### `SIGUSR1` 

杀掉所有的子进程

#### `SIGUSR2`

对gunicorn进程热升级。
这个就很帅了！具体看代码。

## 心跳的实现

```python
    def murder_workers(self):
        """\
        Kill unused/idle workers
        """
        if not self.timeout:
            return
        workers = list(self.WORKERS.items())
        for (pid, worker) in workers:
            try:
                if time.time() - worker.tmp.last_update() <= self.timeout:
                    continue
            except (OSError, ValueError):
                continue

            if not worker.aborted:
                self.log.critical("WORKER TIMEOUT (pid:%s)", pid)
                worker.aborted = True
                self.kill_worker(pid, signal.SIGABRT)
            else:
                self.kill_worker(pid, signal.SIGKILL)
```

总结来说：
	- 子进程在fork之前，先打开一个tempfile
	- 子进程利用os.fchmod定期修改tmpfile的ctime
	- 主进程轮询每个子进程的tmpfile的ctime，对比当前时间，以此判断心跳是否超时

除了叹服还能说什么呢。
其中还有一个涉及到tempfile的技巧，为了不泄露tempfile，创建之后unlink一下就好了,想起自己之前使用tempfile一直在人肉remove简直了。。。

## 总结

gunicorn在进程的管理方面毫无疑问是非常聪明且有意思的：

1. 利用子进程改文件ctime时间，主进程不停的轮询达到心跳的目的
2. 利用各种系统信号，进行IPC
3. gunicorn对待SIGCHLD信号和其他信号的不同处理也值得仔细思考一下🤔

### 参考：

* http://www.linusakesson.net/programming/tty/
* http://curiousthing.org/sigttin-sigttou-deep-dive-linux
* https://stackoverflow.com/questions/11886812/whats-the-difference-between-sigstop-and-sigtstp
* https://stackoverflow.com/questions/881388/what-is-the-reason-for-performing-a-double-fork-when-creating-a-daemon
* https://www.gnu.org/software/libc/manual/html_node/Standard-Signals.html#Standard-Signals
* https://en.wikipedia.org/wiki/Signal_(IPC)

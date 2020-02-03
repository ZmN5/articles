---
title: "怎么理解异步(2)"
date: 2018-12-30T09:23:43+08:00
lastmod: 2018-12-30T09:23:43+08:00
draft: false
keywords: []
description: ""
tags: ["async", "yield from"]
categories: ["async"]
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
  enable: false
  options: ""

sequenceDiagrams: 
  enable: false
  options: ""

---

先回顾一下上一篇文章[怎么理解异步(1)](https://zmn5.github.io/post/how-to-study-async/)

对异步编程来说，需要代码段在把控制权移交给loop时候，保存当前`continuation`, 以便于事件发生的时候，当前代码段可以继续执行，换句话说，也就是：

程序可以`暂停`,`开始`, 想象代码段是一台机器，上面有两个按钮`暂停`, `开始`, 这说的不就是Python里面的`协程`!!!

先来看维基百科怎么讲协程的

> Coroutines are computer-program components that generalize subroutines for non-preemptive multitasking, by allowing multiple entry points for suspending and resuming execution at certain locations. 

主要看这面这段话:

> Subroutines are special cases of coroutines.[3] When subroutines are invoked, execution begins at the start, and once a subroutine exits, it is finished; an instance of a subroutine only returns once, and does not hold state between invocations. By contrast, coroutines can exit by calling other coroutines, which may later return to the point where they were invoked in the original coroutine; from the coroutine's point of view, it is not exiting but calling another coroutine.[3] Thus, a coroutine instance holds state, and varies between invocations; there can be multiple instances of a given coroutine at once. The difference between calling another coroutine by means of "yielding" to it and simply calling another routine (which then, also, would return to the original point), is that the relationship between two coroutines which yield to each other is not that of caller-callee, but instead symmetric.

英语渣就不翻译了，大概是：

对子例程来说:

1. 子例程只是协程的一种特例
2. 子例程从代码开始处执行，一旦退出就结束
3. 在多次调用之间并没有状态保存

对协程来说：

1. 可以通过调用其他协程退出
2. 协程可以在多次被唤醒之间保存状态

还有[Stack Overflow](https://stackoverflow.com/questions/553704/what-is-a-coroutine)中的一个回答：

> Coroutines and concurrency are largely orthogonal. Coroutines are a general control structure whereby flow control is cooperatively passed between two different routines without returning.
> 
> The 'yield' statement in Python is a good example. It creates a coroutine. When the 'yield ' is encountered the current state of the function is saved and control is returned to the calling function. The calling function can then transfer execution back to the yielding function and its state will be restored to the point where the 'yield' was encountered and execution will continue.


### 一、用协程改写爬虫

之前说过，利用回调实现异步的话，会很容易丢失`控制流信息`，而且`共享状态`困难，有一个可以`保存控制流信息`和`共享状态`的办法就是用同步方式进行异步编程

我们希望在一个函数内，执行`send`, `recv`操作，并且不退出函数即可得到结果，也就是希望:

> 告诉`selector`说, 我希望执行`send/recv`然后暂停，执行完毕之后给我结果，我继续执行

那么，该怎么做到这个呢？？？

> 就比如张三给老板说，有砖的时候通知我一下，我来搬？那么老板会问，我该通知你？

张三会说：

> `以后`有砖的时候放进`篮子/仓库`

所以我们就可以把这种终将有东西放进去的地方叫做`Future`(或者叫`Promise`)

那么真的有砖的时候，谁来真的来通知张三呢，总不能真的老板`selector`自己动手吧，这时候需要`秘书`或者随便什么人，就叫`Task`吧, 负责叫醒张三起床干活，通知张三结果

```python
import socket
from selectors import DefaultSelector, EVENT_WRITE, EVENT_READ

selector = DefaultSelector()


class Future:
    def __init__(self):
        self.callbacks = []
        self.result = None

    def set_result(self, result):
        self.result = result
        for cb in self.callbacks:
            cb(self)  # self not result

    def add_callback(self, cb):
        self.callbacks.append(cb)


class Fetcher:
    def __init__(self):
        self.response = b''
        self.sock = None

    def fetch(self):
        self.sock = socket.socket()
        self.sock.setblocking(False)
        try:
            self.sock.connect(('www.zhihu.com', 80))
        except BlockingIOError:
            pass
        f = Future()

        def on_connected(key, mask):
            f.set_result(None)

        selector.register(self.sock.fileno(), EVENT_WRITE, on_connected)
        yield f
        selector.unregister(self.sock.fileno())
        print('connected')
        request = 'GET / HTTP1.1\r\nHost: www.zhihu.com\r\n\r\n'
        self.sock.send(request.encode('ascii'))
        while True:
            f = Future()

            def on_readable(key, mask):
                f.set_result(self.sock.recv(4096))

            selector.register(self.sock.fileno(), EVENT_READ, on_readable)

            chunk = yield f
            selector.unregister(self.sock.fileno())
            if chunk:
                self.response += chunk
            else:
                print(f'recv: {len(self.response)}')
                break


class Task:
    def __init__(self, coro):
        self.coro = coro
        f = Future()
        f.set_result(None)
        self.step(f)

    def step(self, future):
        try:
            next_future = self.coro.send(future.result)
        except StopIteration:
            return
        next_future.add_callback(self.step)


def loop():
    while True:
        events = selector.select()
        for event_key, event_mask in events:
            cb = event_key.data
            cb(event_key, event_mask)


Task(Fetcher().fetch())
loop()
```

可以看到，所有的`控制流信息`,`状态信息`都在一个函数内，几乎跟同步一样的模式, 但是很明显代码还是有点丑

### 二、yield from 改写代码

张三搬砖久了，成了包工头，所以他把最无聊琐碎的`read`工作交给别人`李四`去做，他只负责通知李四干活，最后李四吧完整的结果`response`返回给他就行。

```python
import socket
from selectors import DefaultSelector, EVENT_WRITE, EVENT_READ

selector = DefaultSelector()


class Future:
    def __init__(self):
        self.callbacks = []
        self.result = None

    def set_result(self, result):
        self.result = result
        for cb in self.callbacks:
            cb(self)  # self not result

    def add_callback(self, cb):
        self.callbacks.append(cb)


class Fetcher:
    def __init__(self):
        self.response = b''
        self.sock = None

    def fetch(self):
        self.sock = socket.socket()
        self.sock.setblocking(False)
        try:
            self.sock.connect(('www.zhihu.com', 80))
        except BlockingIOError:
            pass
        f = Future()

        def on_connected(key, mask):
            f.set_result(None)

        selector.register(self.sock.fileno(), EVENT_WRITE, on_connected)
        yield f
        selector.unregister(self.sock.fileno())
        print('connected')
        request = 'GET / HTTP1.1\r\nHost: www.zhihu.com\r\n\r\n'
        self.sock.send(request.encode('ascii'))
        self.response = yield from read_all(self.sock)
        print(f'recv: {len(self.response)}')


def read(sock):
    f = Future()

    def on_readable(key, mask):
        f.set_result(sock.recv(4096))

    selector.register(sock.fileno(), EVENT_READ, on_readable)
    chunk = yield f
    selector.unregister(sock.fileno())
    return chunk


def read_all(sock):
    response = []
    chunk = yield from read(sock)
    while chunk:
        response.append(chunk)
        chunk = yield from read(sock)

    return b''.join(response)


class Task:
    def __init__(self, coro):
        self.coro = coro
        f = Future()
        f.set_result(None)
        self.step(f)

    def step(self, future):
        try:
            next_future = self.coro.send(future.result)
        except StopIteration:
            return
        next_future.add_callback(self.step)


def loop():
    while True:
        events = selector.select()
        for event_key, event_mask in events:
            cb = event_key.data
            cb(event_key, event_mask)


Task(Fetcher().fetch())
loop()

```

参考资料：

1. [Coroutine](https://en.wikipedia.org/wiki/Coroutine#Comparison_with_threads)
2. [A Web Crawler With asyncio Coroutines](http://www.aosabook.org/en/500L/a-web-crawler-with-asyncio-coroutines.html)

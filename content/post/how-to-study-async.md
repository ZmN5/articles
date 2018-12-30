---
title: "怎么理解异步?(1)"
date: 2018-12-29T20:52:03+08:00
lastmod: 2018-12-29T20:52:03+08:00
draft: false 
keywords: []
description: ""
tags: []
categories: []
author: ""

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: false
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

<!--more-->

### 零

从刚开始学Python到现在，一直对异步都只有一个很模糊的印象。

1. 什么是event_loop，event和loop是到底什么关系？
2. 具体的业务和异步框架到底是怎么交互的?
3. 协程、生成器、回调到底是什么关系?
4. 协程和线程是什么关系?

### 一、先从同步说起

如果是一个普通的同步爬虫，会是怎么工作的：

1. 建立连接
2. 分块读取内容

```python
import socket

def fetch():
    sock = socket.socket()
    sock.connect(('www.baidu.com', 80))
    request = f'GET / HTTP1.0\r\nHost: baidu.com\r\n\r\n'
    sock.send(request.encode('ascii'))
    response = b''
    chunk = sock.recv(4096)
    while chunk:
        response += chunk
        chunk = sock.recv(4096)

    parse_links(response)


def parse_links(response):
    print(f'recv: {len(response)} bytes')


fetch()
```

这段代码中需要注意的是什么？读取的每个分块的内容都会存进`本地变量`中，读取完毕后返回。

### 二、 轮询异步

在上面的代码中， `connect`和`recv`会阻塞代码等待操作完成，如果不想在这里阻塞会该怎么样呢

```python
import socket


def fetch():
    sock = socket.socket()
    sock.setblocking(False)
    try:
        sock.connect(('www.baidu.com', 80))
    except BlockingIOError:
        pass

    send(sock)
    response = recv(sock)
    print(f'recv: {len(response)}')


def send(sock):
    request = f'GET / HTTP.0\r\nHost: baide.com\r\n\r\n'
    while True:
        try:
            sock.send(request.encode('ascii'))
            print('sent')
            break
        except IOError:
            pass


def recv_chunk(sock):
    while True:
        try:
            chunk = sock.recv(4096)
            return chunk
        except IOError:
            pass


def recv(sock):
    response = b''
    chunk = recv_chunk(sock)
    while chunk:
        response += chunk
        chunk = recv_chunk(sock)
    return response


fetch()
```

大概步骤为：

1. 建立连接
2. 不停的轮询现在可不可以发消息，可以的话就发送
3. 不停的轮询现在可不可以读一个块的消息，可以的读一下
4. 利用本地变量拼接所有的块，返回

`sock.setblocking(False)`把连接改为异步模式，这样`connect`和`recv`都不再阻塞, 和之前的代码相比有几个不同的地方：

1. 异步模式`connect`时候会默认抛出`BlockingIOError`
2. 在异步模式下，不能知道什么时候`connect`连接完成，可以执行`send`，所以这时候要不停的轮询
3. 同样，也不知道什么时候可以`recv`这时候同样需要轮询
4. 在每次`recv`到的代码块也是存在本地变量中

可以发现，这样的异步效率比同步更加低，在上面的同步代码中，可以利用多线程, 每个线程占用一个端口来提高效率，但是在这种轮询模式下，CPU会极其繁忙，加上GIL的限制，多线程的效率会比同步模式,甚至单线程低的多。

### 三、事件驱动-回调

很明显上面的异步不是我们想要的，如果想提高异步效率，可以采用事件驱动，从`不停的问有没有活干`改为`有活干的话通知我一声`，在Unix系统中，一般可以采用`select`(远古时代), `poll`, `epoll`(Linux), `kqueue`(BSD), 这些怎么工作的就是另外一个话题了，这里暂不讨论。

用原生的接口编程还是很困难的，还好在Python里面封装了很好用的接口

```python
import socket
from selectors import DefaultSelector, EVENT_WRITE, EVENT_READ

selector = DefaultSelector()


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

        selector.register(self.sock.fileno(), EVENT_WRITE, self.connected)

    def connected(self, key, mask):
        print('connected')
        selector.unregister(key.fd)
        request = 'GET / HTTP1.1\r\nHost: www.zhihu.com\r\n\r\n'
        self.sock.send(request.encode('ascii'))
        selector.register(key.fd, EVENT_READ, self.read_response)

    def read_response(self, key, mask):
        chunk = self.sock.recv(4096)
        if chunk:
            self.response += chunk
        else:
            selector.unregister(key.fd)
            print(f'recv count: {len(self.response)}')
            self.parse_response()

    def parse_response(self):
        print(f'recv count: {len(self.response)}')



def loop():
    while True:
        events = selector.select()
        for event_key, event_mask in events:
            cb = event_key.data
            cb(event_key, event_mask)


Fetcher().fetch()
loop()

```

梳理一下这个的执行步骤：

1. 显式指明`sock`和`response`保存在`self` 
2. 建立连接，利用三元组注册`可写`事件， (`文件描述符fd`, `类型`, `回调函数`)
3. 连接建立之后，取消注册事件，发送request, 同时注册可读事件
5. 每次有可读事件时候，读取`4k`的信息
6. 读取完成后，取消注册事件

和上面最大的不同在于，无论是`同步请求`还是`轮询异步`, 自始至终的控制流都是在一个函数内，`response`和`sock`保存在本地变量中，也就是`stack`中(在Python中其实是保存在`heap`里面)。

但是在这个回调版本中，控制流大概为`fetch` -> `loop` -> `connected` -> `loop` -> `read_response` -> `loop`...-> `read_response`， 可以看到控制流在不同的函数之间跳转，需要我们手动、显示的保存需要的值给之后的控制流使用。

也就是说，我们要显示的保存当前的`continuation`，等待事件发生的时候继续执行当前函数。这样就可以做到，当当前代码执行完，需要等待io事件发生时候，就移交控制流，让其他代码段有执行的机会，事件发生时候，当前代码继续执行，不用轮询，不用阻塞。

但是这样有什么问题呢？

比如如果，在`parse_response`时候发生问题

```python
def parsed_response(self):
    print(f'recv count: {len(self.response)}')
	raise Exception('parse error')

```

```python
connected
recv count: 335
Traceback (most recent call last):
  File "test.py", line 51, in <module>
    loop()
  File "test.py", line 47, in loop
    cb(event_key, event_mask)
  File "test.py", line 36, in read_response
    raise Exception('parse error')
Exception: parse error
```

这个时候我们完全不知道从哪调用的这个函数，这个函数要往哪去， 我们丢失了`continuation`, 丢失了控制流信息。

那么，该怎么办？

还记得上面的`同步`和`轮询`吗? 这两个版本没有丢失`continuation`, 没有丢失`控制流`，不用显式保存共享状态， 因为所有代码段再次开始执行时候所需要的信息都在一个函数内！！！

那么异步能不能也这样？也就是，在一个函数内，需要的时候停止，事件发生时候再次开始执行？

等等……这不就是`协程`吗？！



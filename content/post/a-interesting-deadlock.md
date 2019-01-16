---
title: "一种有意思的死锁"
date: 2019-01-16T22:43:13+08:00
lastmod: 2019-01-16T22:43:13+08:00
draft: false
keywords: []
description: ""
tags: ["sticker"]
categories: ["sticker"]
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

刚闲逛时候发现有[一篇文章](https://python-gino.readthedocs.io/zh/latest/why.html#the-story)讲了一种有意思的死锁，不是亲身经历，但是也很有意思了。

先会议一下死锁的定义是什么？翻出压箱底的`现代操作系统`:

> 如果一个进程集合中的每个进程都只能等待由该进程集合中的其他进程才能引发的事件，那么，该进程集合就是死锁的。

所以，还是不知道什么是死锁。

举个例子，

> 比如甲乙两人同时去食堂吃饭，食堂里面只有一个碗和一双筷子，甲抢到了碗，乙抢到了筷子，甲等乙放下筷子，乙等甲放下碗，结果两个人都饿死了。

又比如:

> A和B两个进程同时去写数据库，A锁住了第n行去读第m行，B锁住了第m行去读第n行，结果A，B都无限期等下去了。

无论是`甲`,`乙`,还是`A`,`B`,都在等待另一个已经死锁的进程释放资源。因为所有的进程都不能运行，所以资源永远无法被释放，所以所有进程都不能被唤醒。进程的数量以及占有和请求的资源数量都是无关紧要的，而无论资源和何种类型都会发生何种结果，所以这种死锁被称为资源死锁。

好了，抄书结束，划重点。

好吧，上面这段话都是重点。

废话👆

下面看`占有和请求的资源数量都是无关紧要的🌰`

> For example, the first coroutine starts a transaction and updated a row, then the second coroutine tries to update the same row before the first coroutine closes the transaction. The second coroutine will block the whole thread at the non-async update, waiting for the row lock to be released, but the releasing is in the first coroutine which is blocked by the second coroutine. Thus it will block forever.

之前在协程中，每时每刻实际都只有一件事在执行，发生死锁的概率应该不高吧。too young啊

比如第一个协程发起事务更新数据库中的一行，这时候第二个协程试图更新同一行，如果第二个协程是非异步的更新，那么它就要等待第一个协程释放资源，但是这时候整个线程已经被第二个线程阻塞了，第一个协程无法释放资源，deadlock!

回忆一下书本中关于资源死锁的四个条件，回忆一下:

1. 互斥条件
2. 占有和等待条件
3. 不可抢占条件
4. 环路等待条件

这四个条件只要有一个不成立那么资源死锁就不会发生。

好了现在套用一下上面这个例子。

1. 数据库行锁和当前线程执行都是互斥的，✅
2. 第一个协程占有了数据库中的锁等待线程执行权，第二个协程占有了当前协程的执行等待行锁，✅
3. 不可抢占，✅
4. 环路等待，第二个线程等待行锁，第一个线程等待有机会被执行，✅

死锁bingo!

怎么解决呢？

> A simple fix would be to defer the database operations into threads, so that they won't block the main thread, thus won't cause a dead lock easily. It usually works and there is even a library to do so. However when it comes to ORM, things become dirty.

把数据库相关操作扔给其他线程去执行，确保不会阻塞主线程就行了。

但是真的就行了吗？

在一个复杂的项目中，你永远不知道哪一行代码会调用同步代码阻塞主线程，`Racing condition just happens under pressure, and anything that may block will eventually block`, 可能会发生的事情终将要发生。

所以作者建议:

> This is usually the time when I suggest to separate the server into two parts: "normal blocking with ORM" and "asynchronous without ORM".

不过倒是想起另外一个问题了, 现在公司业务都是用的`gunicorn`+`gevent`, 这个内部是怎么解决这个问题的呢，想不明白诶。。。

不过联想到`gevent`的协程是抢占式的，会不会是因为破坏了`不可抢占条件`而避免了死锁呢，给自己留个问题吧。

参考资料：

1. 现代操作系统
2. https://python-gino.readthedocs.io/zh/latest/why.html#the-story
<!--more-->

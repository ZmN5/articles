---
title: "Python Gil"
date: 2020-08-23T22:19:58+08:00
lastmod: 2020-08-23T22:19:58+08:00
draft: false
keywords: []
description: ""
tags: []
categories: ["python"]
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

# 一、 什么是GIL？

GIL的全称为`Global Interpreter Lock`，是CPython中的一种锁实现，目的是为了在同一时间，只允许一个线程控制解释器。

# 二、为什么需要GIL？

Python的内存管理用的是引用计数，也就是每个对象都会有一个计数器来跟踪被引用的数量，如果计数器的值为0，则此对象占用的内存被释放。

也就是这个引用计数的值需要避免竞态条件(race condition)，不能有多个线程同时修改对象引用计数器的值，否则会造成内存泄漏或者各种奇怪的bug。

如果给每个对象都加锁的话，就很可能造成死锁（只有多个锁的情况下才可能造成死锁），同时影响单线程的性能。

所以为了解决线程安全的内存管理这个问题，Guido的选择是，引入一个粗粒度的全局锁。

其他没有GIL的语言(比如Java)，通过不同的GC方式，来解决这个问题，但是往往同时会引入其他技术来提升单进程的性能，比如JIT。

# 三、GIL的影响

GIL的影响主要在多线程情况下，这里首先要明确几点：

1. python的线程都是原生操作系统线程
2. 操作系统负责线程的调度

这里从IO-bound和CPU-bound两种不同的线程分开说。

## 3.1 协作的IO-bound线程

在使用Python标注库的情况下进行IO操作时，比如网络请求、文件读写、数据库读写等，解释器会在IO操作之前释放GIL。

```C
/* s.connect((host, port)) method */
static PyObject *
sock_connect(PySocketSockObject *s, PyObject *addro)
{
    sock_addr_t addrbuf;
    int addrlen;
    int res;

    /* convert (host, port) tuple to C address */
    getsockaddrarg(s, addro, SAS2SA(&addrbuf), &addrlen);

    Py_BEGIN_ALLOW_THREADS // 释放GIL
    /* Others thread may run now */
    /* sleep(timeout) */
    /* write(fd, buffer, size) */
    /* recv(sock, buffer, size, flags) */
    Py_END_ALLOW_THREADS // 获取GIL

    /* error handling and so on .... */
}
```

总结来说：

1. 在进行IO时候释放GIL
2. 其他处于可执行状态的线程获得执行机会

有点协作的意味，所以在IO密集型操作中使用多线程，性能并不会太低。

## 3.2 程序block，无法处理系统Signal

在Python2里面，如果一个线程从不进行任务IO，解释器每执行`100ticks`(1tick对应解释器中的一条指令)会进行一次`check`。

每次执行`check`时候，会进行：

1. 在主线程中处理各种`signal`
2. release & reacquire GIL

可以看到，

1. check的周期不是以时间为单位的
2. 在一个周期内可以阻塞任何事情，执行不同的指令耗时相差很大

```python
# test.py
def block_everything():
    nums = xrange(1000000000)
    -1 in nums

block_everything()
```

执行这个脚本，然后`ctrl+c`，会发现并不能使程序终止。

```python
dis.dis(block_everything)
      0 LOAD_GLOBAL              0 (xrange)
      3 LOAD_CONST               1 (1000000000)
      6 CALL_FUNCTION            1
      9 STORE_FAST               0 (nums)

     12 LOAD_CONST               2 (-1)
     15 LOAD_FAST                0 (nums)
     18 COMPARE_OP               6 (in)
     21 POP_TOP
     22 LOAD_CONST               0 (None)
     25 RETURN_VALUE
```

其中的原因是：

1. 其中一条指令（`COMPARE_OP`）耗时太久，而解释器在执行100ticks之后才会处理signals
2. 主进程被block，无法处理signal

## 3.3 Python2的优先级倒置（Priority Inversion）

先谈谈GIL的实现：

* GIL不仅仅是mutex
* 具体实现（Unix）要么是匿名信号量，要么是Pthreads条件变量
* 锁机制基于信号signal
	* 获取GIL时候，检查锁，如果锁非空闲，休眠然后等待锁
    * 释放GIL时候，释放锁并且发送signal给其他线程

![GIL](/post/static/python-gil/gil-locking.jpg)

由于线程的调度都是操作系统负责的，那么多个线程就会被调度到多核上同时执行，就会有CPU Battle的现象。

![CPU_battle](/post/static/python-gil/CPU-battle.jpg)

就上图详细来说，线程t2被调度到CPU核上开始执行时候，拿不到GIL被block，t1释放GIL时候，通知t2来抢，但是这时候t1会被留在原先的CPU核上继续执行，GIL又被t1抢走了，t2继续block。

t2: “怎么有种被调戏的感觉。。。”

![CPU_battle_summary](/post/static/python-gil/CPU-battle-summary.jpg)

从代码大概是这样：

```c
if (interpreter_lock) {
  /* Give another thread a chance */
    if (PyThreadState_Swap(Null) != tstate) {
      Py_FatalError("Ceval, tstate mix-up.")
    }
    PyThread_release_lock(interpreter_lock);

    /* Other thread may run now.*/ 
    /* there is no code here. */

    PyThread_acquire_lock(interpreter_lock, 1);
}
```

从代码可以看出，这里最主要的问题是，从释放GIL到重新获取GIL，中间没有其他代码，这个过程只需要几纳秒，其他线程很难抢到GIL。

这点在几个都是CPU-bound的线程上的影响就是，多线程比单线程执行更慢。
在同时有CPU-bound和IO-bound的线程上影响就是，优先级倒置（Priority Inversion）。本来应该高优先级的IO-bound线程，会很难拿到GIL去执行。

最主要的问题是，这里掺杂了两个相冲突的目标：

1. 对Python解释器来说，只想在单线程上执行，但是同时又不做任何和线程调度相关的事情。
2. 对操作系统来说，看到有多个线程，那么就调度到多个核上同时工作吧。

## 3.4 Python3的GIL

从python3.2开始，引入了新的GIL。

1. 新的GIL仍旧基于`condition variable`和`signaling`，但是和旧的完全不同，signal会显著减少
2. 新的线程切换是基于时间的（`time-based`）, 不像过去一样，是基于ticks的

新GIL的设计基于几条原则：

1. 是否要进行线程切换取决于一个全局变量。

    ```c
    static volatile int gil_drop_request = 0;
    ```

2. 线程将一直运行，直到全局变量的值变为1
3. 全局变量变为1时候，线程必须释放GIL

来看看这些怎么发生的？

### 3.4.1 如果只有一个线程

那么这个线程将会一直运行。

### 3.4.2 如果有两个线程

假设有两个线程t1和t2，初始状态t1拿到线程。

那么t2将执行cv_wait(gil, TIMEOUT)，等待GIL被释放，这时候将会有两种情况

1. t1主动释放GIL（sleep/io/等），t1给t2发送signal，t2开始执行
2. t2等t1执行完时间片，把全局变量gil_drop_request置为1，然后继续执行cv_wait
3. t1将不再执行，放弃GIL，并且发送signal通知自己已经放弃GIL
4. t1将开始等待一个`gotgil`的signal，表明GIL已经被其他线程获取

（默认的TIMEOUT时间是5ms）

### 3.4.3影响

1. 新的GIL将允许线程在忽略其他线程以及IO优先级的情况下执行5ms
2. 一个CPU密集型线程将阻塞IO线程至少5ms
3. 这将会影响响应时间
4. 长时间执行的c/c++扩展将影响线程切换，线程切换将不再是抢占式的

# 四、GIL不是什么？

1. 在多线程编程中，即便有了GIL，你的代码仍然不一定是线程安全的，仍然需要其他同步原语

# 五、总结

毫无疑问，Python在时下这么流行是离不开GIL的：

1. GIL让单线程执行的更快
2. GIL让IO-bound的多线程执行的更快
3. GIL让C扩展实现的多线程编程执行的更快😂
4. 实现及使用C扩展更加容易

python选择了粗粒度的GIL了保证内存安全，其他语言选择细粒度的锁实现真正的多线程，反过来说，Python也利用GIL也把GC时候的`Stop the world`，分解成为了一个个更细粒度的内存回收，其实换个角度来说，也就是换成了一个个更细粒度的`Stop the world`吧。

参考:

* [What is the Python Global Interpreter Lock (GIL)?](https://realpython.com/python-gil/)
* [Python's Infamous GIL by Larry Hastings](https://www.youtube.com/watch?v=KVKufdTphKs)
* [Inside the Python GIL](http://www.dabeaz.com/python/GIL.pdf)
* [Inside the New GIL](http://www.dabeaz.com/python/NewGIL.pdf)
<!--more-->

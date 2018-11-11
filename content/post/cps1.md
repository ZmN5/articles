---
title: "怎么理解CPS?(1)"
date: 2018-11-11T02:19:24+08:00
lastmod: 2018-11-11T02:19:24+08:00
keywords: ["lisp", "CPS"]
description: ""
tags: ["racket", "continuation", "CPS", "lisp"]
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

这段时间学racket终于遇到了传说中的`CPS`, 不出意外果然很难理解。先来看一下维基百科中对`CPS`的定义:

> In functional programming, continuation-passing style (CPS) is a style of programming in which control is passed explicitly in the form of a continuation. 

我的理解也就是，CPS也就是一种以`continuation`的方式显式的传递`控制流`？那么， 什么是`continuation`?

## continuation

看一下racket文档里面对continuation的定义：

> A continuation is a value that encapsulates a piece of an expression’s evaluation context.

continuation是一种封装了表达式evaluation context的东西，比如：

> (+ 1 (+ 1 (+ 1 0)))

1. 对上面这个表达式中的`0`来说，它的`continuation`就是`(+ 1 (+ 1 (+ 1 _)))`
2. 对`(+ 1 0)`来说，它的`continuation`就是`(+ 1 (+ 1 _))`

...


## 举个例子

比如一个除去list中所有4的函数

```

(define rember4
  (lambda (ls)
    (cond
      [(null? ls) '()]
      [(= 4 (car ls))  (rember4 (cdr ls))]
      [else (cons (car ls)
                  (rember4 (cdr ls)))])))

```

如果用continuation把这个函数改写一下：

```
(define rember4*
  (lambda (lat k)
    (cond
      [(null? lat) (k '())]
      [(= 4 (car lat)) (rember4* (cdr lat)
                                 (lambda (x)
                                   (k x)))]
      [else (rember4* (cdr lat)
                     (lambda (x)
                       (k (cons (car lat) x))))])))
```

在这个函数中传入`rember4`的`k`就是`rember4`函数被递归调用时候的`continuation`， 在这两个函数中

```
(lambda (x)
  (k (cons (car lat) x)))
  
(lambda (x) (k x))
```

都用闭包把当前的evaluation context封装起来了，而且在函数内部设计好了`expression`和`evaluation context`的控制流。


## call/cc

按照上面的写法，用各种匿名函数捕获continuation显然是很麻烦的，所以scheme/racket中就提供了语法糖，`call/cc`也就是`call-with-current-continuation`， 关于`call/cc`:

> Scheme allows the continuation of any expression to be captured with the procedure call/cc. call/cc must be passed a procedure p of one argument. call/cc constructs a concrete representation of the current continuation and passes it to p. The continuation itself is represented by a procedure k. Each time k is applied to a value, it returns the value to the continuation of the call/cc application. This value becomes, in essence, the value of the application of call/cc.

也就是说， call/cc会捕获当前的continuation传给自己的单参数`p`， `p`接收`continuation`也就是`k`作为参数，`k`作用于一个`value`上面，那么那个`value`也就是`call/cc`的值， 那么如果不使用`k`呢，直接看书里面的一个例子：

```
(call/cc
  (lambda (k)
    (* 5 4)))  >> 20 

(call/cc
  (lambda (k)
    (* 5 (k 4))))  >> 4 

(+ 2
   (call/cc
     (lambda (k)
       (* 5 (k 4))))) >>  6
```

1. 不适用`k`的话， 那么`p` 的结果也就是`call/cc`的值
2. 使用`k`的话，在上面例子中， `call/cc`的值就是4

是不是没想象中的难，哈哈哈。。。

好了，再来看三个例子，

## 再来三个例子

```
((call/cc (lambda (cont) cont)) (lambda (x) "hi"))
```

1. `call/cc`捕获的这个`continuation`可以描述为：把一个`value`应用于 `(lambda (x) "hi")`， 大概可以写成这样`(_ (lambda (x) "hi"))`。
2. `(lambda (cont) cont)`返回cont，所以`call/cc`会返回continuation自己，也就是`(_ (lambda (x) "hi"))`， 这时候的`value`就是`(lambda (x) "hi")`
3. 所以最后得到`((lambda (x) "hi") (lambda x "hi"))`, 结果为`hi`


```
(let ([x (call/cc (lambda (k) k))])
  (x (lambda (ignore) "hi"))) >> "hi"
```

1. `call/cc`捕获的这个`continuation`可以描述为: 把一个value绑定到x，之后把x应用于`(lambda (ignore) "hi")`, 大概可以写为：`(let [x _] (x p))`, `p`即为`(lambda (ignore) "hi")`
2. `(call/cc (lambda (k) k))`返回k , 也就是上面的这个continuation
3. 最后可以写为`(let ([x (lambda (x) x)]) (x (lambda (ignore) "hi")))`

最后一个：

```
(((call/cc (lambda (k) k)) (lambda (x) x)) "HEY!") >> "HEY!"
```

到这个例子时候我已经疯了。。。

从上面三个例子可以看到，`continuation`复制的还有当前环境下的控制逻辑, 而不仅仅是数据！！！

还有其他有意思的例子，比如`阴阳谜题`之类的，在这里就不放了。。。

利用`CPS`可以实现`break`, `coroutine`, `generator`等， 至于怎么实现的，我还没学会。。。
加油！

---

参考资料：

1. [wiki](https://en.wikipedia.org/wiki/Continuation-passing_style#Continuations_as_objects)
2. [racket docs](https://docs.racket-lang.org/guide/conts.html)
3. [The Scheme Programming Language](https://www.scheme.com/tspl4/further.html#./further:h3)
4. [continuation](https://cgi.soic.indiana.edu/~c311/lib/exe/fetch.php?media=cps-notes.scm)

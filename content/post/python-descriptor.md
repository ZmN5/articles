---
title: "Python描述符协议总结"
date: 2019-01-20T19:55:58+08:00
lastmod: 2019-01-20T19:55:58+08:00
draft: false
keywords: ["descriptor", "python"]
description: ""
tags: ["python"]
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

本文主要翻译自[Descriptor HowTo Guide](https://docs.python.org/3.6/howto/descriptor.html#functions-and-methods), 另外加了一下自己的理解。

## 描述符协议是什么？

如果一个类实现了`__set__`, `__get__`, `__delete__`，这三个中任意一个或者多个方法，那么就实现了描述符协议，它的实例这就是一个描述符，同时要明白这也是一个类属性。具体描述符协议如下：

```python
descr.__get__(self, obj, type=None) --> value

descr.__set__(self, obj, value) --> None

descr.__delete__(self, obj) --> None
```

在Python中，对象属性的默认查找顺序，举例来说是这样的，比如`a.x`

1. 先查找`a.__dict__['x']`
2. 如果没有查到的话, `type(a).__dict__['x']`沿着`方法解析顺序`(MRO) 继续向上查找，除了元类

如果要查找的属性是一个实现了描述符协议的对象的话，python也许会用描述符方法覆盖上面的这种默认的查找行为。这种调用优先顺序依赖于哪种描述符方法被定义了。

另外，如果同时实现了`__set__`, `__get__`，被称为`数据描述符`(`data descriptor`), 如果只实现了`__get__`方法，那么被称为`非数据描述符`? why? 因为非数据描述符一般用于`方法`(当然也可以用于其他情况)

`数据描述符`和`非数据描述符`的最主要区别在于，这两个是怎么覆盖对实例字典的访问（毕竟Python是一门构建字典之上的语言嘛）。如果实例的字典有一个字段和描述符有同样的名字，在这种情况下，数据描述符和非数据描述符会有不同的表现, 对数据描述符来说，数据描述符会被优先访问，对非数据描述符来说，实例字典会被优先访问。这种情况造成了，如果想使用描述符使一个属性变为`只读的`， 那么就只能使用数据描述符，并且在`__set__`方法中`raise AttributeError`

举例来说:

```python
class NotDataDescriptor:
    def __get__(self, instance, owner):
        print('In descriptor')


class Test:
    a = NotDataDescriptor()

t = Test()
t.a   # (1) print ->  In descriptor
print(t.__dict__) # (2) print -> {}

t.a = 'a'
print(t.a) #  (3) print -> a
print(t.__dict__) # print -> {'a': 'a'} 

```

在上面的例子中，`a`是一个非数据描述符, 所以在当`t.a`时候，触发`__get__`, 打印出了`In descriptor`，查看这时候的`__dict__`为空。

当要给`t.a='a'`，给`t.a`赋值时候，这时候有两种选择，覆盖实例字典`t.__dict__['a'] = 'a'`, 或者覆盖描述符属性`type(t).__dict__['a'].__set__('a', 'a')`, 因为`a`是一个非数据描述符，没有`t.a.__set__`方法，所以只能使用第一种。当再次读取`t.a`时候，因为`a`是一个非数据描述符，所以覆盖实例字典的优先级高于非数据描述符, 触发第一种操作， 打印出`a`，这时候查看`t.__dict__`会发现，新增的值确实在实例字典中了。

如果同样的例子改写为数据描述符呢?

```python
class DataDescriptor:
    def __init__(self):
        self._data = None

    def __get__(self, instance, owner):
        print('In descriptor')
        return self._data

    def __set__(self, instance, value):
        self._data = value * 2


class Test:
    a = DataDescriptor()


t = Test() # print -> In descriptor
t.a
print(t.__dict__) # print -> {}

t.a = 'a'
print(t.a) # print -> In descriptor\naa
print(t.__dict__)  # print {}
```

## 描述符调用规则

对`obj.d`的调用取决于`obj`是一个`对象object`还是一个`类class`

对象来说，比如对`b.x`的访问来说，`object.__getattribute__()`会把对`b.x`的调用转变为`type(b).__dict__['x'].__get__(b, type(b))`, 这种方法的实现依赖于一种调用链：

1. 数据描述符的调用优先于实例变量
2. 实例变量的调用优先于非数据描述符
3. 最低的优先权为`__getattr__`

对类`class`来说，`type.__getattribute__()`, 把`B.x`的调用转变为`B.__dict__['x'].__get__(None, B)`

Python代码描述如下：

```python
def __getattribute__(self, key):
    "Emulate type_getattro() in Objects/typeobject.c"
    v = object.__getattribute__(self, key)
    if hasattr(v, '__get__'):
        return v.__get__(None, self)
    return v
```

重要的几点在于：

1. 描述符是被`__getattribute__`调用的
2. 覆写`__getattribute__`会阻止自动调用描述符
3. `object.__getattribute__()`和`type.__getattribute__()`对`__get__`的调用是不同的
4. 数据描述符会一直覆盖实例字典
5. 非数据描述符会一直被实例字典覆盖

在上面两个栗子中，我如果自定义`__getattribute__`会怎么样：

```python
class DataDescriptor:
    def __init__(self):
        self._data = None

    def __get__(self, instance, owner):
        print('In descriptor')
        return self._data

    def __set__(self, instance, value):
        self._data = value * 2


class Test:
    a = DataDescriptor()

    def __getattribute__(self, attr):
        """
        丧心病狂一点，直接返回None吧
        """
        return None


t = Test()
t.a # 没有print，因为__get__没有被调用 
print(t.__dict__) # print -> None

t.a = 'a'
print(t.a) # print -> None
print(t.__dict__) # print -> None

```

或者简单改写一下调用优先顺序:

```python
class NotDataDescriptor:
    def __get__(self, instance, owner):
        print('In descriptor')


class Test:
    a = NotDataDescriptor()

    def __init__(self):
        self.test = 'test'

    def __getattribute__(self, attr):
        """
        对非数据描述符由优先调用实例字典改为略过实例字典不调用
        """
        value = type.__getattribute__(type(self), attr)
        if hasattr(value, '__get__'):
            return value.__get__(None, self)
        return value


t = Test()
t.a # print -> In descriptor

t.a = 'a'
t.a # print -> In descriptor
```

## 常用的描述符

之前只知道`property`， `classmethod`, `staticmethod`是用描述符实现的，但是没想到`方法method`也是用描述符实现的，不过也正好解答长久以来的`self`是怎么注入方法里面的疑问。当然在Python里面这些都是用c实现的，Python版本参考本文开始时候介绍的那篇文章。

## 参考资料

1. [Descriptor HowTo Guide](https://docs.python.org/3.6/howto/descriptor.html#functions-and-methods)
2. Python学习手册

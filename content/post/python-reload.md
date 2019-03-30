---
title: "Python Reload"
date: 2019-03-28T22:11:54+08:00
lastmod: 2019-03-28T22:11:54+08:00
draft: false
keywords: ["python", "reload"]
description: ""
tags: ["python"]
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
  enable: false
  options: ""

sequenceDiagrams: 
  enable: false
  options: ""

---

和算法同学的合作，一般都是调用算法同学写的`package`, 然后开始胡搞

但是现在有一个需求，要动态加载算法同学的代码，为什么会有这么奇怪的需求，因为是甲方爸爸提出的啊。

## 一、 第一个思路当然就是`reload`了

算法同学之前一直把Python代码打包成动态链接库``.so``文件。能不能`reload` so文件呢，过程就不放了，答案就是完全行不通。

## 二、zipimport

Python是可以从zip包里面导入代码的，那能不能用这个呢。

```python
import zipimport
import zipfile

pyfile = 'zip_test.py'
zfile = 'zip_test.zip'


def zip_file() -> None:
    with zipfile.ZipFile(zfile, 'w') as zf:
        zf.writestr(pyfile, code)

# --- step1 ---


code = '''
def func():
    return 1
'''

zip_file()

importer = zipimport.zipimporter('zip_test.zip')

module = importer.load_module('zip_test')
print(module.func())


# --- step2 ---

code = '''
def func():
    return 2
'''

zip_file()

importer = zipimport.zipimporter('zip_test.zip')

module = importer.load_module('zip_test')
print(module.func())

```

打印的结果为: 

```
1
2
```

说明是可以做到的，但是在实验的过程中有一个致命的确定，如果用`zip`打包不统一，比如用的系统打包工具，偶尔会出现导出`zip`错误，造成整个服务只有重启才行。既然是打包的问题，那就换一种打包方式，换成Python代码打包方式，打包成`egg`是否就解决了这个问题呢。

## 三、egg + reload

首先我们copy from stack overflow，复制一个打包egg的脚本:

https://stackoverflow.com/questions/47286690/how-do-i-create-and-load-an-egg-file-in-python

然后写一个简单的被导入的包

```
# ├── egg_test_raw
# │   ├── __init__.py
# │   └── func.py

# __init__.py

from importlib import reload

from . import func

reload(func)

# func.py

def test_func():
    return 1
```

我们会把这个包打包两次，只是返回值不同，分别命名为`egg_test_v1.egg`和`egg_test_v2.egg`

然后:

```python

# main.py

import sys
from importlib import reload
import shutil

shutil.copyfile('egg_test_v1.egg', 'egg_test.egg')

sys.path.append('egg_test.egg')

#  --- step 1 ---

import egg_test

from egg_test.func import test_func

print(test_func())


# --- step 2 ---

shutil.copyfile('egg_test_v2.egg', 'egg_test.egg')

reload(egg_test)

from egg_test.func import test_func

print(test_func())
```

看到结果会输出

```
1
2
```

是不是大功告成了呢？？
？？？
？？？

！！！！！！当然不是

## 四、尽量不用reload

我以为可以交作业时候，去问了一下教授会不会有什么坑，结果教授说

> 尽量不要用reload

不过，为什么呢？教授说reload之后，需要手动清理之前的所有已经`load`的对象。

什么？ `reload`之后，之前的对象还在吗？？？？

做个试验，把上面的代码复制一份，稍微修改一下:

```python
import sys
from importlib import reload
import shutil

shutil.copyfile('egg_test_v1.egg', 'egg_test.egg')

sys.path.append('egg_test.egg')

#  --- step 1 ---

import egg_test

from egg_test.func import test_func



# --- step 2 ---

shutil.copyfile('egg_test_v2.egg', 'egg_test.egg')


reload(egg_test)

from egg_test.func import test_func as func2

assert test_fun is func2
```

输出：

```
Traceback (most recent call last):
  File "main.py", line 26, in <module>
    assert test_func is func2
AssertionError
```

之前的对象确实仍然存在，并没有被回收！！！

而且通常算法同学的对象还都特别大，细思极恐！！！

但是要手动清理之前的对象吗，别开玩笑！！！

所以，该怎么办？

首先，不用`reload`

## 五、新起进程

我们需要的是可以reload代码，同时清理之前的对象，同时这些是异步任务，那么就可以把整个服务重启嘛！！

但是服务在甲方的集群里面部署，并没有那么高的权限。

但是在Python里面，import代码的机制是，在每个进程里面import一遍代码。

所以新开一个进程跑任务就行了嘛

还有一个问题，zipimport的对象会被回收么？


### 代码链接：

https://github.com/fucangyu/blog-code/tree/master/python_reload

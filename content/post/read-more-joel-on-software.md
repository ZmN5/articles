---
title: "软件随想录"
date: 2020-02-17T21:54:25+08:00
lastmod: 2020-02-17T21:54:25+08:00
draft: false
keywords: []
description: ""
tags: []
categories: ["reading"]
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

<!--more-->

> 作者Joel Spolsky，非常好用的两个产品`StackOverflow`和`Trello`的创建者，作为老牌程序员，创业者，他对行业的描述毫无疑问是很值得让人思考的。

> 这本书是前公司师傅大鹏推荐的，感谢。

> 本文内容是书中内容以及自己的一点想法。

## # 程序员

### 1. 我们为什么要学习算法、编译原理、计算机网络等工作中几乎永远也无法用上的东西？

* 训练思维，锻炼记忆力，培养逻辑能力。不得不说，一直只在增删改查这样的业务中的摸爬滚打的话，技术/逻辑思维方面几乎没有任务进益，最后只能跟刚毕业的年轻人比体力，然后完败35岁失业。
* 这些东西并不是用不上，用不上的原因大概率是不知道，不了解，不懂，然后碰到了思维的边界，工作中遇到的问题只知道在增删改查中找答案。
* 现在业界的技术层出不穷，仅仅前端来说就有vue/react/angular，hadoop没火多久就有spark/flink，docker/kubernetes等容器化技术眼花缭乱，go/kotlin/scala/rust永远学不完，也就是说，如果跟这些新技术的话，毫无疑问凭借自己的精力，哪怕艰难跟上也很难长久。那怎么办，总不能失业啊。但是换个角度来说，计算机科学也就是证明（递归），算法（递归），语言（lambda演算），操作系统（指针），编译器（lambda演算）组成的，MapReduce的灵感总归还是来自于函数式编程，go/rust/scala终归也就是在c/lisp/haskell的圈子里面，容器化技术的爆发也是建立在操作系统已经建立namespace/cgroup和其他容器化理论的基础上，所以，还是学基础，追前沿的东西也比较容易。

### 2. 锻炼表达，勤于写作

> 能不能清晰地写出技术内容的文章决定了你是一个口齿不清的程序员还是一个领袖。

根据我浅薄的两年半职业生涯的观察，作为程序员，日常写代码的时间很难超过50%，其他大部分时间都是在沟通，怎么清晰的表达自己的想法，怎么快速捕获同事的意思，真的是一门艺术啊。

除了写漂亮的代码表达自己的意思，优秀的文档表达能力绝对是优秀程序员的标准之一。

### 3. 作为程序员，多写代码

作为程序员，不用解释

### 4. 学好微观经济学

> 你一定要去学微观经济学，你必须搞懂供给和需求，你必须明白竞争优势，你必须理解深恶是理解净现值（NPV），什么是边际效用，只有这样，你才会懂得为什么生意是现在这种做法。

除了作者说的这些，我觉得对自己更重要的是，避免让自己的思维局限在编程里面，多学一点东西，毕竟很难保证明天自己还会不会写代码。

### 5. 程序效率没那么重要

程序的执行效率不是不重要，而是，重要性没那么高，最起码排在下面几点之后：

* 功能

    > 从长远来看，哪些不关心效率、不关心程序是否臃肿，一个劲往软件中加入高级功能的程序员终将拥有更好的产品

* 代码可读性

    可读性也就意味着可维护性，也就意味着软件的寿命。就像SICP中说的，代码是用来给人读的，只是恰好可以被用来执行。无论多么好的文档，精确性、时效性和完整性也远远无法和代码相比。

* 整洁

## # 编码

* 最有生产效率的编程环境是那些允许你在不同层次上进行抽象的编程环境
* 让错误的代码能够一眼被看出来
* 让错误的代码能够被一眼看出，这种做法有一个前提，即正确的东西在显示屏上必须紧挨在一块 

    * 尽量将函数写的简短
    * 变量声明的位置距离使用位置越近越好
    * 不要使用宏（macro）去创建你自己的编程语言
    * 不要让右括号与对应的左括号的距离超过一个屏幕

## # 设计/产品

* 如果你问人们喜欢什么样的风格和设计，除非这些人受过专门的训练，否则他们会选择他们最喜欢的品种。
* 每天前进一小步，将一件东西做的比昨天好一点点。
* 别给用户太多选择，太多选择最终限制了我们的自由。

    感觉这句话可以做量方面思考，一方面每个选择需要思考，所有的选择都需要成本，而人类是懒惰的。另外，选择一般都需要建立在了解、认识甚至专业的基础上，而让用户在所有方面都了解/专业明显就是异想天开。


## # 管理

### 1. 用`认同法`管理一个公司和团队

相于`军事管理法`，`经济利益驱动法`来说作者Joel更加偏向于`认同法`。

军事管理法，不招人喜欢，更重要的原因是，编程不像军队，每个人都有不同的工作，不同的方向，不同的节奏，另外，程序员接受的信息比leader更多，更适合做决策。

经济利益驱动法也有两个最大的问题:

1. 它会把内部激励变成外部激励。
2. 人们有追求局部利益最大化的倾向。在这种规则下，也就是在鼓励人们和制度博弈。绩效考核都会经历两个阶段：

    * 得到绩效考核自己想得到的
    * 所有人都会把考核的指标给最大化。用之前听过的一句话总结就是，`考核什么，就会得到什么`。考核代码行数，那么就会得到恐怖的代码行数增量，考核bug数，那么就没人写代码，也就没有了bug。

所以只剩下了`认同法`，不同于经济利益驱动法，认同法是为了强化内部激励。如果团队中的员工都认同团队的目标，文化，认同自己是在和很牛的人一块工作，以及其他听起来比较虚的东西，总会形成良性循环。

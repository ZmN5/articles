---
title: "Django orm改字段名造成数据丢失"
date: 2018-11-10T21:34:22+08:00
lastmod: 2018-11-10T21:34:22+08:00
keywords: []
description: ""
tags: ["django"]
categories: ["python", "django"]
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

### 一. 现场：
以为是修改一个变量名，但在django orm内部并不这么认为：

model:

```python
旧列名：
cos_path = models.CharField(max_length=200, null=True, verbose_name='存储路径')

新列名:
storage_path = models.CharField(max_length=200, null=True,verbose_name='存储路径')
```

migrations:

```python
operations = [
    migrations.RemoveField(
        model_name='insurancedocument',
        name='cos_path',
    ),
    migrations.AddField(
        model_name='insurancedocument',
        name='storage_path',
       field=models.CharField(max_length=200, null=True, verbose_name='存储路径'),
    ),
```

sql:

```sql
BEGIN;
--
-- Remove field cos_path from insurancedocument
--
ALTER TABLE `apis_insurancedocument` DROP COLUMN `cos_path`;
--
-- Add field storage_path to insurancedocument
--
ALTER TABLE `apis_insurancedocument` ADD COLUMN `storage_path` varchar(200) NULL;
COMMIT;
```

所以在Django orm里面认为这是删去了一个字段，然后又新增了一个字段，然后数据就丢了。

### 二. 其他django改列名手段

#### 1. 利用db_column 改名

操作如下，利用db_column来改变字段名的，然后：


```python
cos_path = models.CharField(max_length=200, null=True, verbose_name='存储路径')

storage_path = models.CharField(max_length=200, null=True, db_column='cos_path', verbose_name='存储路径')
```


```sql

BEGIN;
--
-- Remove field cos_path from insurancedocument
--
ALTER TABLE `apis_insurancedocument` DROP COLUMN `cos_path`;
--
-- Add field storage_path to insurancedocument
--
ALTER TABLE `apis_insurancedocument` ADD COLUMN `cos_path` varchar(200) NULL;
COMMIT;

```

django orm会先把cos_path那个字段给drop掉，数据还会丢失。


#### 2. 利用rename field改名

(1) 利用python manage.py makemigrations --empty apis创建一个新的migrations文件, 编辑：

```python
from django.db import migrations
from django.db import models


class Migration(migrations.Migration):

    dependencies = [
        ('apis', '0006_auto_20180724_1039'),
    ]

    operations = [
        migrations.AlterField(
            model_name='insurancedocument',
            name='cos_path',
            field=models.CharField(
                max_length=200, null=True, verbose_name='存储路径'
            )
        ),
        migrations.RenameField(
            model_name='insurancedocument',
            old_name='cos_path',
            new_name='storage_path',
        ),
    ]

```

(2)执行的sql语句：

```sql
BEGIN;
--
-- Alter field cos_path on insurancedocument
--
--
-- Rename field cos_path on insurancedocument to storage_path
--
ALTER TABLE `apis_insurancedocument` CHANGE `cos_path` `storage_path` varchar(200) NULL;
COMMIT;
```
这时候django才认为执行的是改名操作， 查看migrate之后的数据库发现数据没有丢失。

但是这种操作对mysql来说貌似都是重建一张新表，然后把旧表内容插入新表里面，耗时长, 需要停服 而且危险， 所以补充毅总说的一种方法：

#### 3 不停服改名

1. 先加列，改代码来读老列&&双写，
2. 通过 sql 或者代码复制原列数据到新列，
3. 代码再切换成读写新列，
4. 再做清理老列的事情

#### 4 还是不要改名字了


## 反思:
---

1. 在线上执行这种需要Schema Operations的事情，确认是否必要
2. 还是不要用django migration了
3. 执行sqlmigrate， 确认django orm到底会干什么事情
4. 本地实验，证明操作无害


<!--more-->

---
title: "Python列表"
date: 2023-02-08T20:12:02+08:00
lastmod: 2023-02-08T20:12:02+08:00
author: ["熊大如如"]
keywords:
  - "python"
tags: # 标签
  - "python"
description: ""
weight:
slug: ""
summary: "python列表的常用方法"
draft: false # 是否为草稿
mermaid: true #是否开启mermaid
showToc: false # 显示目录
TocOpen: false # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示路径
cover:
  image: "https://cdn.jsdelivr.net/gh/xxrBear/image/icons8-python-500.png"
---

## 概述

Python 内置的数据结构，可变的、可迭代的对象。广泛应用在 Python 程序中，这么说吧，如果你开始学习 Python 语言，那么列表是每天都会遇到的

## 常用方法

- 添加数据

```python

>>> name_list = []
>>> name_list.append(1)
>>> name_list
[1]
```

- 指定索引位置增加数据

```python
>>> name_list.insert(0, 'bob')
>>> name_list
['bob', 1]
```

- 删除列表最后一个数据

```python
>>> name = name_list.pop()
>>> name
1
```

- 删除列表第一个数据

```python
>>> name = name_list.pop(0)
```

- 扩展列表
```python
>>> name_list.extend([4, 5])
```
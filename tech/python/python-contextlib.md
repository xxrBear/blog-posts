---
title: "Python Context"
date: 2025-09-16T10:55:14+08:00
lastmod: 2025-09-16T10:55:14+08:00
author: 熊大如如
tags: # 标签
  - "python"
description: ""
weight:
slug: ""
summary: "Python 上下文管理知识点梳理"
draft: false # 是否为草稿
mermaid: true #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示路径
cover:
    image: "https://cdn.jsdelivr.net/gh/xxrBear/image/icons8-python-500.png"  # 文章的图片
---

## 一、基础概念
+ 上下文管理器：定义了代码块进入和退出时应执行的逻辑
+ 典型用途：`with` 语句，保证资源正确释放，即使出现异常也能安全退出

例子：

```python
with open("data.txt", "r") as f:
    content = f.read()

# 自动调用 f.__enter__() 和 f.__exit__()
```

执行过程：

1. 调用对象的 `__enter__` 返回值赋给 `f`
2. 执行代码块
3. 执行对象的 `__exit__`无论是否发生异常

## 二、实现

### 类实现
```python
class MyContext:
    def __enter__(self):
        print("进入上下文")
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        print("退出上下文")
        if exc_type:
            print(f"发生异常: {exc_val}")
        return True  # True 表示异常被处理，不会再往外抛
```

### 异常处理
在 Python 中，`with` 语句依赖上下文管理器，对象需要实现 `__enter__` 和 `__exit__` 方法  
其中异常处理核心在 `__exit__` 方法：

```python
class MyContext:
    def __enter__(self):
        print("进入上下文")
        return self
    
    def __exit__(self, exc_type, exc_value, traceback):
        print("退出上下文")
        print("异常类型:", exc_type)
        print("异常值:", exc_value)
        print("追踪:", traceback)
        return False  # 返回 True 表示抑制异常；False 表示继续抛出
```

`__exit__` 的三个参数

如果 `with` 块内发生异常：

+ `exc_type`: 异常的类型
+ `exc_value`: 异常实例
+ `traceback`: 异常的堆栈信息

如果没有异常发生，三个参数都为 `None`

**异常是否抑制**

`__exit__` 的返回值决定异常是否继续传播

+ `True` → 表示异常被捕获并吞掉，不会继续往外抛
+ `False` 或 `None` → 异常会继续向外传播

例子：

```python
class SuppressException:
    def __enter__(self):
        return self
    def __exit__(self, exc_type, exc_value, tb):
        return True  # 吞掉异常

with SuppressException():
    raise ValueError("错误!")  # 不会报错，异常被抑制

print("程序继续运行")
```

## 三、常用工具

### 管理多个上下文对象
#### 用途
+ 动态管理多个上下文（文件、锁、连接…）
+ 注册清理回调（退出时自动执行）

#### 核心方法
+ `enter_context(cm)`：进入一个上下文，自动负责退出
+ `callback(func, *args)`：注册一个退出时要执行的函数

#### 使用示例
+ 动态打开多个文件

```python
from contextlib import ExitStack

with ExitStack() as stack:
    files = [stack.enter_context(open(name, "w"))
             for name in ("a.txt", "b.txt", "c.txt")]
    # 在这里可以安全使用 files
# 自动关闭所有文件
```

+ 注册清理函数

```python
from contextlib import ExitStack

with ExitStack() as stack:
    stack.callback(print, "清理任务1")
    stack.callback(print, "清理任务2")
    print("执行中")
# 退出时会先执行 "清理任务2"，再执行 "清理任务1"
```

#### 特点
+ 清理顺序：后进先出
+ 异常发生时，仍会执行所有清理操作
+ 返回 `True` 的回调/上下文可以吞掉异常

### 忽略指定异常
#### 作用
`contextlib.suppress` 是一个上下文管理器，用来屏蔽指定异常在 `with` 块中如果抛出指定的异常，就会被吞掉，不会继续传播

#### 基本用法
```python
from contextlib import suppress

with suppress(FileNotFoundError):
    open("not_exist.txt")  # 文件不存在 → 不会报错
print("继续执行")
```

#### 可以同时抑制多个异常
```python
with suppress(FileNotFoundError, ZeroDivisionError):
    x = 1 / 0       # ZeroDivisionError 被吞掉
    open("abc.txt") # FileNotFoundError 也被吞掉
```

#### 特点
+ 只会抑制指定类型的异常
+ 其他异常依然会抛出
+ 实际上就是一个简化版的 `try/except`

等价于：

```python
try:
    risky_operation()
except (FileNotFoundError, ZeroDivisionError):
    pass
```

#### 常见场景
+ 忽略文件不存在，删除临时文件时
+ 忽略某些无关紧要的错误，网络超时、权限不足…

```python
import os
from contextlib import suppress

with suppress(FileNotFoundError):
    os.remove("temp.txt")  # 文件不存在时忽略
```

### 重定向
#### 作用
+ 临时重定向输出（stdout 或 stderr）到其他文件或对象
+ 常用于：
    - 捕获 print 输出
    - 将错误信息重定向到文件
    - 日志测试或屏蔽输出

#### 基本用法
```python
import sys
from contextlib import redirect_stdout

with open("output.txt", "w") as f:
    with redirect_stdout(f):
        print("这行会写入 output.txt，而不是屏幕")
print("这行还是打印到屏幕")
```

+ `redirect_stdout(f)`：把 `print` 输出写到 `f`
+ `redirect_stderr(f)`：把 `sys.stderr.write` 输出写到 `f`

#### 捕获到 `io.StringIO`
可以捕获到内存对象，用于测试或进一步处理：

```python
import io
from contextlib import redirect_stdout

f = io.StringIO()
with redirect_stdout(f):
    print("捕获这行文字")
output = f.getvalue()
print("捕获内容:", output)
```

#### 特点
1. 只在 `with` 块内生效
2. 可以嵌套或同时重定向 stdout 和 stderr
3. 对异常也有效（异常打印会重定向到新的 stderr）

#### 同时重定向 stdout 和 stderr
```python
from contextlib import redirect_stdout, redirect_stderr

with open("log.txt", "w") as f:
    with redirect_stdout(f), redirect_stderr(f):
        print("普通输出")
        raise Exception("错误信息")  # 也会写入 log.txt
```

### 啥也不做上下文
#### 简介
+ `nullcontext`提供一个 什么都不做的上下文管理器
+ 常用于条件性使用 `with`，或者在 API 需要上下文管理器时使用
+ 简单来说，就是一个占位符 `with`

#### 基本用法
```python
from contextlib import nullcontext

with nullcontext():
    print("就像普通代码块，没有上下文效果")
```

+ 这个上下文不会做任何操作，也不会抛异常

#### 作为占位符
+ 场景：有时函数需要上下文管理器，但在某些情况下不需要

```python
from contextlib import nullcontext

use_file = False
ctx = open("file.txt") if use_file else nullcontext()

with ctx as f:
    if f:
        f.write("内容")  # use_file=False 时 f=None，不执行写入
```

+ 可以避免写重复的 `if/else with` 逻辑
+ 这在某些场景下很有用，能优雅的处理 `file`不存在的情况

#### 带有返回值
`nullcontext` 可以返回一个对象：

```python
with nullcontext("hello") as x:
    print(x)  # 输出: hello
```

+ 相当于 `__enter__` 返回指定对象，`__exit__` 什么也不做

## 四、经典应用场景
+ 文件操作：`with open()`
+ 线程/进程锁：`with threading.Lock()`
+ 数据库事务：`with transaction:`
+ 临时环境变量：`with patch.dict(os.environ, {...})`
+ 重定向输出 / 日志捕获
+ 测试代码中 patch：`with unittest.mock.patch()`

## 五、实战案例
### 数据库事务
```python
class Transaction:
    def __init__(self, conn): self.conn = conn
    def __enter__(self): self.conn.begin(); return self.conn
    def __exit__(self, exc_type, exc, tb):
        if exc_type:
            self.conn.rollback()
        else:
            self.conn.commit()
```

### 锁管理
```python
import threading

lock = threading.Lock()

with lock:
    # 临界区
    ...
```

### 资源切换
```python
from contextlib import contextmanager

@contextmanager
def set_attr(obj, name, value):
    old = getattr(obj, name)
    setattr(obj, name, value)
    try:
        yield
    finally:
        setattr(obj, name, old)
```

---
title: "Python GIL 的前世今生"
date: 2025-12-14T22:27:22+08:00
lastmod: 2025-12-14T22:27:22+08:00
author: 熊大如如
tags: # 标签
  - "python"
description: ""
weight:
slug: ""
summary: "Python GIL 的前世今生"
draft: false # 是否为草稿
mermaid: true #是否开启mermaid
showToc: false # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示路径
cover:
    image: ""  # 文章的图片
---

## 概述
随着 Python 3.14 的发布，CPython 开始提供实验性的自由线程构建，用户无需再手动编译源码以关闭 GIL。这意味着社区会推出两套完全不同的 Python 构建模式，用以同时支持 GIL 和自由线程。

虽然自由线程版本要完全取代传统 GIL 版本可能还需要许多年的时间，但这标志着 Python 中最令人诟病的 GIL 有望逐步走入历史。然而，很多人可能还不太了解：GIL 是如何产生的？为什么存在？它又解决了哪些问题？本文将结合本人收集到的的多种资料并加入博主本人的理解带你系统了解 GIL 的前世今生。

要想系统性的了解 GIL，我们需要先了解一些 CPython 的内存管理机制以及多线程下的资源争用问题。

## CPython 的内存管理
CPython 的内存管理主要依赖引用计数，辅以垃圾回收和小整数唯一化等机制，其中引用计数是核心处理逻辑。

所谓引用计数，就是在 CPython 的实现中，每个 Python 对象的结构体中都会有一个变量，用于记录这个对象被引用的次数。每当对象被引用一次，计数加 1；每当引用减少一次，计数减 1。当计数减到 0 时，CPython 会销毁该对象，从而完成内存回收。

如果你的 C 语言还不错的话，可以点击这里查看 CPython [_object](https://github.com/python/cpython/blob/v3.14.2/Include/object.h) 的实现源码。变量 `ob_refcount` 就用来记录对象的引用次数。

举个例子，如果一个对象被两个变量引用，那么它的 `ob_refcount` 就是 2。在 Python 层面，我们可以用以下示例验证：

```python
>>> import sys
>>> a = []
>>> b = a
>>> sys.getrefcount(a)
3
```

解释一下这段代码：

+ `a` 创建了一个空列表
+ `b` 引用了同一个列表
+ `sys.getrefcount(a)` 返回对象的引用次数

你可能会疑惑，为什么结果是 3 而不是 2？这是因为在调用 `getrefcount` 时，参数 `a` 作为函数参数会产生一次临时引用，所以返回值比预期多 1

如果你对这个函数感兴趣，可以点击这里  [sys.getrefcount](https://docs.python.org/3.14/library/sys.html#sys.getrefcount) 查看文档内容，文档中明确说明了这一现象。

> 返回 object 的引用计数。返回的计数通常比预期的多一，因为它包括了作为 [getrefcount()](https://docs.python.org/zh-cn/3.14/library/sys.html#sys.getrefcount) 参数的这一次（临时）引用。
>

## 多线程的数据竞争问题
上一节我们介绍了 CPython 的内存管理机制主要依赖引用计数。每次对象被引用，计数加 1；引用减少时计数减 1。然而，在多线程环境下，用户代码对共享变量的操作可能会出现数据竞争，导致结果不可预期。下面通过一个示例来说明这一点：

```python
import threading

a = 0


def count():
    global a
    for _ in range(1000000):
        a += 1


t1 = threading.Thread(target=count)
t2 = threading.Thread(target=count)
t1.start()
t2.start()
t1.join()
t2.join()

print(a)
```

当你使用 Python3.9 及以下版本运行这段程序的时候，你会发现程序并不会总是等于 2000000，但是 3.9 以上的版本却总是等于 2000000，但这并不是说明 3.9 以上的 Python 不存在数据竞争的问题，在任何 CPython 版本中，该结果都没有官方背书，具体原因可以看这个视频：

[【python】听说因为有GIL，多线程连锁都不需要了？](https://www.bilibili.com/video/BV1ae411Z7WU)

以上示例说明了，在 Python 多线程程序中，用户对共享变量的操作可能会因为数据竞争而产生不可预期的结果。

这个很重要，前面我们说的全局变量会在多线程条件下出错，在当前 CPython 中，由于 GIL 的存在，对象的 `ob_refcount` 修改是线程安全的，不会出现引用计数错误。但如果在 无 GIL 环境 下，同时允许多个线程修改同一个对象的引用计数，就可能出现类似前面示例中用户变量的最终值不可预期的情况。

解决共享资源竞争的常见方法是使用锁。为公共变量加锁可以保证同一时间只有一个线程访问，从而避免数据竞争问题。然而，加锁也可能引入死锁：当两个或多个线程互相等待对方释放锁时，程序会陷入永久等待状态而无法继续执行。下面用一个 Python 示例演示死锁现象：

```python
import threading
import time

lock_a = threading.Lock()
lock_b = threading.Lock()

def thread1():
    lock_a.acquire()
    time.sleep(1)
    lock_b.acquire()

def thread2():
    lock_b.acquire()
    time.sleep(1)
    lock_a.acquire()

t1 = threading.Thread(target=thread1)
t2 = threading.Thread(target=thread2)

t1.start()
t2.start()

t1.join()
t2.join()
```

在这段程序中，线程 1 先获取 `lock_a`，线程 2 先获取 `lock_b`，随后两者互相等待对方释放锁，从而导致 死锁，程序会陷入永久阻塞，无法继续执行。

## GIL 的诞生
为了解决前面的问题，在 CPython 的早期设计中，就采用了单全局锁的解释器模型，这一机制后来被称为全局解释器锁。

简单来说 GIL 是一个互斥锁，它同一时刻只允许一个 Python 线程执行 Python 字节码。

> 如果你不知道什么是 Python 字节码，简单来说，CPython 解析 Python 代码会先把 Python 文件解析成字节码，然后再由虚拟机 PVM 执行字节码。
>

在现代计算任务中，我们通常将程序分为 CPU 密集型 和 IO 密集型：

+ CPU 密集型：大量计算操作，例如矩阵运算、科学计算等
+ IO 密集型：大量等待外部操作完成，例如网络请求、文件读写、数据库操作

GIL 对两种不同类型任务的影响不同：

+ 在 CPU 密集型任务中，线程会频繁获取和释放 GIL，但宏观效果上，多线程 CPU 密集型程序往往表现为串行执行，因此性能提升有限
+ 在 IO 密集型任务中，Python 会在等待 IO 的过程中释放 GIL，使其他线程有机会运行，因此 GIL 对 IO 密集型任务的影响较小

虽然 GIL 常被批评限制多核 CPU 的利用率，但它也有显著优势，下面我们来介绍一下

## GIL 的优势
首先，GIL 的实现非常简单，对于大型工程来说，简单真的是一件非常重要的事，重要到怎么强调都不为过，GIL 非常容易实现，这意味着 CPython 的维护和管理会变得非常容易，对于对每个对象都维护一个锁来说。

其次，GIL 能提升单线程性能。在单线程场景下，CPython 可以避免大量原子操作、锁和内存屏障，从而获得更高的执行效率。这对于 IO 密集型任务尤为重要，因为即便是单线程的并发操作，也可能涉及对同一文件的访问，GIL 可以帮助保证数据一致性。

最后还有很重要的一点，对于 C 扩展来说非常重要，由于 GIL 的存在，C 扩展可以在操作 Python 对象时无需额外考虑线程安全问题，从而降低开发难度，提高可靠性。

以上优势，是 GIL 能够长期存在的主要原因之一

此外，容易维护使得 Python 开发者们可以专心将用户体验放在首位，这也间接促成了 Python 的简单性和易用性，也使得它成为科学计算、数据分析和人工智能领域的首选语言。

## GIL 的劣势
说完了优势，现在让我们来说说它的劣势，GIL 最为人所诟病的一点是它无法充分利用现代计算机的多核性能，使用 Python 开发的程序可能会出现“一核有难，八核围观”的问题。

深入追究原因的话，前面说过，Python 同一时刻只允许一个线程执行字节码，而 CPU 密集型的任务在执行的过程中，会一直占据着 GIL 锁直到计算完成，这就导致了多线程 CPU 密集型任务会变成单线程顺序执行代码。

原因在于，Python 同一时刻只允许一个线程执行字节码。所以从宏观上看，虽然多线程将计算任务分配到不同的 CPU 核心，但程序仍表现为串行执行，从而导致多线程在多核 CPU 上的性能几乎没有提升

例如：

```python
mport threading
import time


def cpu_task():
    x = 0
    for _ in range(50_000_000):
        x += 1


start = time.time()

t1 = threading.Thread(target=cpu_task)
t2 = threading.Thread(target=cpu_task)

t1.start()
t2.start()

t1.join()
t2.join()

print("多线程版本耗时:", time.time() - start)

start = time.time()
cpu_task()
cpu_task()

print(f"单线程版本耗时：{time.time() - start}")

# 多线程版本耗时: 6.41266393661499
# 单线程版本耗时：6.362610101699829
```

在我的 6 核 12 线程的 Intel 10400F CPU 上，有时单线程版本的运行时间甚至比多线程版本更短。这是正常现象，因为多线程调度和上下文切换本身也会消耗资源

多进程程序可以解决 GIL 带来的问题，但是这是另一种情况，这里不做研究。

## GIL 会移除吗？
龟叔在这篇文章中介绍过这个问题，[It isn't Easy to Remove the GIL](https://www.artima.com/weblogs/viewpost.jsp?thread=214235)，简单来说，移除 GIL 后导致的单线程性能大幅度下降让龟叔不能接受，在解决这个问题之前，可能不会让 GIL 移除（3.14 依旧没有解决）。

此外，移除 GIL 意味着可能需要重写现有的所有 C 扩展，这会带来巨大的向前兼容性问题。如果立即移除 GIL，许多依赖现有 Python API 的代码可能都需要调整，这样做的代价多大。

从历史角度来看，Python 创建于 1990 年，GIL 诞生于 1992 年。当时多核 CPU 几乎不存在，直到 2005 年，多核 CPU 才逐渐进入消费级市场。因此，GIL 的设计最初并非问题，而是历史遗留。随着现代计算机多核成为标配，甚至手机、手表、眼镜等设备都具备多核能力，GIL 的局限性才逐渐显现出来。

但是无论如何，nogil 版本的 Python 已经发布，我们可以来尝鲜了

## 尝试自由线程
最后，让我们来尝试一下 nogil 版本的 Python，让我们来执行 3.14 freethread 版本。

让我们使用 uv 来安装 python3.14 自由线程版本

```shell
uv python list -- 显示可用Python版本

cpython-3.14.0a5+freethreaded-windows-x86_64-none    C:\Users\bear\AppData\Roaming\uv\python\cpython-3.14.0a5+freethreaded-windows-x86_64-none\python.exe
```

让我们使用这个版本的 Python 执行上面的 CPU 密集型任务程序查看运行效率

```shell
# 多线程版本耗时: 1.6974902153015137
# 单线程版本耗时：3.315401077270508
```

从测试结果来看，free-threaded Python 在 CPU 密集型多线程任务中实现了真正的并行计算，多线程性能明显优于传统 GIL 版本

但是需要注意的是，这一性能表现高度依赖具体的实现、编译方式和运行环境。官方尚未保证 free-threaded Python 在单线程场景下一定会比普通 Python 更快，因此在实际应用中仍需根据任务类型进行选择。

## 参考
+ [Larry Hastings - Python's Infamous GIL](https://www.youtube.com/watch?v=4zeHStBowEk)
+ [【python】天使还是魔鬼？GIL的前世今生。一期视频全面了解GIL！](https://www.bilibili.com/video/BV1za411t7dR)
+ [【python】听说因为有GIL，多线程连锁都不需要了？](https://www.bilibili.com/video/BV1ae411Z7WU/)
+ [It isn't Easy to Remove the GIL](https://www.artima.com/weblogs/viewpost.jsp?thread=214235)
+ [What Is the Python Global Interpreter Lock (GIL)?](https://realpython.com/python-gil/)
+ [PEP 703 – Making the Global Interpreter Lock Optional in CPython](https://peps.python.org/pep-0703/)
+ [PEP 779 – Criteria for supported status for free-threaded Python](https://peps.python.org/pep-0779/)


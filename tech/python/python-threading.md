---
title: "Python Threading"
date: 2025-12-17T21:27:38+08:00
lastmod: 2025-12-17T21:27:38+08:00
author: 熊大如如
tags: # 标签
  - "python"
description: ""
weight:
slug: ""
summary: "Python threadin 模块"
draft: false # 是否为草稿
mermaid: true #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示路径
cover:
    image: ""
---

## 概述
`threading` 模块提供了一种在单个进程内部并发地运行多个 [线程](https://en.wikipedia.org/wiki/Thread_(computing)) (从进程分出的更小单位) 的方式。 它允许创建和管理线程，以便能够平行地执行多个任务，并共享内存空间。 线程特别适用于 I/O 密集型的任务，如文件操作或发送网络请求，在此类任务中大部分时间都会消耗于等待外部资源

查看 threading 模块的源码，第一句是：

```python
Thread module emulating a subset of Java's threading model.

Thread 模块模拟了 Java 线程模型的一个子集
```

## 运行线程
```python
import time
import threading


def fetch_info():
    time.sleep(5)
    print("Info fetched")


tasks = []
for _ in range(5):
    t = threading.Thread(target=fetch_info)
    tasks.append(t)

print(tasks)
for task in tasks:
    task.start()

print("All tasks done!")
```

执行上面的程序，大约 5 秒打印全部输出语句，输出结果

```python
[<Thread(Thread-1 (fetch_info), initial)>, <Thread(Thread-2 (fetch_info), initial)>, <Thread(Thread-3 (fetch_info), initial)>, <Thread(Thread-4 (fetch_info), initial)>, <Thread(Thread-5 (fetch_info), initial)>]
All tasks done!
Info fetched
Info fetched
Info fetched
Info fetched
Info fetched
```

### 等待主线程
join 方法可以等待子线程执行完成

试想一个结果，如果子线程的工作是去修改修改一个文件，如果子线程没有完成主线程就读取这个文件，这个时候就会发生数据错误。

```python
import time
import threading


def fetch_info():
    time.sleep(5)
    print("Info fetched")


tasks = []
for _ in range(5):
    t = threading.Thread(target=fetch_info)
    tasks.append(t)

print(tasks)
for task in tasks:
    task.start()

for task in tasks:
    task.join()

print("All tasks done!")

```

输出结果：

```python
[<Thread(Thread-1 (fetch_info), initial)>, <Thread(Thread-2 (fetch_info), initial)>, <Thread(Thread-3 (fetch_info), initial)>, <Thread(Thread-4 (fetch_info), initial)>, <Thread(Thread-5 (fetch_info), initial)>]
Info fetched
Info fetched
Info fetched
Info fetched
Info fetched
All tasks done!
```

## 判断线程是否启动
在 Python 多线程中，`threading.Event` 是一种线程间同步机制，用来在多个线程之间传递信号，让线程知道何时可以执行或暂停。

基本概念`Event` 有一个内部标志位，线程可以调用：

+ `wait()` 阻塞等待，直到 flag 为 True
+ `set()`  将 flag 设为 True，唤醒所有等待的线程
+ `clear()` 将 flag 设为 False
+ `is_set()` 查询 flag 状态

示例：

```python
import threading
import time

event = threading.Event()

def worker():
    print("Worker waiting for event...")
    event.wait()  # 阻塞，直到 event 被 set
    print("Worker starts working!")

t = threading.Thread(target=worker)
t.start()

time.sleep(3)
print("Main thread sets event")
event.set()  # 唤醒 worker
```

输出：

```plain
Worker waiting for event...
Main thread sets event
Worker starts working!
```

`worker` 线程在 `event`未触发时会阻塞，直到主线程发信号后才执行。`Event`对象最好只使用一次，因为多线程执行的复杂情况，导致无法预料哪个线程会先执行，是重置 `Event`对象的线程先执行，还是根据 `Event`标志的对象先执行。

## 线程锁
多线程程序共同修改一个资源时，大概率会引发竞态条件，例如：

```python
import threading

a = 0


def add():
    global a
    for i in range(1000000):
        a += 1


t1 = threading.Thread(target=add)
t2 = threading.Thread(target=add)
t1.start()
t2.start()


t1.join()
t2.join()

print(a)
```

> 注意：以上程序在 py3.9 及以下版本会不等于 2000000，但是在 3.10 以上会等于 2000000 究其原因可以看这个视频 [【python】听说因为有GIL，多线程连锁都不需要了？](https://www.bilibili.com/video/BV1ae411Z7WU/?spm_id_from=333.1387.search.video_card.click&vd_source=9bcfca4212a15c95e7c4eaa938a7fc3f)
>

为了防止多线程程序竞争同一资源导致的问题，我们可以给资源加 Lock 锁

```python
import threading

a = 0

lock = threading.Lock()


def add():
    lock.acquire()
    global a
    for i in range(1000000):
        a += 1
    lock.release()


t1 = threading.Thread(target=add)
t2 = threading.Thread(target=add)
t1.start()
t2.start()


t1.join()
t2.join()

print(a)
```

这样就能杜绝多线程竞争冒险。现在让我们来解决死锁的问题：

```python
import threading

lock = threading.Lock()


def f1():
    lock.acquire()
    f2()
    lock.release()


def f2():
    lock.acquire()
    print("ok")
    lock.release()


t1 = threading.Thread(target=f1)
t2 = threading.Thread(target=f2)
t1.start()
t2.start()
t1.join()
t2.join()

```

看这段代码，如果正常运行，那么程序必定卡死，因为两个线程如果其一没有释放锁，那么另一个就无法拿到锁，互相等待。

```python
import threading

lock = threading.RLock()


def f1():
    lock.acquire()
    f2()
    lock.release()


def f2():
    lock.acquire()
    print("ok")
    lock.release()


t1 = threading.Thread(target=f1)
t2 = threading.Thread(target=f2)
t1.start()
t2.start()
t1.join()
t2.join()

```

使用可重入锁 RLock 对象就可以了，重入锁必须由获取它的线程释放。一旦线程获得了重入锁，同一个线程再次获取它将不阻塞；线程必须在每次获取它时释放一次。

## 限制并发数量
```python
import threading
import time

sem = threading.Semaphore(2)

def worker(i):
    with sem:
        print(f"worker {i} enter")
        time.sleep(2)
        print(f"worker {i} leave")

threads = []
for i in range(5):
    t = threading.Thread(target=worker, args=(i,))
    t.start()
    threads.append(t)

for t in threads:
    t.join()
```

这个类的功能是限制同时能访问资源的线程数量，上面的示例中就是两个。

## 线程等待
```python
import threading
import time

cond = threading.Condition()
queue = []

def producer():
    time.sleep(1)
    with cond:
        queue.append("task")
        print("produce task")
        cond.notify()

def consumer():
    with cond:
        while not queue:
            print("consumer wait")
            cond.wait()
        item = queue.pop()
        print("consume", item)

t1 = threading.Thread(target=consumer)
t2 = threading.Thread(target=producer)

t1.start()
t2.start()

t1.join()
t2.join()
```

## 延迟执行
```python
import threading
import time

def task():
    print("task run", time.time())

print("start", time.time())

t = threading.Timer(2, task)
t.start()
```

两秒后，线程会打印字符串，Timer 的作用就是执行时间后运行线程。

## 线程统一等待
让一组线程在某个同步点全部到齐后，再一起继续执行

```python
import threading
import time

barrier = threading.Barrier(3)

def worker(i):
    print(f"worker {i} prepare")
    time.sleep(i)      # 模拟准备时间不同
    print(f"worker {i} wait")
    barrier.wait()
    print(f"worker {i} start")

threads = []
for i in range(3):
    t = threading.Thread(target=worker, args=(i,))
    t.start()
    threads.append(t)

for t in threads:
    t.join()
```

执行这段程序，你会发现 `start`最后统一打印。


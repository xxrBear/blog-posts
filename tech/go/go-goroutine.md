---
title: "Go 协程"
date: 2025-11-17T17:48:30+08:00
lastmod: 2025-11-17T17:48:30+08:00
author: 熊大如如
tags: # 标签
  - "go"
description: ""
weight:
slug: ""
summary: "Go 协程知识点"
draft: false # 是否为草稿
mermaid: true #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示路径
cover:
  image: "https://raw.githubusercontent.com/xxrBear/image/master/blog/gopher_small_1.png"  # 文章的图片
---

## 基本概念与启动
goroutine 是 Go 的轻量线程，由 Go 运行时管理 用 `go f()` 启动：非阻塞，函数在后台并发执行

```go
package main
import (
    "fmt"
    "time"
)

func say(s string) {
    for i:=0;i<3;i++ {
        fmt.Println(s, i)
        time.Sleep(100 * time.Millisecond)
    }
}

func main() {
    go say("world") // 在后台并发执行
    say("hello")
}
```

解释：主 goroutine 会先执行 `say("hello")`，同时另一个 goroutine 在打印 `world`。如果 main 提早退出，后台 goroutine 会被杀掉

## 自定义使用核心
+ 并发：同时有多个任务在处理
+ 并行：任务在多个 CPU 核心上同时运行
+ `runtime.GOMAXPROCS(n)` 控制能并行运行的 OS 线程数

示例：

```go
package main
import (
    "fmt"
    "runtime"
    "time"
)
func busy(id int) {
    for i:=0;i<5;i++ {
        fmt.Println("worker", id, "step", i)
        time.Sleep(100 * time.Millisecond)
    }
}
func main() {
    runtime.GOMAXPROCS(1) // 强制单核执行（观察并发但非并行）
    go busy(1)
    go busy(2)
    time.Sleep(700 * time.Millisecond)
    fmt.Println("Goroutines:", runtime.NumGoroutine())
}
```

解释：即使 GOMAXPROCS=1，也能并发执行（协作式调度），但不会真正并行占用多核

## channel 基础
+ channel 是 goroutine 间通信的管道
+ `ch := make(chan T)`（无缓冲，同步）或 `make(chan T, N)`（缓冲，异步）

无缓冲示例（同步）：

```go
package main
import "fmt"
func main() {
    ch := make(chan int)
    go func() {
        ch <- 42 // 发送者阻塞直到有人接收
    }()
    fmt.Println(<-ch) // 接收
}
```

缓冲示例（异步）：

```go
ch := make(chan int, 2)
ch <- 1
ch <- 2
// 发送不会阻塞，除非缓冲满
fmt.Println(<-ch, <-ch)
```

## channel 方向、关闭、range 接收
+ 声明只发送或只接收的 channel：`chan<- int` 或 `<-chan int`
+ `close(ch)`：关闭 channel，接收者可接收到零值并继续读取（以检测结束）
+ 用 `for v := range ch` 迭代直到 channel 关闭

示例：

```go
package main
import "fmt"
func producer(ch chan<- int) {
    for i:=0;i<5;i++ { ch <- i }
    close(ch)
}
func main() {
    ch := make(chan int)
    go producer(ch)
    for v := range ch {
        fmt.Println("recv", v)
    }
}
```

注意：不要向已关闭的 channel 发送，否则 panic

## select：多路复用与超时/默认分支
+ `select` 让你在多个 channel 操作中等待任一可用
+ 支持 `default`（无阻塞）和 `time.After` 用于超时

示例（超时）：

```go
package main
import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan string)
    go func() {
        time.Sleep(500 * time.Millisecond)
        ch <- "result"
    }()
    select {
    case res := <-ch:
        fmt.Println(res)
    case <-time.After(200 * time.Millisecond):
        fmt.Println("timeout")
    }
}
```

解释：这里会打印 `timeout`，因为接收比超时慢

## WaitGroup：等待一组 goroutine 完成
+ `sync.WaitGroup` 用于等待多个 goroutine 完成，常见模式：`wg.Add(n)` → 每个 goroutine 执行 `defer wg.Done()` → `wg.Wait()`

示例：

```go
package main
import (
    "fmt"
    "sync"
)

func worker(id int, wg *sync.WaitGroup) {
    defer wg.Done()
    fmt.Println("work", id)
}

func main() {
    var wg sync.WaitGroup
    for i:=0;i<3;i++ {
        wg.Add(1)
        go worker(i, &wg)
    }
    wg.Wait()
    fmt.Println("all done")
}
```

## Mutex / RWMutex：互斥与读写锁
+ `sync.Mutex`：互斥锁，保护共享数据
+ `sync.RWMutex`：读写锁，允许并发读，但写互斥

示例（Mutex）：

```go
var (
    counter int
    mu sync.Mutex
)

func inc() {
    mu.Lock()
    counter++
    mu.Unlock()
}
```

示例（RWMutex）：

```go
var (
    data map[string]int
    rw sync.RWMutex
)

func read(k string) int {
    rw.RLock()
    v := data[k]
    rw.RUnlock()
    return v
}

func write(k string, v int) {
    rw.Lock()
    data[k] = v
    rw.Unlock()
}
```

## 原子操作
+ 对于简单的整型计数，`sync/atomic` 提供比 Mutex 更快的原子操作（无锁）
+ 常用：`atomic.AddInt64`, `atomic.LoadInt64`, `atomic.CompareAndSwapInt64`

示例：

```go
package main
import (
    "fmt"
    "sync"
    "sync/atomic"
)

func main(){
    var counter int64
    var wg sync.WaitGroup
    for i:=0;i<1000;i++ {
        wg.Add(1)
        go func(){ atomic.AddInt64(&counter, 1); wg.Done() }()
    }
    wg.Wait()
    fmt.Println("counter", atomic.LoadInt64(&counter))
}
```

注意：原子适合简单场景，复杂结构仍需 Mutex

## 管道化
+ Pipelining：把处理分成多个 stage，通过 channel 串联起来
+ Fan-out：多个 worker 接收同一输入 channel 并发工作
+ Fan-in：多个 worker 的输出合并回一个 channel

示例（简单 pipeline + fan-out/fan-in）：

```go
package main
import (
    "fmt"
    "sync"
)

func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func(){
        for _, n := range nums { out <- n }
        close(out)
    }()
    return out
}

func sq(in <-chan int) <-chan int {
    out := make(chan int)
    go func(){
        for n := range in { out <- n*n }
        close(out)
    }()
    return out
}

func main() {
    in := gen(2,3,4)
    out := sq(in)
    for v := range out { fmt.Println(v) } // 4,9,16
}
```

示例：

```go
func worker(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for j := range jobs {
        results <- j * 2
    }
}

func main() {
    jobs := make(chan int, 5)
    results := make(chan int, 5)
    var wg sync.WaitGroup
    for w:=1;w<=3;w++ { wg.Add(1); go worker(w,jobs,results,&wg) }
    for j:=1;j<=5;j++ { jobs <- j }
    close(jobs)
    go func(){ wg.Wait(); close(results) }()
    for r := range results { fmt.Println(r) }
}
```

## goroutine 泄漏与预防
+ 泄漏场景：goroutine 在等待无法返回的 channel 或永远阻塞
+ 预防：使用 `context` + timeout、确保 channel 最终会被关闭、select 带超时或取消分支

常见泄漏示例（错误）：

```go
func f(ch <-chan int) {
    <-ch // 如果 ch 永远不关闭，这个 goroutine 永久阻塞
}
```

改进（带取消）：

```go
func f(ctx context.Context, ch <-chan int) {
    select {
    case <-ch:
    case <-ctx.Done():
    }
}
```

## runtime 包相关
+ `runtime.Gosched()`：让出当前线程，调度器去运行其他 goroutine（相当于短暂停）
+ `runtime.NumGoroutine()`：返回当前 goroutine 数
+ `runtime.GOMAXPROCS(n)`：设置可并行的操作系统线程数
+ `runtime.Stack` / `pprof` 可用于获取 goroutine 堆栈

示例：

```go
import "runtime"
fmt.Println("goroutines:", runtime.NumGoroutine())
```

## 设计与最佳实践总结
+ 尽量避免共享可变状态：优先通过 channel 传递数据（消息传递优于共享内存）
+ 用 context 传播取消/超时：每个后台 goroutine 都考虑可取消性
+ 短小单一职责 goroutine：一个 goroutine 做一件事（易理解、易管理）
+ 避免长时间阻塞在外部资源：用超时、重试、断路器模式保护
+ 及时关闭 channel：producer 负责 close，consumer 检测结束
+ 使用 WaitGroup 管理生命周期：主流程等待 worker 结束
+ 加入日志与指标（metrics）：记录 goroutine 启停、错误、处理速率
+ 使用 race detector 做开发期检查，并用 pprof 做性能分析

---
title: "Go 异常"
date: 2025-11-03T17:39:45+08:00
lastmod: 2025-11-03T17:39:45+08:00
author: 熊大如如
tags: # 标签
  - "go"
summary: "Go 语言异常"
draft: false # 是否为草稿
mermaid: true #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示路径
cover:
    image: "https://raw.githubusercontent.com/xxrBear/image/master/blog/gopher_small_1.png"
---

## 一、Go 的异常体系
Go 语言中没有像 Python、Java 那样的 `try catch`异常处理，因为 Go 的作者认为那样的异常处理方式容易被滥用而且向上抛出异常也会让程序性能不佳。取而代之的 Go 语言使用返回值来处理普通异常，使用 `panic`与 `recover`来处理严重异常，下面我们逐一介绍

## 二、Go 的错误机制
在 Go 中，`error` 是一个内建接口

```go
type error interface {
    Error() string
}
```

Go 语言的 `errors`包下有一个`errorString`结构体实现了`error`接口，当你想自定义一个异常的时候，你可以使用这个接口，例如：

```go
package main

import "fmt"
import "errors"


var err = errors.New("Not found error")

func main() {
	fmt.Printf("error: %v", err)
}
// error: Not found error
```

下面我们介绍一下 Go 语言推荐的错误处理方式：

```go
content, err := readFile("data.txt")

if err != nil {
    log.Println("read failed:", err)
    return
}

fmt.Println(content)
```

你看到了，`if err != nil`语句你以后会天天写，早点习惯它吧！

除了上面的错误创建方式，Go 语言也提供一种`fmt.Errorf`方法

```go
err := fmt.Errorf("invalid value: %d", x)
```

## 三 、运行时异常与恢复
`panic` 方法会立即停止当前函数执行，并开始向上回溯调用栈

```go
func main() {
    panic("something went wrong")
}
```

输出：

```plain
panic: something went wrong
goroutine 1 [running]:
main.main()
```

我们已经学会了怎么抛出异常，现在让我来学习如何处理异常吧，处理异常我们需要使用 `recover`方法`recover` 只能在`defer`函数内部调用，否则无效

```go
func safeRun() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered:", r)
        }
    }()
    panic("crash!")
}
```

> `recover()` 返回 panic 的参数，可以用来判断 panic 原因
>

让我通过一个示例看看多个 defer 函数时的调用顺序

```go
func f() {
    defer fmt.Println("A")
    defer fmt.Println("B")
    panic("C")
}
```

输出：

```plain
B
A
panic: C
```

> defer 是后进先出
>

## 四、调用栈清理机制
`defer` 延迟函数调用，在函数返回前执行

```go
func main() {
    defer fmt.Println("world")
    fmt.Println("hello")
}
```

输出：

```plain
hello
world
```

`defer`函数常常用于以下场景中

+ 关闭文件、连接

```go
f, _ := os.Open("a.txt")
defer f.Close()
```

+ 解锁 mutex

```go
mu.Lock()
defer mu.Unlock()
```

+ panic 恢复

```go
defer func() {
    if r := recover(); r != nil {
        log.Println("Recovered:", r)
    }
}()
```

参数在 defer 声明时求值，不是执行时

```go
func f() {
    x := 1
    defer fmt.Println(x)
    x = 2
}
f() // 输出 1
```

## 五、实践模式与技巧
+ 封装统一错误处理函数

```go
func handleErr(err error) {
    if err != nil {
        log.Println("Error:", err)
    }
}
```

+ 统一 panic 捕获

```go
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                http.Error(w, "Internal Server Error", 500)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

+ 自定义错误分类

```go
var (
    ErrNotFound = errors.New("not found")
    ErrTimeout  = errors.New("timeout")
)
```

使用 `errors.Is(err, ErrNotFound)` 判断错误类型


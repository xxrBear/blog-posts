---
title: "Go 异常"
date: 2025-11-03T17:39:45+08:00
lastmod: 2025-11-03T17:39:45+08:00
author: 熊大如如
tags: # 标签
  - "go"
description: ""
weight:
slug: ""
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
## 一、Go 的异常体系总览
Go 中有三种异常相关机制：

| 机制      | 用途                   | 类似语言       |
| --------- | ---------------------- | -------------- |
| `error`   | 可预期业务错误、IO错误 | 类似返回错误码 |
| `panic`   | 不可恢复异常           | 类似抛异常     |
| `recover` | 从 `panic` 中恢复      | 类似 catch     |


> 设计理念：Go 鼓励开发者显式处理错误，不隐藏、不捕获，而是直接返回
>

## 二、Go 的显式错误机制
### 1. error 本质
在 Go 中，`error` 是一个内建接口：

```go
type error interface {
    Error() string
}
```

任何实现了 `Error()` 方法的类型，都是 `error`

### 2. 返回错误的惯用法
```go
func readFile(name string) (string, error) {
    data, err := os.ReadFile(name)
    if err != nil {
        return "", err
    }
    return string(data), nil
}
```

> 函数返回值规则：结果值 + error 是 Go 最常见的函数签名
>

### 3. 错误处理的推荐方式
```go
content, err := readFile("data.txt")
if err != nil {
    log.Println("read failed:", err)
    return
}
fmt.Println(content)
```

> Go 鼓励早返回，而不是 try-catch
>

### 4. 创建错误
#### 使用标准库
```go
err := errors.New("something went wrong")
```

#### 使用 `fmt.Errorf`
```go
err := fmt.Errorf("invalid value: %d", x)
```

#### 包装错误（Go 1.13+）
```go
err := fmt.Errorf("read config: %w", ioErr)
```

### 5. 错误解包与判断
#### `errors.Is`
判断某错误是否等于目标错误：

```go
if errors.Is(err, os.ErrNotExist) {
    fmt.Println("file not found")
}
```

#### `errors.As`
判断并提取错误类型：

```go
var pathErr *os.PathError
if errors.As(err, &pathErr) {
    fmt.Println("op:", pathErr.Op)
}
```

### 6. 自定义错误类型
```go
type MyError struct {
    Code int
    Msg  string
}

func (e *MyError) Error() string {
    return fmt.Sprintf("code=%d, msg=%s", e.Code, e.Msg)
}

func doSomething() error {
    return &MyError{404, "not found"}
}
```

### 7. 错误的 nil 陷阱
```go
func f() error {
    var e *MyError = nil
    return e // 返回非空接口
}
```

接口中保存了类型信息，因此不为 `nil`

> 正确写法：
>

```go
if e == nil {
    return nil
}
return e
```

## 三、panic 与 revocer
### 1. `panic`：触发运行时异常
`panic` 会立即停止当前函数执行，并开始向上回溯调用栈

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

### 2. 触发 panic 的常见情况
| 类型           | 示例             |
| -------------- | ---------------- |
| 数组越界       | `a[10]`          |
| 空指针解引用   | `*nilPtr`        |
| 除以 0         | `x / 0`          |
| 手动调用 panic | `panic("error")` |


### 3. `recover`：从 panic 中恢复
`recover` 只能在 defer 函数内部调用，否则无效

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

### 4. panic 的传播机制
+ `panic` 会逐层向上抛；
+ 每一层的 `defer` 都会被执行；
+ 若无任何 `recover` 捕获，最终程序崩溃。

### 5. panic 与 defer 的调用顺序
示例：

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

## 四、defer 与调用栈清理机制
### 1. defer 基础
`defer` 延迟函数调用，在函数返回前执行。

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

### 2. defer 的常见用途
+ 关闭文件、连接：

```go
f, _ := os.Open("a.txt")
defer f.Close()
```

+ 解锁 mutex：

```go
mu.Lock()
defer mu.Unlock()
```

+ panic 恢复：

```go
defer func() {
    if r := recover(); r != nil {
        log.Println("Recovered:", r)
    }
}()
```

### 3. defer 参数求值时机
参数在 defer 声明时求值，不是执行时。

```go
func f() {
    x := 1
    defer fmt.Println(x)
    x = 2
}
f() // 输出 1
```

## 五、error 与 panic 的区别
| 特性         | error                | panic                   |
| ------------ | -------------------- | ----------------------- |
| 用途         | 业务逻辑错误         | 不可恢复错误            |
| 处理方式     | 函数返回显式处理     | 使用 defer+recover 捕获 |
| 传播方式     | 手动传递             | 自动回溯栈              |
| 推荐使用场景 | 文件未找到、网络错误 | 索引越界、逻辑 bug      |


> 最佳实践：只有在真正无法继续运行时才使用 panic
>

## 六、实践模式与技巧
### 1. 封装统一错误处理函数
```go
func handleErr(err error) {
    if err != nil {
        log.Println("Error:", err)
    }
}
```

### 2. 统一 panic 捕获
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

### 3. 自定义错误分类
```go
var (
    ErrNotFound = errors.New("not found")
    ErrTimeout  = errors.New("timeout")
)
```

使用 `errors.Is(err, ErrNotFound)` 判断错误类型

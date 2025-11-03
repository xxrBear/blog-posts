---
title: "Go 接口"
date: 2025-11-03T17:38:40+08:00
lastmod: 2025-11-03T17:38:40+08:00
author: 熊大如如
tags: # 标签
  - "go"
description: ""
weight:
slug: ""
summary: "Go 语言接口知识点"
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

## 一、接口的基本概念
### 1. 接口的定义
接口是一组方法的集合，用于定义对象的行为规范

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

任何实现了 `Write([]byte) (int, error)` 方法的类型，都自动实现了 `Writer` 接口，无需显式声明

### 2. 接口的隐式实现机制
Go 没有像 Java 一样用 `implements`，而是结构体自动实现接口：

```go
type MyWriter struct{}

func (w MyWriter) Write(p []byte) (int, error) {
    fmt.Println(string(p))
    return len(p), nil
}

func main() {
    var w Writer
    w = MyWriter{} // 自动匹配方法签名
    w.Write([]byte("Hello"))
}
```

只要方法签名一致，就视为实现接口。这就是 Go 的鸭子类型

### 3. 接口类型变量
接口变量可以存放任何实现了该接口的对象

```go
var w io.Writer
w = os.Stdout
w = new(bytes.Buffer)
```

Go 会在运行时维护一个接口值，包含：

+ 动态类型
+ 动态值

## 二、接口的底层结构与原理
在底层，接口值是一个结构体：

```go
interface {
    tab  *itab   // 类型信息表
    data unsafe.Pointer // 实际数据指针
}
```

> Go 编译器在编译期会构造一个 `itab`，用来表示 类型 T 实现了接口 I
>

当你调用接口方法时：

1. Go 通过 `itab` 查找到实际类型的方法表；
2. 再通过函数指针调用具体实现；
3. 这就是接口调用的动态分派

### 1. 空接口
空接口没有任何方法，因此所有类型都实现了它

```go
var x interface{}
x = 42
x = "hello"
x = struct{}{}
```

这使得空接口常被用于：

+ 泛型容器（如 `map[string]interface{}`）
+ 动态类型处理（如 JSON、反射）

### 2. 类型断言
```go
var w io.Writer
w = os.Stdout

f, ok := w.(*os.File)  // 安全的类型断言
if ok {
    fmt.Println("It's a file:", f.Name())
}
```

+ `w.(T)`：如果类型不匹配，会 panic
+ `w.(T)` + `ok`：安全断言，不 panic

### 3. 类型 switch
```go
switch v := x.(type) {
case string:
    fmt.Println("string:", v)
case int:
    fmt.Println("int:", v)
default:
    fmt.Println("unknown")
}
```

用于动态判断接口中存储的真实类型。

## 三、接口的高级特性
### 1. 接口嵌套
接口可以组合其他接口：

```go
type Reader interface { Read(p []byte) (n int, err error) }
type Writer interface { Write(p []byte) (n int, err error) }

type ReadWriter interface {
    Reader
    Writer
}
```

相当于多重继承但更灵活

### 2. 值接收者 vs 指针接收者
方法的接收者类型不同，会影响接口的实现：

```go
type Cat struct{}

func (c Cat) Speak() { fmt.Println("meow") }

type Speaker interface {
    Speak()
}

var s Speaker
s = Cat{}   // OK
s = &Cat{}  // OK
```

如果方法是指针接收者：

```go
func (c *Cat) Speak() { fmt.Println("meow") }

s = &Cat{} // OK
s = Cat{}  // 不行
```

> 结论：
>

+ 值接收者方法，值和指针都实现接口；
+ 指针接收者方法，只有指针实现接口。

### 3. 接口间的类型转换
```go
var rw io.ReadWriter
var r io.Reader = rw  // 向上转换
```

可以从大接口赋值给小接口，反之不行

### 4. 接口的零值与比较
```go
var x interface{}
fmt.Println(x == nil) // true

x = (*int)(nil)
fmt.Println(x == nil) // false
```

因为：

+ 第一个是动态类型和值都为 nil；
+ 第二个类型不为 nil，值为 nil。

> 接口为 nil 当且仅当：type == nil && value == nil
>

## 四、接口的常见实践
### 1. 多态设计
```go
type Shape interface {
    Area() float64
}

type Circle struct{ R float64 }
func (c Circle) Area() float64 { return 3.14 * c.R * c.R }

type Rect struct{ W, H float64 }
func (r Rect) Area() float64 { return r.W * r.H }

func PrintArea(s Shape) {
    fmt.Println("Area:", s.Area())
}
```

### 2. Mock 测试与依赖注入
接口可用于单元测试，轻松 mock 出依赖

```go
type DB interface {
    Query(string) string
}

func GetUser(db DB, id int) string {
    return db.Query(fmt.Sprintf("SELECT name FROM user WHERE id=%d", id))
}

// Mock
type MockDB struct{}
func (m MockDB) Query(q string) string { return "mock_user" }
```

### 3. 空接口与反射
```go
func PrintAny(x interface{}) {
    v := reflect.ValueOf(x)
    fmt.Println("type:", v.Type(), "value:", v)
}
```

## 五、接口的陷阱与性能
### 1. 接口 nil 陷阱
```go
func f() error {
    var p *MyError = nil
    return p // 返回非空接口
}
```

因为接口中记录了类型信息，返回的接口值并不为 nil 解决：`if err != nil` 判断时要谨慎

### 2. 接口调用的开销
接口调用比直接调用略慢，因为需要通过 itab 查找方法表并间接调用。在性能关键路径中应尽量避免大量小粒度接口调用。

### 3. 接口复制陷阱
接口赋值是复制整个接口值结构体，如果底层值较大且被共享，可能引发不期望的复制或逃逸。

### 4. 空接口类型歧义
空接口常导致类型信息丢失，不便于静态分析。建议在可能的情况下使用 泛型替代

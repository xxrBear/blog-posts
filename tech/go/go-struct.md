---
title: "Go 结构体"
date: 2025-10-29T22:12:31+08:00
lastmod: 2025-10-29T22:12:31+08:00
author: 熊大如如
tags: # 标签
  - "go"
description: ""
weight:
slug: ""
summary: "Go 语言结构体"
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

## 简介
Go 语言的结构体的作用是：

> 把多个不同类型的数据组合成一个整体，形成一种自定义的数据类型
>

```go
type Person struct {
    Name string
    Age  int
}
```

没有结构体时，你只能写：

```go
var name string
var age int
```

这样变量是分散的，不方便传递和管理

## 可复用性
Go 没有传统的类，也没有继承、构造函数、重载这些概念。但结构体 + 方法就是 Go 的 面向对象核心机制

```go
type Dog struct {
    Name string
}

func (d *Dog) Bark() {
    fmt.Println(d.Name, "is barking")
}

d := Dog{"Rex"}
d.Bark() // Rex is barking
```

> 所以在 Go 中，结构体 + 方法组合后，就形成了一个“对象”式的编程模型
>

## 工程用途
+ 请求/响应模型

```go
type CreateUserRequest struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

type CreateUserResponse struct {
    ID int `json:"id"`
}
```

框架如 `gin` / `fiber` 都用结构体来解析 JSON、校验参数

+ 数据库映射

在 `gorm` 等 ORM 框架中，每个结构体就代表一张表。

```go
type User struct {
    ID   uint   `gorm:"primaryKey"`
    Name string `gorm:"size:100"`
    Age  int
}
```

ORM 会自动根据结构体字段生成 SQL 表结构。

+ 配置管理

读取配置文件时，可以自动映射到结构体：

```go
type Config struct {
    Port int    `json:"port"`
    Env  string `json:"env"`
}
```

读取配置文件后，结构体直接承载配置信息

+ 消息与数据传输结构

网络通信中，结构体是消息体的载体

```go
type TradeMessage struct {
    TradeID string
    Amount  float64
    Time    time.Time
}
```

+ 组合与复用逻辑

Go 没有继承，但可以用结构体嵌入实现复用

```go
type BaseModel struct {
    ID        uint
    CreatedAt time.Time
}

type User struct {
    BaseModel
    Name string
}
```

`User` 自动拥有 `ID` 和 `CreatedAt` 字段，就像继承一样

## 类型系统
Go 是强类型语言。结构体允许你定义自己的复杂类型，然后让编译器进行严格的类型检查

```go
type Account struct {
    ID   string
    Cash float64
}

func Deposit(a *Account, amt float64) {
    a.Cash += amt
}
```

这样可以防止使用错误的数据类型（比如传错字段、漏字段），让编译器在编译时就能发现逻辑错误

## 定义与声明
+ 定义结构体类型

```go
type Person struct {
    Name string
    Age  int
}
```

`Person` 是一个自定义类型，由两个字段组成

+ 声明变量的多种方式

```go
var p1 Person                 // 零值初始化
p2 := Person{}                // 同上
p3 := Person{Name: "Tom", Age: 18}  // 指定字段初始化
p4 := Person{"Jerry", 20}     // 按顺序初始化（不推荐）
p5 := &Person{"Alice", 25}    // 返回结构体指针
```

> 按顺序初始化在字段变更后容易出错，不推荐
>

## 字段
**字段类型与可见性**

+ 字段名首字母大写可导出
+ 小写仅包内可见

例如：

```go
type user struct {
    Name string // 可导出
    age  int    // 仅包内可见
}
```

**字段标签**

字段后可加反引号字符串，用于反射：

```go
type Student struct {
    Name string `json:"name" db:"student_name"`
}
```

`encoding/json`、`gorm`、`protobuf` 等库都会解析这些 tag

## 指针与访问方式
```go
p := &Person{Name: "Tom", Age: 18}
fmt.Println(p.Name) // 自动解引用，相当于 (*p).Name
```

Go 会自动帮你解引用，所以 `p.Name` 和 `(*p).Name` 是等价的

## 零值与比较
**零值初始化**

所有字段会自动设置为零值：

```go
var p Person
fmt.Println(p.Name) // ""
fmt.Println(p.Age)  // 0
```

**可比较性**

+ 结构体可比较当且仅当所有字段都可比较

```go
type Point struct{ X, Y int }
p1 := Point{1, 2}
p2 := Point{1, 2}
fmt.Println(p1 == p2) // true
```

> 包含切片、map、函数的结构体不可比较
>

## 嵌套
Go 没有继承，但可以通过结构体嵌套实现组合与“伪继承”

```go
type Base struct {
    ID int
}

type User struct {
    Base
    Name string
}
```

访问：

```go
u := User{Base{1}, "Tom"}
fmt.Println(u.ID) // 自动提升字段，相当于 u.Base.ID
```

**同名字段冲突**

```go
type A struct { Name string }
type B struct { Name string }
type C struct {
    A
    B
}
c := C{}
c.A.Name = "a"
c.B.Name = "b"
fmt.Println(c.A.Name, c.B.Name)
```

> 若直接访问 `c.Name` 会编译错误：`ambiguous selector Name`
>

## 方法绑定
方法可以绑定到结构体或其指针上：

```go
func (p Person) SayHello() { fmt.Println("Hi,", p.Name) }
func (p *Person) GrowUp()  { p.Age++ }
```

+ 值接收者：方法操作副本（拷贝）
+ 指针接收者：方法操作原对象

Go 会自动帮你转换：

```go
p := Person{"Tom", 18}
p.GrowUp() // 自动取址
```

## 内存布局与对齐
**字段排列影响内存占用**

字段会按声明顺序排列，并遵守对齐规则：

+ 各字段起始地址必须是其类型大小的倍数
+ 可用 `unsafe.Sizeof` 查看

```go
type A struct {
    a byte
    b int32
    c int64
}
fmt.Println(unsafe.Sizeof(A{})) // 16 或 24，取决于平台
```

> 优化方法：把小字段放一起可减少内存填充
>

## 反射
通过 `reflect` 包可以动态操作结构体字段：

```go
t := reflect.TypeOf(Student{})
for i := 0; i < t.NumField(); i++ {
    field := t.Field(i)
    fmt.Println(field.Name, field.Tag.Get("json"))
}
```

修改值需通过`reflect.Value`，且目标必须是可寻址的

## JSON 与数据库
**JSON 序列化**

```go
type User struct {
    Name string `json:"name"`
    Age  int    `json:"age,omitempty"`
}
u := User{"Tom", 0}
data, _ := json.Marshal(u)
fmt.Println(string(data)) // {"name":"Tom"}
```

`omitempty` 表示零值字段会被忽略

## 嵌入接口字段
接口字段允许动态赋值：

```go
type Logger interface{ Log(msg string) }

type MyStruct struct {
    Logger
}

func (m *MyStruct) Do() {
    m.Log("something happened")
}
```

只要在实例化时赋值：

```go
m := &MyStruct{Logger: log.New(os.Stdout, "", 0)}
m.Do()
```





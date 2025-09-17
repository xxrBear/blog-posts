---
title: "Go 函数"
date: 2025-09-17T17:52:56+08:00
lastmod: 2025-09-17T17:52:56+08:00
author: 熊大如如
tags: # 标签
  - "go"
description: ""
weight:
slug: ""
summary: "Go 语言函数知识点"
draft: false # 是否为草稿
mermaid: true #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示路径
cover:
    image: "https://raw.githubusercontent.com/xxrBear/image/master/blog/go.jpg"
---

## 一、 定义一个函数
```go
func main() {
    fmt.Println("Hello World!")
}
```

注意：你不能把函数的第一个 `{`另起一行写，因为这会让编译器报错

> syntax error: unexpected semicolon or newline before {
>

原因是 go 语言的编译器会为每行自动添加分号 `;`结尾，`func main();`显然是错误的。

## 二、函数内的自动推断类型
Go 语言在函数中，可以使用 `:=`实现自动类型推断，但是只能用在函数中，不能用于全局变量。

```go
func main() {
    a := 1
}
```

## 三、init 函数
Go 语言每个文件可以有多个 `init`函数，执行顺序是从上到下，你可以初始化变量。`init`函数会自动运行，无需手动调用。

```go
func init() {
    fmt.Println("init")
}
```

## 四、参数传递
Go 语言的默认参数传递是值传递，也就是说传入函数内部的参数，会直接复制一份，不会修改原始值。

```go
func main() {
	s := []string{"1", "2", "3"}
	fmt.Println(s)

	fmt.Println(adapteString(s))

	fmt.Println(s)
}

func adapteString(s []string) []string {
	return append(s, "4")
}

// 输出
[1 2 3]
[1 2 3 4]
[1 2 3]
```

如果你想要使用`引用传递`你需要使用引用类型的数据，例如切片、map、chan、指针。

## 五、变长参数
```go
func greet(prefix string, names ...string) {
    for _, name := range names {
        fmt.Println(prefix, name)
    }
}

func main() {
    greet("Hello", "Tom", "Jerry", "Spike")
    // 输出：
    // Hello Tom
    // Hello Jerry
    // Hello Spike
}
```

如果你已经有一个切片，可以用展开运算符 `...` 传给可变长参数

```go
func main() {
    arr := []int{1, 2, 3}
    fmt.Println(sum(arr...))  // 展开切片传递
}
```

## 六、defer
你可以把它理解成：在函数退出之前，一定要做的事 类似 Python 的 finally

### 基本用法
```go
func main() {
    fmt.Println("A")
    defer fmt.Println("B")
    fmt.Println("C")
}
```

输出结果：

```plain
A
C
B
```

### 多个 defer
多个 `defer` 会 按栈的方式（后进先出 LIFO） 执行：

```go
func main() {
    defer fmt.Println("1")
    defer fmt.Println("2")
    defer fmt.Println("3")
}
```

输出：

```plain
3
2
1
```

### 常见用途
+ 资源释放

```go
f, err := os.Open("data.txt")
if err != nil {
    log.Fatal(err)
}
defer f.Close()  // 无论函数如何返回，都会关闭文件
```

+ 解锁 / 释放锁

```go
mu.Lock()
defer mu.Unlock()
// 临界区代码
```

+ 数据清理

```go
func demo() {
    fmt.Println("start")
    defer fmt.Println("clean up")
    panic("something wrong") // 即使 panic 了，defer 仍会执行
}
```

输出：

```plain
start
clean up
panic: something wrong
```

### defer 的执行时机
+ 在函数返回之前执行（包括正常 `return` 或 `panic`）
+ 表达式参数在定义时求值，不是在执行时求值：

```go
func main() {
    x := 10
    defer fmt.Println(x) // 立即捕获 x 的值 10
    x = 20
}
// 输出 10
```

## 七、匿名函数
在 Go 语言中，匿名函数就是没有名字的函数。它们可以像普通函数一样使用，只是不用定义函数名。匿名函数常用于：

+ 临时使用一次的逻辑
+ 作为函数参数传递
+ 在函数内部形成闭包

### 立即调用匿名函数
```go
package main
import "fmt"

func main() {
    // 定义并立即调用
    result := func(a, b int) int {
        return a + b
    }(3, 5)

    fmt.Println(result) // 8
}
```

### 匿名函数赋值给变量
```go
package main
import "fmt"

func main() {
    add := func(a, b int) int {
        return a + b
    }

    fmt.Println(add(2, 4)) // 6
}
```

### 匿名函数作为参数
```go
package main
import "fmt"

func operate(a, b int, f func(int, int) int) int {
    return f(a, b)
}

func main() {
    res := operate(3, 4, func(x, y int) int {
        return x * y
    })
    fmt.Println(res) // 12
}
```

### 匿名函数与闭包
匿名函数可以捕获外部变量，形成闭包

```go
package main
import "fmt"

func main() {
    counter := 0

    inc := func() int {
        counter++
        return counter
    }

    fmt.Println(inc()) // 1
    fmt.Println(inc()) // 2
    fmt.Println(inc()) // 3
}
```

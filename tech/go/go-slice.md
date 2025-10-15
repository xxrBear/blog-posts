---
title: "Go 切片"
date: 2025-10-15T09:58:49+08:00
lastmod: 2025-10-15T09:58:49+08:00
author: 熊大如如
tags: # 标签
  - "go"
description: ""
weight:
slug: ""
summary: "Go 语言切片"
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
## 一、概念
Go 语言的切片是一种引用类型，它是数组的一个连续片段的引用。

那么为什么已经有了数组还需要切片呢？

首先，数组是固定长度的而切片是可变长度的，更灵活。

其次，拷贝数组的效率比切片高。

所以在日常开发中，使用切片的频率要远高于数组。

## 三、创建切片
### 直接申明
```go
var s []int
fmt.Println(s == nil) // true
```

### 字面量初始化
```go
s := []int{1, 2, 3}
fmt.Println(len(s), cap(s)) // 3 3
```

### 从数组引用
```go
arr := [5]int{1, 2, 3, 4, 5}
s := arr[1:4]
fmt.Println(s)
```

### make 创建
```go
s := make([]int, 3, 5)
fmt.Println(len(s), cap(s)) // 3 5
```

底层会分配一个长度为 `cap` 的数组，并返回前 `len` 个元素的切片

## 二、切片的常用操作
### 遍历切片
```go
var s = []int{1, 2, 3}

for i, e := range s {
    fmt.Println(i, e)
}
```

### 追加切片
如果想增加切片的容量，我们必须创建一个新的更大的切片并把原分片的内容都拷贝过来。下面的代码描述了从拷贝切片的 `copy` 函数和向切片追加新元素的 `append()` 函数

```go
package main
import "fmt"

func main() {
	slFrom := []int{1, 2, 3}
	slTo := make([]int, 10)

	n := copy(slTo, slFrom)
	fmt.Println(slTo)
	fmt.Printf("Copied %d elements\n", n) // n == 3

	sl3 := []int{1, 2, 3}
	sl3 = append(sl3, 4, 5, 6)
	fmt.Println(sl3)
}
```

### 清空切片
```go
s = s[:0] // 清空内容，但不释放底层数组
// 或者
s = nil   // 同时释放底层数组
```

## 四、切片易错知识点

### 索引越界
跟 Python 等语言一样，获取不存在的索引项，会出错

```go
s := []int{1, 2, 3}
_ = s[3] // panic: index out of range
```

### nil 与空切片区别
```go
var a []int       // nil
b := []int{}      // 空切片
fmt.Println(a == nil) // true
fmt.Println(b == nil) // false
```

### 切片共享机制！
```go
arr := [5]int{1, 2, 3, 4, 5}
s1 := arr[1:4]
s2 := s1[1:3]
s2[0] = 99
fmt.Println(arr) // [1 2 99 4 5]
```

修改 `s2` 的内容会影响 `s1` 和 `arr`

### append 函数重分配
`append` 触发重新分配，导致数据不再共享

```go
s1 := []int{1,2,3}
s2 := s1
s2 = append(s2, 4)
fmt.Println(s1) // [1 2 3]
fmt.Println(s2) // [1 2 3 4]
```

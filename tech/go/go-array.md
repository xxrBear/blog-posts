---
title: "Go 数组"
date: 2025-09-17T17:52:56+08:00
lastmod: 2025-09-17T17:52:56+08:00
author: 熊大如如
tags: # 标签
  - "go"
description: ""
weight:
slug: ""
summary: "Go 语言数组知识点"
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

## 简介
数组是具有相同唯一类型的一组已编号且长度固定的数据项序列

### 创建一个数组
```go
func main() {
    names := [5]string {"1", "2", "3", "4", "5"}
}
```

+ 使用 `:=`自动推导数组的类型

### 声明一个数组
```go
package main

import (
	"fmt"
)


func main() {
    var names  [5]string
	fmt.Println(names)
}
```

## 遍历数组
```go
func main() {
	var names [5]string

	for i, v := range names {
		fmt.Println(i, v)
	}
}
```

+ `for-range`循环遍历数组

## 数组传递给函数
大数组当作函数参数传递时，通常传递数组的指针或切片

```go
package main

import "fmt"

func main() {
	array := [3]float64{0.1, 0.2, 0.5}

	x := Sum(&array)
	fmt.Println(x)
}

func Sum(a *[3]float64) (sum float64) {

	for _, v := range a {
		sum += v
	}
	return
}
```


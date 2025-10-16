---
title: "Go 映射"
date: 2025-10-16T16:05:22+08:00
lastmod: 2025-10-16T16:05:22+08:00
author: 熊大如如
tags: # 标签
  - "go"
summary: "Go 语言映射"
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

## 一、简介
Go 语言内置 键值对 数据结构，它有如下特点

+ 无序，键值对的顺序是随机的
+ 键唯一，不可重复
+ 键必须是可比较类型
+ 值可以是任意类型
+ 查找、插入、删除的平均时间复杂度都是 O(1)

## 二、创建映射
### 使用字面量创建
```go
ages := map[string]int{
    "Alice": 25,
    "Bob":   30,
}
```

### 使用 make 创建空 map
```go
ages := make(map[string]int)
ages["Tom"] = 20
```

### 使用 var 声明
```go
var ages map[string]int
fmt.Println(ages == nil) // true
// ages["Tom"] = 20 // panic：不能往 nil map 写入
```

> 注意:只能对非 nil map 进行赋值
>
> 读取 nil map 是安全的
>

## 三、基本操作
### 插入或更新
```go
ages["Alice"] = 25
```

### 访问键值
```go
fmt.Println(ages["Alice"]) // 25
```

### 判断键是否存在
```go
age, ok := ages["Bob"]
if ok {
    fmt.Println("存在：", age)
} else {
    fmt.Println("不存在")
}
```

### 删除键
```go
delete(ages, "Alice")
```

### 遍历 map
```go
for key, value := range ages {
    fmt.Println(key, value)
}
```

## 四、零值行为
```go
var m map[string]int
fmt.Println(m == nil)     // true
fmt.Println(m["nothing"]) // 0（零值，不会 panic）
```

> 但不能往 nil map 写入值

## 五、值类型
值可以是任意类型，包括：

+ 基本类型
+ 结构体
+ 切片
+ map（嵌套 map）

### 嵌套 map 示例：
```go
students := map[string]map[string]int{
    "class1": {"Alice": 90, "Bob": 80},
    "class2": {"Tom": 95},
}
fmt.Println(students["class1"]["Alice"])
```

## 六、容量与扩容机制
虽然 `make(map[T]U, n)` 可以指定初始容量，但：

> 它只是提示容量，不是限制，也不是固定大小

当存入更多键值时，map 会自动扩容，底层重新分配桶

## 七、常见技巧与最佳实践
### 最佳创建方式
```go
m := make(map[string]int)
m["A"] = 1
```

- 错误写法

```go
var m map[string]int
m["A"] = 1 // panic: assignment to entry in nil map
```

解释：  
`var` 声明的 `map` 是 `nil`，必须先用 `make` 分配空间才能写入

### 预估容量提升性能
```go
m := make(map[string]int, 1000)
```

> Go 的第二个参数只是容量提示，不是固定大小但给出合理容量可以减少哈希桶扩容次数，显著提升性能

### 用字面量初始化更简洁
```go
m := map[string]int{
    "apple":  5,
    "banana": 7,
}
```

### 判断键是否存在的正确姿势
```go
if v, ok := m["apple"]; ok {
    fmt.Println("存在:", v)
} else {
    fmt.Println("不存在")
}
```

> 注意：直接判断 `if m["apple"] == 0` 不安全，因为零值和不存在是一样的

### 删除键时使用 `delete()`
```go
delete(m, "apple")
```

> 删除不存在的键不会报错，相当于安全操作

### 遍历时不要依赖顺序
```go
for k, v := range m {
    fmt.Println(k, v)
}
```

> Go 的 map 是无序的。如果需要排序输出

```go
keys := make([]string, 0, len(m))
for k := range m {
    keys = append(keys, k)
}
sort.Strings(keys)
for _, k := range keys {
    fmt.Println(k, m[k])
}
```

### 不能直接取元素地址
```go
// 编译错误
p := &m["key"] // cannot take the address of m["key"]
```

> 因为 map 扩容时键值位置会改变，地址不稳定

### 修改值时应先取出再写回
```go
v := m["count"]
v++
m["count"] = v
```

### 避免在循环中频繁创建 map
```go
// 不推荐
for i := 0; i < 1000; i++ {
    m := make(map[string]int)
    m["x"] = i
}
```

> 应该复用一个 map 或放循环外部

### 大量 map 查找性能建议
+ 尽量使用整型或短字符串作为 key；
+ 避免使用复杂结构体作为 key；
+ 如果 key 是 struct，确保字段可比较（不要包含 slice/map）


### 若要保留内存空间，可逐个删除
```go
for k := range m {
    delete(m, k)
}
```

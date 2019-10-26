---
layout:     post
title:      "Golang 数据结构--Slice"
subtitle:   "Golang 数据结构--Slice"
date:       2019-05-14 10:30:40
author:     "Mathew"
catalog: true
header-img: "img/post-bg-2017.jpg"
tags:
    - Golang
    - Slice
---

# Golang的 slice 数据结构

[TOC]

@(Golang笔记)

``` go
package main

import (
	"fmt"
)

func main() {
	a := []int{1, 2, 3, 4, 5, 6, 7, 8}
	b := []int{}
	fmt.Println(a)
	fmt.Println(b)
}
```

 当然 slice 里面也可以放其他的元素的
``` go
package main

import (
	"fmt"
)

func main() {
	a := []struct {
		name string
		age int
	}{
		{
			name: "Angela",
		},
		{
			name:"Mathew",
			age:28,
		},
		{

		},
	}
	fmt.Println(a[1])
}
```
还可以是空接口, 里面放置各种类型的
``` go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	a := []interface {
	}{
		struct {
			Name string
			Age int
		}{
			Name:"Mathew",
			Age:28,
		},
		func(a, b int) string{
			return strconv.Itoa(a + b)
		},
		"mathew",
	}
	fmt.Println(a[0])
}
```

## 操作Slice
### 访问


``` go
package main

import "fmt"

func main() {
	var c = 100
	var d = 20
	a := []interface{}{
		Person{
			name: "Mathew",
			age:  23,
		},
		Animal{},
		SumAndMulti(c, d),
	}

	fmt.Println(a[0].(Person).name)
	fmt.Println(a[2])
}

type Person struct {
	name string
	age  int
}
type Animal struct {
	Name string
	Age  int
}

func SumAndMulti(a, b int) (sum int) {
	sum = a + b
	return
}
```
### 截取

``` go
package main

import "fmt"

func main() {
	a := []int{1, 2, 3, 4, 5, 6, 7, 8, 9,}
	fmt.Println(a[3:])
	fmt.Println(a[0:])
	fmt.Println(a[:])
	fmt.Println(a[:4])
}
```


### 删除

``` go
package main

import "fmt"

func main() {
	a := []int{1, 2, 3, 4, 5, 6, 7, 8, 9,}
	var b []int
	var removeItem = 5
	for _, item := range a {
		if item != removeItem{
			b = append(b,item)
		}
	}
	fmt.Println(b)
}
```
从这里可以看到, golang 是一门相当原始的语言, 删除一个数组需要自己去写一个遍历, 而不是根据他的index提供删除的方法.
这个方法, 需要新建一个数组, 从某种程度上增加了空间	


### 添加
``` go
package main

import "fmt"

func main() {
	a := []int{1, 2, 3, 4, 5, 6, 7, 8, 9,}
	b := append(a, 21, 123, 453)
	fmt.Println(b)
}
```


### 更新元素
``` go
package main

import "fmt"

func main() {
	a := []int{1, 2, 3, 4, 5, 6, 7, 8, 9,}
	a[3] = 123
	fmt.Println(a)
}

```
slice更新一个元素, 可以直接


## 参考文献
1. [Go 语言数组和切片的原理](https://draveness.me/golang-array-and-slice)
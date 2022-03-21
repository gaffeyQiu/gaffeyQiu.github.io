---
title: "关于 go 语言中切片泄漏的例子"
date: 2022-03-21T18:48:25+08:00
tags: ["go", "slice"]
categories: ["go"]
author: "gaffey"
---
# Slice 使用误区
假设我们业务代码中用一个切片临时接收一个大数据，同时经过逻辑处理后使用 `[m:n]`  语法取出大切片的前10条数据供我们后续使用， 那么这个临时的大切片会被 GC 回收掉吗？  
示例代码  
```go
package main

import (
	"fmt"
	"runtime"
)

const (
	BIG_SLICE_LENGTH   = 10000000
	SMALL_SLICE_LENGTH = 10
)

func createBigSlice() []int32 {
	bigSlice := make([]int32, BIG_SLICE_LENGTH)
	for i := 0; i < BIG_SLICE_LENGTH; i++ {
		bigSlice[i] = int32(i)
	}
	fmt.Printf("big slice length %d, cap %d\n", len(bigSlice), cap(bigSlice))
	return bigSlice
}
func createSmallFromBig() []int32 {
	bigSlice := createBigSlice()
	smallSlice := bigSlice[:SMALL_SLICE_LENGTH]
	return smallSlice
}
func main() {
	small := createSmallFromBig()
	runtime.GC()
	fmt.Printf("big slice length %d, cap %d\n", len(small), cap(small))
}
```
输出结果是
```
big slice length 10000000, cap 10000000
big slice length 10, cap 10000000
```
可以看到，我们在函数内部临时创建的大切片，依然在主动 GC 之后，还存在。  
借助 debug 工具验证一下  
![](https://ypy.7bao.fun/img/202203211854882.png)  
首先可以看到我们确实取出了大切片的前10条数据，但是 cap 字段显示了，我们这个小切片的底层数组依然是旧的。
我们来验证一下，首先找到下标9的数据地址  
![](https://ypy.7bao.fun/img/202203211859285.png)  
然后在这个地址的基础上+4看看（一个int32占4字节）
![](https://ypy.7bao.fun/img/202203211900341.png)  
我们依然取出了大数组的下标10的数据  
所以，以后如果用 `[m:n]` 方法复用切片的时候要注意内存泄漏的问题。
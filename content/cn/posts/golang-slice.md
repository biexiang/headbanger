---
title: "关于Golang Slice，你知道的不多"
date: 2022-01-30T17:18:44+08:00
categories: ['Golang','Share']
draft: false
keywords: ['Golang','slice','源码阅读','深入解析Golang Slice底层实现','切片','高级用法']
description: "Golang slice切片的高级用法有哪些？使用上有什么坑？底层结构是什么样子？如何实现扩容？如何实现拷贝？什么时候使用数组，不使用切片？深入解析Golang Slice底层实现，源码阅读"
---

{{< music id="1386055858" >}}

## 前言
切片是Golang中的一等公民，业务代码中广泛使用，这里记录下以下几个问题：
* 切片的数据结构是什么？如何实现扩容？
* 切片有哪些少见的高级用法？
* 什么时候应该使用数组，而不是切片？

## 数据结构
编译期类型为 [Slice](https://github.com/golang/go/blob/3b2a578166bdedd94110698c971ba8990771eb89/src/cmd/compile/internal/types/type.go#L346)，运行期为 [SliceHeader](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/reflect/value.go#L1994)，具体结构如下：
```
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
```

### 初始化
#### 使用下标初始化
<script id="asciicast-466323" src="https://asciinema.org/a/466323.js" async></script>
./ssa.html 如下：
```
v11 (8) = Load <*[3]int> v10 v9 (&arr[*[3]int])
v25 (9) = Const64 <int> [1]
v26 (9) = Const64 <int> [3]
v27 (9) = SliceMake <[]int> v11 v25 v26
```
通过编译期的中间代码，`SliceMake` 主要涉及四个参数：元素类型、数组指针、切片大小和容量，这些运行期的 `SliceHeader` 字段定义相似。使用下标初始化会创建一个指向原数组的切片结构体，并不会拷贝原数据。

#### 通过字面量初始化
大部分工作都在编译器完成，比如 `[]int{1, 2, 3}`，底层会创建一个长度为3的数组，并且创建一个数组指针指向这块数组，通过`[:]`操作符初始化切片。

#### 关键字make创建切片
`make`函数除了类型，还接收len和cap两个参数，[类型检查期](https://github.com/golang/go/blob/da54dfb6a1f3bef827b9ec3780c98fde77a97d11/src/cmd/compile/internal/gc/typecheck.go#L326)会检查len和cap设置的合理性。  
中间代码生成的[walkexpr](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/cmd/compile/internal/gc/walk.go#L411)函数会根据切片的大小和是否发生逃逸（逃逸分析参考[escape-analysis]({{<ref "/posts/golang-escape-analysis">}})）决定初始化的方式：
* 当切片非常小且不会发生逃逸时，直接在编译阶段初始化数组，通过下标初始化切片，转成前面提到的`SliceMake`调用，这部分都是在栈上或者静态存储区完成
* 当切片非常大或者发生逃逸时，通过运行时[makeslice](https://github.com/golang/go/blob/3b2a578166bdedd94110698c971ba8990771eb89/src/runtime/slice.go#L83)在堆上初始化切片
```
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	mem, overflow := math.MulUintptr(et.size, uintptr(cap))
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		mem, overflow := math.MulUintptr(et.size, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen()
		}
		panicmakeslicecap()
	}
	return mallocgc(mem, et, true)
}
```

### 如何扩容
如下面代码，在对长度为0的切片进行append的时候，中间代码会调用[runtime.growslice](https://github.com/golang/go/blob/ac0ba6707c1655ea4316b41d06571a0303cc60eb/src/runtime/slice.go#L125)方法，下面分开来看这个扩容方法。
```
func newSlice() []int {
	slice := make([]int, 0)
	slice = append(slice,1)
	return slice
}

// v34 (9) = StaticCall <mem> {runtime.growslice} [64] v33
```

代码和计算新容量的公式如下：
* 若预期新CAP大于两倍的老CAP，则新的容量就是预期新CAP
* 若预期新CAP小于等于两倍的老CAP，并且老CAP小于1024，则新CAP等于两倍的老CAP（新CAP < 2048）
* 若预期新CAP小于等于两倍的老CAP，并且老CAP大于等于1024，则老CAP按照自身的25%逐步自增，知道大于预期新CAP，得到新CAP值
```
// It is passed the slice element type, the old slice, and the desired new minimum capacity
func growslice(et *_type, old slice, cap int) slice {
...
newcap := old.cap
doublecap := newcap + newcap
if cap > doublecap {
    newcap = cap
} else {
    if old.cap < 1024 {
        newcap = doublecap
    } else {
        // Check 0 < newcap to detect overflow
        // and prevent an infinite loop.
        for 0 < newcap && newcap < cap {
            newcap += newcap / 4
        }
        // Set newcap to the requested cap when
        // the newcap calculation overflowed.
        if newcap <= 0 {
            newcap = cap
        }
    }
}
```
有了前面计算的新CAP值后，还需要根据切片里元素的大小进行内存对齐，再调整下新CAP值，这部分实现可以看代码，这里不展开。
然后判断了是否溢出，需要分配的内存是否超出最大值，没问题的话就进行内存分配，并且`memmove`开始挪数据。

## 少见的高级用法
下面列出几个相对少见的用法。

### 三下标形式初始化
假设 baseContainer 是一个切片或者数组，则有下面两种初始化派生切片方式，派生切片的长度为 high - low、容量为 max - low ：
```
baseContainer[low : high : max] // 三下标形式
baseContainer[low : high]       // 双下标形式，等价于 baseContainer[low : high : cap(baseContainer)]
```
省略规则如下：
* 如果下标low为零，则它可被省略。此条规则同时适用于双下标形式和三下标形式
* 如果下标high等于len(baseContainer)，则它可被省略。此条规则同时只适用于双下标形式
* 三下标形式中的下标max在任何情况下都不可被省略

具体例子，[database/sql]({{<ref "/posts/golang-database-sql">}}) 里计算过期连接的切片使用如下：
```
closing = db.freeConn[:i:i]
db.freeConn = db.freeConn[i:]
```
`db.freeConn[:i:i]` 因为low下标为0可以被忽略，所以这里代表的是len=i且cap=i的派生切片，实际测试：
```
package main

import "log"

func main() {
	var a = [...]int{0, 1, 2, 3, 4, 5, 6, 7}
	a0 := a[:]
	a1 := a0[1:3]
	a2 := a0[1:5:6]
	log.Println(a0, len(a0), cap(a0))
	log.Println(a1, len(a1), cap(a1))
	log.Println(a2, len(a2), cap(a2))
}

// 运行结果
// 2022/02/03 14:11:21 [0 1 2 3 4 5 6 7] 8 8
// 2022/02/03 14:11:21 [1 2] 2 7
// 2022/02/03 14:11:21 [1 2 3 4] 4 5
```

### 切片拷贝的方式
可以直接使用`copy`函数，也可以通过`append`函数实现：
```
// 假设元素类型为T。
func Copy(dest, src []T) int {
	if len(dest) < len(src) {
		_ = append(dest[:0], src[:len(dest)]...)
		return len(dest)
	} else {
		_ = append(dest[:0], src...)
		return len(src)
	}
}
```

### 切片克隆的方式
* 通过自动扩容来克隆，缺点是自动扩容申请了额外可能不需要的空间
```
package main

import "log"

func main() {
	var a = [...]int{0, 1, 2, 3, 4, 5, 6, 7}
	a0 := a[:]
	log.Printf("%v , %p , %d , %d", a0, &a0, len(a0), cap(a0))
	aClone := append(a0[:0:0], a0...)
	log.Printf("%v , %p , %d , %d", aClone, &aClone, len(aClone), cap(aClone))
}

// 运行结果
// 2022/02/03 14:54:34 [0 1 2 3 4 5 6 7] , 0xc00000c0a0 , 8 , 8
// 2022/02/03 14:54:34 [0 1 2 3 4 5 6 7] , 0xc00000c0e0 , 8 , 8
```
* make + copy，最新版本存在优化 ✅
```
sClone := make([]T, len(s))
copy(sClone, s)
```

* make + append
```
sClone := append(make([]T, 0, len(s)), s...)
```

### 字符串展开成切片
据说是个语法糖，字符串的`...`展开。
```
package main

import "fmt"

func main() {
	hello := []byte("Hello ")
	world := "world!"

	helloWorld := append(hello, world...) // 语法糖
	fmt.Println(string(helloWorld))

	helloWorld2 := make([]byte, len(hello)+len(world))
	copy(helloWorld2, hello)
	copy(helloWorld2[len(hello):], world) // 语法糖
	fmt.Println(string(helloWorld2))
}
```

### 删除切片中的一个元素
* append，保证顺序
```
s = append(s[:i], s[i+1:]...)
```
* copy，保证顺序，`copy`返回最终拷贝了几个元素
```
s = s[:i + copy(s[i:], s[i+1:])]
```
* 删除最后一个元素，不保证顺序，[database/sql]({{<ref "/posts/golang-database-sql">}}) 就是这么用的
```
s[i] = s[len(s)-1]
s = s[:len(s)-1]
```

### 带条件删除切片中的元素
```
// 假设T是一个小尺寸类型。
func DeleteElements(s []T, keep func(T) bool, clear bool) []T {
	// result := make([]T, 0, len(s))
	result := s[:0] // 无须开辟内存
	for _, v := range s {
		if keep(v) {
			result = append(result, v)
		}
	}
	if clear { // 避免暂时性的内存泄露。
		temp := s[len(result):]
		for i := range temp {
			temp[i] = t0 // t0是类型T的零值
		}
	}
	return result
}
```

### 其他用法
* 因为`for-range`出来的val是拷贝出来的，如果切片元素比较大，可以减少拷贝，使用`for`循环通过索引去取

## 什么时候用数组而不是切片
数组相比切片有如下特点：
* 数组是可比较类型，可以进行 `a1 == a2` 的判断，而切片则不支持
* 编译安全，数组必须声明大小，即便是 `[...]array` 也会在编译器推断出大小，如果对于数组的使用越界了，编译期报错，而切片使用如果越界了，编译器没问题，只能到运行时panic了
* 数组的长度是类型声明的一部分，可以有语义的表达一些类型，比如 `type RGB [3]byte` 可以表达颜色
* 规划内存布局，简单来说，方便定义结构体的时候占坑

虽然业务代码里大量用切片，但是可以根据实际情况，使用数组可能会有更好的效果。


## Reference
* [切片](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-array-and-slice/)
* [数组、切片和映射](https://gfw.go101.org/article/container.html)
* [在 Go 中如何转储一个方法的 GOSSAFUNC 图](https://linux.cn/article-12350-1.html)
* [Go 数组比切片好在哪？](https://developer.51cto.com/article/661996.html)
* [Go 编译器原理](https://xiaomi-info.github.io/2019/11/13/golang-compiler-principle/)
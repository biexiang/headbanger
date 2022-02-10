---
title: "Golang 逃逸分析 - 我想要带你去海边"
date: 2022-01-30T17:30:52+08:00
categories: ['Golang','Share']
draft: false
keywords: ['Golang','escape analysis','源码阅读','逃逸分析','data-flow-analysis']
description: "什么是逃逸分析？栈和堆相比优势在哪？逃逸分析中的data-flow-analysis如何运行？哪些场景有逃逸分析？如何判断是否内联？"
---

{{< music id="1413863166" >}}

通过`GopherCon TW 2020`的分享来理解下逃逸分析。

{{< bilibili 544231107 >}}

## 什么是逃逸分析
C/C++没有垃圾回收机制，都是开发人员进行内存分配，要么分配到栈，要么分配到堆上，同时堆内存对象的生命周期管理给开发人员带来了心智负担，为了降低这方面的心智负担，有的编程语言比如Go、Java就支持了垃圾回收，当分配到堆上的对象不再被引用时，就会被回收。   
垃圾回收带来了便利，但是也带来了性能损耗，堆内存对象过多会给垃圾回收带来压力，所以需要尽量减少在堆上的内存分配。        
逃逸分析（escape analysis）就是在程序编译阶段根据程序代码中的数据流，对代码中哪些变量需要在栈上分配，哪些变量需要在堆上分配进行静态分析的方法。  

## 栈 vs 堆
Go中声明变量的具体分配策略如下：
* 堆上
    * 全局存储空间
    * 共享的存储对象
    * 被GC管理的存储对象
* 栈上
    * 函数内部的本地存储空间
    * 协程自己的栈帧
    * 私有的存储对象
    * 帧生命周期内的存储对象

从变量声明角度，堆和栈的差异：
* 在栈上分配的对象比在堆上快很多
* 协程可以完全控制自己的栈帧
* 栈上，没有锁，没有GC，开销少

下面通过Benchmark来证明栈的优势，具体代码参考[escape_analysis_test.go](https://github.com/biexiang/code-snippet/blob/main/escape_analysis/bench/escape_analysis_bench_test.go)：   
<script id="asciicast-467861" src="https://asciinema.org/a/467861.js" async></script>


## 逃逸分析如何运行
### 基本概念
* 逃逸分析在源码库中的代码：[escape.go](https://github.com/golang/go/blob/master/src/cmd/compile/internal/escape/escape.go)，编译器所处的阶段如下图所示：

[![HGc7QO.png](https://s4.ax1x.com/2022/02/09/HGc7QO.png)](https://imgtu.com/i/HGc7QO)
* 逃逸的概念
    * 逃逸会分析声明变量之间的赋值关系
    * 通常一个变量在以下情况下逃逸
        * 被`&`取地址
        * 至少一个相关的变量已经发生逃逸

### 如何判断释放逃逸
判断是否逃逸，通过 `data-flow-analysis` 或者其他的基本规则。
#### 数据流分析
数据流分析是个有向无环图，它被用来基于AST分析变量之间的转换关系，有以下概念：
* 点        
    * 展示所有被声明的变量
* 边 
    * 代表变量之间的赋值逻辑
    * 每条边都有权重，代表取地址/被引用的次数 

具体例子如下图所示：
[![HGIBFg.png](https://s4.ax1x.com/2022/02/09/HGIBFg.png)](https://imgtu.com/i/HGIBFg)

数据流分析的具体处理流程：
* 收集所有函数声明的变量，产出点    

[![HGTos1.png](https://s4.ax1x.com/2022/02/09/HGTos1.png)](https://imgtu.com/i/HGTos1)

* 收集所有变量的赋值逻辑，产出边

[![HGTjRH.png](https://s4.ax1x.com/2022/02/09/HGTjRH.png)](https://imgtu.com/i/HGTjRH)

* 开始分析和遍历
    * 从每个点开始分析
    * 如果关联的源数据点已经逃逸，并且关联的边路径权重为-1，则当前变量也标识为逃逸
    * 对于已经逃逸的变量，停止扩张分析

[![HG7SsI.png](https://s4.ax1x.com/2022/02/09/HG7SsI.png)](https://imgtu.com/i/HG7SsI)

[![HG7CeP.png](https://s4.ax1x.com/2022/02/09/HG7CeP.png)](https://imgtu.com/i/HG7CeP)

[![HG7kFS.png](https://s4.ax1x.com/2022/02/09/HG7kFS.png)](https://imgtu.com/i/HG7kFS)

[![HG7QoT.png](https://s4.ax1x.com/2022/02/09/HG7QoT.png)](https://imgtu.com/i/HG7QoT)

[![HG7wTK.png](https://s4.ax1x.com/2022/02/09/HG7wTK.png)](https://imgtu.com/i/HG7wTK)

* 遍历所有变量，收集已逃逸变量的逃逸原因

[![HG76ld.png](https://s4.ax1x.com/2022/02/09/HG76ld.png)](https://imgtu.com/i/HG76ld)

#### 其他基本规则
* 大对象
    * `var` or `:=` 直接声明的变量，大小如果超过10MB，则分配到堆上
    * 隐式声明，大小超过64KB，则分配到堆上，比如make指定CAP值
* 切片，若切片通过`make`关键字初始化，并且CAP值传的非常量，则分配到堆上，一定要常量！
* 映射，如果一个变量被`map`的key or value引用，则分配到堆上
* 函数入参，若函数入参泄露，则入参逃逸到堆上，Injecting changes to the passed parameters instead of return values back！
* 闭包外的变量，被闭包内变量取地址赋值使用，闭包内使用的参数一定要显示传进去!

测试代码地址：[escape_analysis.go](https://github.com/biexiang/code-snippet/blob/main/escape_analysis/test/escape_analysis.go)
{{< highlight go "linenos=table,linenostart=1" >}}
package main

type HugeExplicitT struct {
	a [3 * 1000 * 1000]int32 // 12MB
}

var vCapSize = 10

const cCapSize = 10

func f0() {
	// moved to heap: t1
	t1 := HugeExplicitT{}

	// make([]int32, 0, 17 * 1000) escapes to heap
	t2 := make([]int32, 0, 17*1000)

	// make([]int32, vCapSize, vCapSize) escapes to heap
	t3 := make([]int32, vCapSize, vCapSize)

	// make([]int32, cCapSize, cCapSize) does not escape
	t4 := make([]int32, cCapSize, cCapSize)

	// make([]int32, 10, 10) does not escape
	t5 := make([]int32, 10, 10)

	// make([]int32, vCapSize) escapes to heap
	t6 := make([]int32, vCapSize)

	//  make(map[*int]*int) does not escape
	t7 := make(map[*int]*int)
	// moved to heap: k1
	// moved to heap: v1
	k1, v1 := 0, 0
	t7[&k1] = &v1

	_, _, _, _, _, _ = t1, t2, t3, t4, t5, t6
}

func f1() map[string]int {
	//  make(map[string]int) escapes to heap
	v := make(map[string]int)
	return v
}

func f2() []int {
	// make([]int, 0, 10) escapes to heap
	v := make([]int, 0, 10)
	return v
}

func f3() *int {
	// moved to heap: t
	t := 0
	return &t
}

func f4(v1 *int) **int {
	// moved to heap: v1
	return &v1
}

func main() {
	// 大小超出、切片、映射
	f0()

	// 返回值情况
	_, _, _, _ = f1(), f2(), f3(), f4(&vCapSize)

	var x1 *int
	fn1 := func() {
		// moved to heap: y
		y := 1
		x1 = &y
	}
	fn1()

	var x2 *int
	fn2 := func(input *int) {
		// input does not escape
		y := 1
		input = &y
	}
	fn2(x2)
}
{{< / highlight >}}


<!-- ## 如何确定逃逸分析
`go build -gcflags "-m -l"` 传入-l是为了关闭inline，屏蔽掉inline对这个过程以及最终代码生成的影响。

### 哪些场景有逃逸分析

## 如何进行逃逸分析优化我们的项目

## 如何判断是否内联

### 什么情况下不应该内联 -->

## Reference
* [详解Go内联优化](https://segmentfault.com/a/1190000039146279)
* [Go: Inlining Strategy & Limitation](https://medium.com/a-journey-with-go/go-inlining-strategy-limitation-6b6d7fc3b1be)





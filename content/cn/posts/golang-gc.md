---
title: "Golang 垃圾回收"
date: 2022-02-23T21:44:07+08:00
categories: ['Golang','Share']
draft: false
keywords: ['Golang', '源码阅读']
description: "垃圾回收分为哪些，优缺点是什么？Golang垃圾回收的历史是什么？三色标记+混合写屏障的原理，为什么减少了STW的时间？"
---

{{< music id="521169592" >}}

## 垃圾回收历史
目前提供GC的语言有Python、PHP、Java、Javascript...，同时比如C、C++被设计为手动管理内存，但是也可以自行实现GC，还有Rust可以在编译器，依靠编译器插入清理代码的方式，精准清理。

### Golang GC Changelog
* Go 1：单线程版的标记清扫，全过程STW
* Go 1.3：标记阶段STW, 清扫阶段并行执行，停顿时间在约几百毫秒
* Go 1.5：三色标记，标记和清扫可以并行执行，但是标记阶段的前后需要STW来做GC的准备，停顿时间在一百毫秒以内
* ~~Go 1.6：使用 bitmap 来记录回收内存的位置，大幅优化垃圾回收器自身消耗的内存，停顿时间在十毫秒以内~~
* ~~Go 1.7：独立栈收缩，停顿时间控制在两毫秒以内~~
* Go 1.8：混合写屏障，停顿时间在半毫秒左右
* ~~Go 1.9：彻底移除了栈的重扫描过程~~
* ~~Go 1.12：整合了两个阶段的 Mark Termination~~
* ~~Go 1.13：着手解决向操作系统归还内存的，提出了新的 Scavenger~~
* ~~Go 1.14：替代了仅存活了一个版本的 scavenger，全新的页分配器，优化分配内存过程的速率与现有的扩展性问题，并引入了异步抢占，解决了由于密集循环导致的 STW 时间过长的问题~~

## GC 策略
### Golang使用的策略
{{< quote >}}
The GC runs concurrently with mutator threads, is type accurate (aka precise), allows multiple GC thread to run in parallel. It is a concurrent mark and sweep that uses a write barrier. It is non-generational and non-compacting. Allocation is done using size segregated per P allocation areas to minimize fragmentation while eliminating locks in the common case.
{{< /quote >}}

Go的GC使用的是一种非分代的没有整理过程的Concurrent Mark and Sweep算法（CMS算法），类似于Java的CMS GC。

通常，垃圾回收器的执行过程可根据代码的行为被划分为两个半独立的组件：赋值器（Mutator）和回收器（Collector）。赋值器一词最早由Dijkstra引入，意指用户态代码。因为对垃圾回收器而言，需要回收的内存是由用户态的代码产生的，用户态代码仅仅只是在修改对象之间的引用关系进行操作。回收器即为程序运行时负责执行垃圾回收的代码。

### 标记清扫
aka `Mark-Sweep`，Golang早期使用这种朴素的策略，策略分为标记追踪和清扫回收两个步骤，因为回收器执行时，并发执行的赋值器要挂起，出现STW。如果回收器和赋值器并发执行，就会产生程序的正确性问题：`如何保证回收器不会将存活的对象回收，同时赋值器访问到的对象都是重新整理和移动过的新对象`。

### 三色标记
从垃圾回收器的视角来看，三色抽象规定了三种不同类型的对象，并用不同的颜色相称：  
[![bno2AU.png](https://s4.ax1x.com/2022/02/27/bno2AU.png)](https://imgtu.com/i/bno2AU)    
* 白色对象：未被回收器访问到的对象。在回收开始阶段，所有对象均为白色，当回收结束后，白色对象均不可达
* 灰色对象：已被回收器访问到的对象，但回收器需要对其中的一个或多个指针进行扫描，因为他们可能还指向白色对象
* 黑色对象：已被回收器访问到的对象，其中所有字段都已被扫描，黑色对象中任何一个指针都不可能直接指向白色对象

当垃圾回收开始时，只有白色对象。随着标记过程开始进行时，灰色对象开始出现，这时候波面便开始扩大。当一个对象的所有子节点均完成扫描时，会被着色为黑色。当整个堆遍历完成时，只剩下黑色和白色对象，这时的黑色对象为可达对象，即存活；而白色对象为不可达对象，即死亡。这个过程可以视为以灰色对象为波面，将黑色对象和白色对象分离，使波面不断向前推进，直到所有可达的灰色对象都变为黑色对象为止的过程。

#### Go 1.5
该版本的处理流程如下：
* Sweep Termination: 收集根对象，清理上一轮未清扫完的span，启用写屏障和辅助GC，辅助GC将一定量的标记和清扫工作交给用户Goroutine来执行
* Mark: 扫描所有根对象和通过根对象可达的对象，并标记它们
* Mark Termination: 完成标记工作，重新扫描部分根对象(要求STW)，关闭写屏障和辅助GC
* Sweep: 按标记结果清理对象

同时因为并行标记清理产生了如下问题：
{{< quote >}}
假设某个灰色对象A指向白色对象B，而此时赋值器并发的将黑色对象C指向（ref3）了白色对象B，并将灰色对象A对白色对象B的引用移除（ref2），则在继续扫描的过程中，白色对象B永远不会被标记为黑色对象了（回收器不会重新扫描黑色对象），进而产生被错误回收的对象B。
{{< /quote >}}

可以证明，当以下两个条件同时满足时会破坏垃圾回收器的正确性：
* 条件1: 赋值器修改对象图，导致某一黑色对象引用白色对象
* 条件2: 从灰色对象出发，到达白色对象的、未经访问过的路径被赋值器破坏       

由此得出两个不变式：
* 强三色不变式：不允许黑色对象引用白色对象
* 弱三色不变式：黑色对象可以引用白色对象，但是白色对象的上游必须存在灰色对象

为了实现上面的不变式，引入了赋值器屏障，赋值器屏障作为一种同步机制，使赋值器在进行指针写操作时，能够`通`知回收器，进而不会破坏弱三色不变性，一共有两种赋值器屏障：
* Dijkstra 插入写屏障，如果某一对象的引用被插入到已经被标记为黑色的对象中（黑色不会重新扫描了），这类屏障会保守地将其作为非白色存活对象，以满足强三色不变性
* Yuasa 删除写屏障，在删除引用时，如果被删除引用的对象自身为灰色或者白色，那么被标记为灰色，满足弱三色不变式，灰色对象到白色对象的路径不会断      

插入写屏障和删除写屏障各有优缺点，插入写屏障在标记开始时无需STW，可直接开始，并发进行，但结束时需要STW来重新扫描栈，标记栈上引用的白色对象的存活；删除写屏障则需要在GC开始时STW扫描堆栈来记录初始快照，这个过程会记录开始时刻的所有存活对象，但结束时无需STW。

Go 1.5使用的Dijkstra 插入屏障实现。

#### Go 1.8
1.8版本引入混合写屏障机制，核心定义如下：
* GC刚开始的时候，将栈上的可达对象全部标为黑色
* GC期间，任何在栈上新创建的对象，均为黑色
* 堆上被删除的对象标记为灰色
* 堆上新添加的对象标记为灰色


### 其他策略
#### 引用计数
aka `Reference-Counting`，脚本语言常见，Python、PHP。常见问题是需要每个变量都需要额外空间存储引用次数，而且存在循环引用问题。比如PHP里，因为循环引用变量变成垃圾后，就只能等根缓存区存满了后触发GC或者运行时调用函数`gc_collect_cycles()`进行垃圾回收。
#### 分代回收
aka `Generational Collection`，Java和NodeJs V8在用的策略，核心是将对象分为新生代、老生代、永久代进行管理和收集，根据每个分代的特点采用不同的垃圾回收算法。因为Golang存在编译期的逃逸分析，大部分简单关系的新生对象都会分配到栈上，那些需要长期存在的对象才会被分配到需要垃圾回收的堆上，所以没有使用分代回收。
<!-- #### 标记压缩
AKA `Mark-Compact`，为什么不选择标记压缩
相比标记清扫，有一些成本，比如 压缩需要计算成本、实现复杂和CPU缓存等。 -->

## 调步算法
```
// src/runtime/debug
func SetGCPercent(percent int) int
```
默认情况下，`percent`等于100，含义比如：最后一次垃圾完成后正在使用中的堆内存是2MB。由于GC百分比设置为100％，因此下一次收集会在在增加2MB为4MB堆内存时启动。

调步算法理解需要结合具体测试用例和`go tool trace`等工具来分析，以后补充。

[![bGRjG8.png](https://s4.ax1x.com/2022/03/02/bGRjG8.png)](https://imgtu.com/i/bGRjG8)

BTW，`go tool trace` 工具确定可以，和Final Cut Pro X一样的手感，比如上图25%的P进行GC，也有MARK ASSIST。

## Reference
* [Go语言垃圾收集器的实现原理](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-garbage-collector/)
* [垃圾回收](https://golang.design/under-the-hood/zh-cn/part2runtime/ch08gc/basic/)
* [关于Golang GC的一些误解](https://studygolang.com/articles/31431)
* [一文弄懂 Golang GC、三色标记、混合写屏障机制](https://juejin.cn/post/7040737998014513183)
* [Go ballast](https://medium.com/@angus258963/go-memory-ballast-how-i-learnt-to-stop-worrying-and-love-the-heap-bc5eb2181f93)
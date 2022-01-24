---
title: "Golang Channel"
date: 2022-01-16T22:07:24+08:00
draft: true
categories: ['Share','Golang']
---

{{< quote >}}
Do not communicate by sharing memory; instead, share memory by communicating.
{{< /quote >}}

### 简单使用
channel的创建、发送、接受、关闭的使用如下：

<script id="asciicast-463838" src="https://asciinema.org/a/463838.js" async></script>

通过dlv debug进行disass可以知道主要调用下面几个方法，本次也主要分析这些方法的调用。
* runtime.makechan，创建一个channel
* runtime.chansend1，发送数据
* runtime.chanrecv1，接收数据
* runtime.closechan，关闭channel

### 数据结构
channel的数据结构[hchan](https://github.com/golang/go/blob/0d0193409492b96881be6407ad50123e3557fdfb/src/runtime/chan.go#L33)如下：

```
type hchan struct {
	qcount   uint               // chan队列长度
	dataqsiz uint               // chan队列容量
	buf      unsafe.Pointer     // 队列指针
	sendx    uint               // 发送的当前索引
	recvx    uint               // 接收的当前索引

	elemsize uint16             // 队列长度？
	elemtype *_type             // chan的数据类型

	recvq    waitq              // 接收协程队列
	sendq    waitq              // 发送协程队列

	closed   uint32             // 是否关闭
	lock mutex                  // 互斥锁
}
```
hchan里一共有三个队列，channel数据的缓存区是循环队列，sendq和recvq是链表队列，都是FIFO。

#### 创建
在makechan的[代码](https://github.com/golang/go/blob/0d0193409492b96881be6407ad50123e3557fdfb/src/runtime/chan.go#L72)中，分为检查和创建。
检查包括：
* 检查Element的结构体大小，需要小于 1<<16
* 检查hchan的内存对齐
* 计算Element结构体大小 * bufferSize 是否会溢出，并且获取需要分配的mem内存大小
创建包括：
* 如果是无缓存chan，mem为0，则直接分配一个hchanSize的大小
* 如果是有缓存且非指针类型的chan，则分配hchanSize+mem大小的内存，这里为连续内存(contiguous memory)
* 如果是有缓存且指针类型的chan，则先分配hchanSize自身内存，再单独分配mem的缓冲区内存

然后设置了elemsize、elemtype和dataqsiz。

`这里有个问题，为什么有缓存的指针类型不能分配连续内存呢？`

#### 关闭
在closechan的[代码](https://github.com/golang/go/blob/0d0193409492b96881be6407ad50123e3557fdfb/src/runtime/chan.go#L356)中，分为检查和关闭。
检查包括：
* chan是否为nil
* chan是否已经关闭
* 是否开启race
关闭包括：
* 释放接受协程队列
* 释放发送协程队列
* 唤醒所有等待的协程


### 发送数据
[chansend](https://github.com/golang/go/blob/0d0193409492b96881be6407ad50123e3557fdfb/src/runtime/chan.go#L159) 被调用有两种情况：
* 情况1：`c <- x`，block=true，强阻塞，要么等待，要么panic
* 情况2：如下所示，block=false，发送不了就 return false
```
select {
case c <- v:
	... foo
default:
	... bar
}
```


```
package main

import (
	"log"
	"time"
)

func main() {
	var ch chan int = nil

	go func() {
		time.Sleep(3 * time.Second)
	}()

	//ch <- 123

	select {
	case ch <- 123:
		log.Println(123)
	}
}

```


#### 直接发送
如果等待接收的队列有协程在等待，直接出队，将要发送的数据拷贝到等待协程的elem指针指向的地址上，然后将elem指针置空，唤醒该等待的协程。

```
	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

    ...

    func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
        ...
    	if sg.elem != nil {
            sendDirect(c.elemtype, sg, ep)
            sg.elem = nil
        }
        goready(gp, skip+1)
    }

    func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
        dst := sg.elem
        typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
        memmove(dst, src, t.size)
    }
```

#### 缓冲区
如果等待队列为空，缓冲区不为空，通过 `chanbuf` 函数计算当前sendx基于buf的具体内存地址，将发送指针指向的数据拷贝到 `chanbuf` 计算出来的内存地址上，其他的就是 RingBuffer 的内容了。

```
	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			racenotify(c, c.sendx, nil)
		}
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}
```

#### 阻塞发送
前面两种都没走进去，就只能暂存了，通过 `getg()` 获取当前协程指针，生成一个状态对象 `sudog`，记录 要发送的数据ep、协程指针gp、ch等，然后将 `sudog` 入队到sendq中，并且告诉调度器，当前协程要进入阻塞状态了。

```
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}

	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)

	atomic.Store8(&gp.parkingOnChan, 1)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)

	KeepAlive(ep)

	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	closed := !mysg.success
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg)
	if closed {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	return true
```

### 接收数据
[chanrecv](https://github.com/golang/go/blob/0d0193409492b96881be6407ad50123e3557fdfb/src/runtime/chan.go#L455) 被调用是三种情况：
* `x <- c`，对应函数 `chanrecv1`，仅返回数据，强阻塞(block=true)
* `x, ok := <- c`，对应函数 `chanrecv2`，返回数据和状态，强阻塞(block=true)
* 调用点如下代码，对应函数 `selectnbrecv`，返回数据和状态，强阻塞(block=false)
```
    select {
    case v, ok = <-c:
    	... foo
    default:
    	... bar
    }
```


下面具体分析 `chanrecv` 的内容。

#### 直接接收

#### 缓冲区

#### 阻塞接收


### Reference
* [channel底层的数据结构是什么](https://blog.frognew.com/2021/11/read-go-sources-channel-make.html)
* [Go 的并发：Channel 源码浅析](https://paulzhn.me/posts/go-channel.html)
* [Channel](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/#645-%E6%8E%A5%E6%94%B6%E6%95%B0%E6%8D%AE)
* [Delve调试器](https://chai2010.cn/advanced-go-programming-book/ch3-asm/ch3-09-debug.html)
* [Golang中MulUintptr实现原理](https://juejin.cn/post/6886045069149732877)
---
title: "Golang Channel"
date: 2022-01-16T22:07:24+08:00
draft: false
categories: ['Share','Golang']
---

{{< music id="29431219" >}}

{{< quote >}}
Do not communicate by sharing memory; instead, share memory by communicating.
{{< /quote >}}

### 简单使用
channel的创建、发送、接收、关闭的简单使用如下：

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

	elemsize uint16             // elem结构的大小，用于计算缓冲区所需的大小
	elemtype *_type             // chan的数据类型

	recvq    waitq              // 接收协程队列
	sendq    waitq              // 发送协程队列

	closed   uint32             // 是否关闭
	lock mutex                  // 互斥锁
}
```
hchan里一共有三个队列，channel数据的缓冲区是循环队列，sendq和recvq是链表队列，都是FIFO。

#### 创建
在makechan的[代码](https://github.com/golang/go/blob/0d0193409492b96881be6407ad50123e3557fdfb/src/runtime/chan.go#L72)中，分为检查和创建。


检查包括：
* 检查Element的大小，需要小于 1<<16
* 检查hchan的内存对齐
* 计算Element的大小 * bufferSize 是否会溢出，并且获取需要分配的mem内存大小

```
	var c *hchan
	switch {
	case mem == 0:
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}
```
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

关闭包括：
* 释放接收协程队列
* 释放发送协程队列
* 唤醒所有等待的协程


### 发送数据
[chansend](https://github.com/golang/go/blob/0d0193409492b96881be6407ad50123e3557fdfb/src/runtime/chan.go#L159) 被调用有两种情况：
* 情况一：`c <- x`，block=true，对应函数 `chansend1`

如果同时chan是nil，往nil chan写东西，会调用`gopark`方法将当前协程休眠，一直阻塞下去。下面示例代码会阻塞3秒后，因为没有其他alive的协程了而报错退出（fatal error: all goroutines are asleep - deadlock!）。
```
package main

import (
	"time"
)

func main() {
	var ch chan<- struct{} = nil
	go func() {
		time.Sleep(time.Second * 3)
	}()
	ch <- struct{}{}
}
```

* 情况二：`select { case c <- v: ... }`，block=false，对应函数 `selectnbsend`

差不多的示例代码，在select case下就不会阻塞，因为block=false会直接返回。
```
package main

import (
	"time"
)

func main() {
	var ch chan<- struct{} = nil
	go func() {
		time.Sleep(time.Second * 3)
	}()
	select {
	case ch <- struct{}{}:
	default:
	}
}
```

#### 直接发送
如果等待接收的队列里有协程在等待，直接出队，将要发送的数据拷贝到等待协程的elem指针指向的地址上，然后将elem指针置空，唤醒该等待的协程。

```
if sg := c.recvq.dequeue(); sg != nil {
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
如果等待队列为空，缓冲区不为空，通过 `chanbuf` 函数计算当前sendx基于buf的具体内存地址，将发送指针指向的数据拷贝到 `chanbuf` 计算出来的内存地址上，其他的就是 [RingBuffer]({{<ref "/posts/ring-buffer">}}) 的内容了。

```
if c.qcount < c.dataqsiz {
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
* `x <- c`，对应函数 `chanrecv1`，仅返回数据
* `x, ok := <- c`，对应函数 `chanrecv2`，返回数据和状态
* `select { case v, ok = <-c:`，对应函数 `selectnbrecv`，返回数据和状态

和发送数据类似，`chanrecv1` 和 `chanrecv2` 对于nil channel会强阻塞，调用`gopark`休眠当前协程，select case则直接返回。
```
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    ...
    if c == nil {
        if !block {
            return
        }
        gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }
    ...
}
```

下面具体分析 `chanrecv` 的内容。

#### 直接接收
如果发送队列里有`sudog`，调用recv进行传输，这里分为两种情况：
* 情况一：channel是无缓存队列，直接将`sudog`中的element挪到待接收的地址ep
* 情况二：channel是有缓存队列，首先拿recvx计算当前接收的缓存队列地址qp，将qp地址的数据挪到待接收地址ep，再将`sudog`里的element数据挪到qp地址，再将recvx自增1。

然后将`sudog`的elem置为nil，唤醒该`sudog`的协程。

`这里我没搞懂为啥 c.sendx = c.recvx？`

```
if sg := c.sendq.dequeue(); sg != nil {
    recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
    return true, true
}

func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if c.dataqsiz == 0 {
		if raceenabled {
			racesync(c, sg)
		}
		if ep != nil {
			// copy data from sender
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
		// Queue is full. Take the item at the
		// head of the queue. Make the sender enqueue
		// its item at the tail of the queue. Since the
		// queue is full, those are both the same slot.
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			racenotify(c, c.recvx, nil)
			racenotify(c, c.recvx, sg)
		}
		// copy data from queue to receiver
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		// copy data from sender to queue
		typedmemmove(c.elemtype, qp, sg.elem)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
	}
	sg.elem = nil
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	sg.success = true
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)
}

```

#### 缓冲区
和发送类似，调用`chanbuf`计算当前接收索引qp（读），将当前接收索引qp的数据拷贝到ep指针，并且清空当前qp指针的数据，其他的也类似 [RingBuffer]({{<ref "/posts/ring-buffer">}})。

```
if c.qcount > 0 {
    // Receive directly from queue
    qp := chanbuf(c, c.recvx)
    if raceenabled {
        racenotify(c, c.recvx, nil)
    }
    if ep != nil {
        typedmemmove(c.elemtype, ep, qp)
    }
    typedmemclr(c.elemtype, qp)
    c.recvx++
    if c.recvx == c.dataqsiz {
        c.recvx = 0
    }
    c.qcount--
    unlock(&c.lock)
    return true, true
}
```

#### 阻塞接收
和发送类似，获取当前协程指针和`sudog`，给`sudog`赋值当前接收的ep地址、当前协程地址以及chan，然后将`sudog`入队到recv q中，调用gopark休眠当前协程。

```
gp := getg()
mysg := acquireSudog()
mysg.releasetime = 0
if t0 != 0 {
    mysg.releasetime = -1
}

mysg.elem = ep
mysg.waitlink = nil
gp.waiting = mysg
mysg.g = gp
mysg.isSelect = false
mysg.c = c
gp.param = nil
c.recvq.enqueue(mysg)

atomic.Store8(&gp.parkingOnChan, 1)
gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

// someone woke us up
if mysg != gp.waiting {
    throw("G waiting list is corrupted")
}
gp.waiting = nil
gp.activeStackChans = false
if mysg.releasetime > 0 {
    blockevent(mysg.releasetime-t0, 2)
}
success := mysg.success
gp.param = nil
mysg.c = nil
releaseSudog(mysg)
return true, success
```


### 总结
待解决前面几个问题：
* [为什么有缓存的指针类型不能分配连续内存呢？]({{<ref "/posts/golang-channel#创建">}})
* [为啥c.sendx = c.recvx？]({{<ref "/posts/golang-channel#直接接收">}})


### Reference
* [channel底层的数据结构是什么](https://blog.frognew.com/2021/11/read-go-sources-channel-make.html)
* [Go 的并发：Channel 源码浅析](https://paulzhn.me/posts/go-channel.html)
* [Golang-gopark函数和goready函数原理分析](https://blog.csdn.net/u010853261/article/details/85887948)
* [Delve调试器](https://chai2010.cn/advanced-go-programming-book/ch3-asm/ch3-09-debug.html)
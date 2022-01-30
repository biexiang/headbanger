---
title: "Ring Buffer 介绍"
date: 2022-01-16T21:58:21+08:00
draft: false
categories: ['Algorithm','Leetcode']
keywords: ['Golang','LeetCode','力扣','算法',"Ring Buffer","环形队列"]
description: "什么是环形队列？怎么用Golang实现一个环形队列？什么是无锁的环形队列？Golang Channel和环形队列有什么关系？"
---

{{< music id="455420533" >}}

### 起源
看Go Channel源码的时候，都有提到 `Ring Buffer`，同时力扣上也刷到了类似的题目 [622. 设计循环队列](https://leetcode-cn.com/problems/design-circular-queue/)，这里记录下 Ring Buffer 的调查结果，[ Golang Channel ]({{<ref "/posts/golang-channel">}}) 也用到了循环队列。


### 实现
下面分为普通不加锁的循环队列和无锁的循环队列，代码实现和TestCase都在仓库 [ring_buffer](https://github.com/biexiang/code-snippet/tree/main/ring_buffer)。

#### 不加锁的循环队列
循环队列类似 `Producer-Consumer` 模式，Tail指针的移动产生数据，Head指针的移动消费数据。
循环队列可以使用取余来获取新的索引，但CPU做位运算性能更高，所以改成Capacity为2的幂次，这样通过和Mask做与操作实现Turn Arround。

```
package no_lock

// no_lock指不用锁

type RingBuffer struct {
	Queue      []interface{}
	Head, Tail uint64
	Cap, Mask  uint64
}

func findPowerOfTwo(givenMum uint64) uint64 {
	givenMum--
	givenMum |= givenMum >> 1
	givenMum |= givenMum >> 2
	givenMum |= givenMum >> 4
	givenMum |= givenMum >> 8
	givenMum |= givenMum >> 16
	givenMum |= givenMum >> 32
	givenMum++
	return givenMum
}

func Constructor(k int) RingBuffer {
	capacity := findPowerOfTwo(uint64(k))
	return RingBuffer{
		Queue: make([]interface{}, capacity),
		Head:  uint64(0),
		Tail:  uint64(0),
		Cap:   capacity,
		Mask:  capacity - 1,
	}
}

func (c *RingBuffer) EnQueue(value interface{}) bool {
	if c.IsFull() {
		return false
	}
	newTail := (c.Tail + 1) & c.Mask
	c.Tail = newTail
	c.Queue[newTail] = value
	return true
}

func (c *RingBuffer) DeQueue() (value interface{}, success bool) {
	if c.IsEmpty() {
		return nil, false
	}
	newHead := (c.Head + 1) & c.Mask
	c.Head = newHead
	return c.Queue[newHead], true
}

func (c *RingBuffer) IsEmpty() bool {
	return c.Head == c.Tail
}

func (c *RingBuffer) IsFull() bool {
	return c.Tail-c.Head == c.Cap-1 || c.Head-c.Tail == 1
}

```

#### 无锁的循环队列
lock-free，是一种底层通过CPU提供的硬件同步原语CAS指令（`compare and swap` / `compare and set`）来实现的乐观并发控制算法，因为原子指令本身就是带锁的操作，只是锁的粒度比较小，所以虽然没有显式的调用锁，但是实现了锁的功能。

Golang里可以通过atomic包来进行CAS原子操作，循环队列里设计到并发读写的就是head、tail和queue这三个参数的操作，这里全部通过atomic包进行读写。这里涉及的unsafe包的使用，可以参考 [ Golang Unsafe Pointer ]({{<ref "/posts/golang-unsafe-pointer">}})。

```
package lock_free_1

import (
	"sync/atomic"
	"unsafe"
)

type RingBuffer struct {
	queue      []interface{}
	head, tail uint64
	cap, mask  uint64
}

func findPowerOfTwo(givenMum uint64) uint64 {
	givenMum--
	givenMum |= givenMum >> 1
	givenMum |= givenMum >> 2
	givenMum |= givenMum >> 4
	givenMum |= givenMum >> 8
	givenMum |= givenMum >> 16
	givenMum |= givenMum >> 32
	givenMum++
	return givenMum
}

func Constructor(k int) RingBuffer {
	capacity := findPowerOfTwo(uint64(k))
	return RingBuffer{
		queue: make([]interface{}, capacity),
		head:  uint64(0),
		tail:  uint64(0),
		cap:   capacity,
		mask:  capacity - 1,
	}
}

func (c *RingBuffer) EnQueue(value interface{}) bool {
	// EnQueue only 非nil的值
	if value == nil {
		return false
	}

	oldHead := atomic.LoadUint64(&c.head)
	oldTail := atomic.LoadUint64(&c.tail)
	if IsFull(oldHead, oldTail, c.cap) {
		return false
	}

	newTail := (oldTail + 1) & c.mask
	// 判断newTailData是否为nil
	if newTailData := atomic.LoadPointer((*unsafe.Pointer)(unsafe.Pointer(&c.queue[newTail]))); newTailData != nil {
		return false
	}

	if !atomic.CompareAndSwapUint64(&c.tail, oldTail, newTail) {
		return false
	}

	atomic.StorePointer((*unsafe.Pointer)(unsafe.Pointer(&c.queue[newTail])), unsafe.Pointer(&value))
	return true
}

func (c *RingBuffer) DeQueue() (value interface{}, success bool) {
	oldHead := atomic.LoadUint64(&c.head)
	oldTail := atomic.LoadUint64(&c.tail)
	if IsEmpty(oldHead, oldTail) {
		return nil, false
	}

	newHead := (oldHead + 1) & c.mask
	headData := atomic.LoadPointer((*unsafe.Pointer)(unsafe.Pointer(&c.queue[newHead])))
	if headData == nil {
		return nil, false
	}

	if !atomic.CompareAndSwapUint64(&c.head, oldHead, newHead) {
		return nil, false
	}

	// 原数据置为nil
	atomic.StorePointer((*unsafe.Pointer)(unsafe.Pointer(&c.queue[newHead])), nil)
	return *(*interface{})(headData), true
}

func IsEmpty(head, tail uint64) bool {
	return head == tail
}

func IsFull(head, tail, cap uint64) bool {
	return tail-head == cap-1 || head-tail == 1
}

```

CAS的 `ABA问题`，以下面的代码举例，`Producer0`在CAS判断之前已经拿到了Tail值，这时候被让出了调度权，`Producer非0` 开始不断的 `EnQueue`，直到环形队列Turn Arround，碰巧`Producer0`重新开始调度时Tail值又回到了之前的值，`Producer0`进行CAS操作成功，以为什么都没发生，然后set了值，这样`Producer非0`设置的值就被覆盖了。

维基百科的例子如下：
{{< quote >}}
你拿着一个装满钱的手提箱在飞机场，此时过来了一个火辣性感的美女，然后她很暖昧地挑逗着你，并趁你不注意的时候，把用一个一模一样的手提箱和你那装满钱的箱子调了个包，然后就离开了，你看到你的手提箱还在那，于是就提着手提箱去赶飞机去了。
{{< /quote >}}

这里解决办法是修改 Tail和Head的赋值逻辑，改成自增，而不是和mask进行与操作，因为字段类型是 `uint64`，所以前面的ABA问题，`Producer0` 让出调度权后，队列要 `EnQueue` 1<<64 - 1 次，可能性极低。具体代码参考 [仓库](https://github.com/biexiang/code-snippet/blob/main/ring_buffer/lock_free_2/lock_free_2.go)。


### Reference
* [无锁队列并非真的无锁](https://www.fournoas.com/posts/lock-free-queue-is-not-lock-free/)
* [无锁队列的实现](https://coolshell.cn/articles/8239.html)
* [一个简单的 Lock Free Ring Buffer，有多简单？](https://lenshood.github.io/2021/04/19/lock-free-ring-buffer/)

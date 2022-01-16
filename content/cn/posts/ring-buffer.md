---
title: "Ring Buffer 介绍"
date: 2022-01-16T21:58:21+08:00
draft: true
categories: ['Algorithm','Leetcode']
---

### 起源
看Go Channel源码的时候，都有提到 `Ring Buffer`，同时力扣上也刷到了类似的题目 [622. 设计循环队列](https://leetcode-cn.com/problems/design-circular-queue/)，这里记录下 Ring Buffer 的调查结果，同时挖个[ Golang Channel ]({{<ref "/posts/golang-channel">}})的坑。


### 实现
以下分为普通的循环队列和并发安全的循环队列。

#### 普通的循环队列
循环队列类似 `Producer-Consumer` 模式，Tail指针的移动产生数据，Head指针的移动消费数据。同时和Channel相似的是关键的函数签名，DeQueue类似 `val, ok <- ch`，EnQueue类似 `ch <- val`，只不过下面的RingBuffer实现还不会阻塞等待。
原本循环队列使用取余来获取新的索引，因为CPU做位运算性能更高，所以改成Capacity为2的幂次，这样通过和Mask做与操作可以实现Turn Arround。

```
package ringBuffer

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

func (c *RingBuffer) Front() interface{} {
	if c.IsEmpty() {
		return nil
	}
	return c.Queue[c.Head]
}

func (c *RingBuffer) Rear() interface{} {
	if c.IsEmpty() {
		return nil
	}
	return c.Queue[c.Tail]
}

func (c *RingBuffer) IsEmpty() bool {
	return c.Head == c.Tail
}

func (c *RingBuffer) IsFull() bool {
	return c.Tail-c.Head == c.Cap-1 || c.Head-c.Tail == 1
}
```

#### 并发安全的循环队列
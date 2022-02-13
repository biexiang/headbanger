---
title: "关于LRU和LFU算法"
date: 2022-01-09T14:37:13+08:00
draft: false
categories: ['Algorithm','Leetcode']
keywords: ['Golang','LeetCode 146','LeetCode 460','力扣','LRU','LFU','算法']
description: "什么是LRU算法？什么是LFU算法？他们俩有什么区别？"
---

### 源头
力扣有这两个题，[146. LRU 缓存](https://leetcode-cn.com/problems/lru-cache/) 和 [460. LFU 缓存](https://leetcode-cn.com/problems/lfu-cache/submissions/)，记录下两者的区别和实现。


### 区别
首先各自的概念，LRU是最近不使用的淘汰，LFU是最近用的频率少的淘汰。然后下面举个栗子，Capacity=3:
```
[1,2,1,2,1,2,1,2,3,4]
```
如果是LRU，插入4时淘汰的是1，1是最近不使用的。
如果是LFU，插入4时淘汰的是3，最小使用频次是1，频次1里面只有一个3，所以淘汰3。


### 实现
#### LRU
因为要求GET和PUT的时间复杂度都是O(1)，所以GET查询肯定得用map，因为栈和队列没法查询中间元素，单链表删除需要遍历找到前驱节点，为了删除时能在O(1)完成，使用双向链表。

下面的Golang实现里，双向链表用的官方的标准库 `container/list`，标准库的Value是个 interface{} 类型，断言会有性能消耗，如果真有需求自己实现LRU，可以自己实现双向链表省去断言的消耗。

```
package cache

import (
	"container/list"
	"log"
)

type Pair struct {
	Key, Value int
}

type LRUCache struct {
	Cap int
	L   *list.List
	M   map[int]*list.Element
}

func Constructor(capacity int) LRUCache {
	return LRUCache{
		L:   list.New(),
		M:   make(map[int]*list.Element, capacity),
		Cap: capacity,
	}
}

func (c *LRUCache) Get(key int) int {
	if c.M[key] == nil {
		return -1
	}
	ele := c.M[key]
	c.L.MoveToFront(ele)
	return ele.Value.(Pair).Value
}

func (c *LRUCache) Put(key int, value int) {
	if ele, ok := c.M[key]; ok {
		ele.Value = Pair{key, value}
		c.L.MoveToFront(ele)
	} else {
		ele := c.L.PushFront(Pair{
			key, value,
		})
		c.M[key] = ele
	}
	if c.L.Len() > c.Cap {
		if ele := c.L.Back(); ele != nil {
			if kv, ok := ele.Value.(Pair); ok {
				c.L.Remove(ele)
				delete(c.M, kv.Key)
			}
		}
	}
}

```

#### LFU
LFU相比LRU多了频次的逻辑。当容量满了的时候，如下：
* Key的使用频次都不一样，淘汰频次最低的
* Key的频次都一样，或者有两个以上的Key的频次都是最低的，需要按照LRU的处理办法，淘汰最近没有在用的

需要注意的是，Put新的Key时，要先 check capacity。如果先添加，后 check capacity，当容量满了，且热Key的频次都是大于1的，那么添加的新Key会在check时被删掉。

```
package cache

import (
	"container/list"
	"log"
)

type Pair struct {
	Key, Value int
	Frequency  int
}

type LFUCache struct {
	Cap  int
	Min  int
	MinL map[int]*list.List
	EleM map[int]*list.Element
}

func Constructor(capacity int) LFUCache {
	return LFUCache{
		Cap:  capacity,
		Min:  -1,
		MinL: make(map[int]*list.List),
		EleM: make(map[int]*list.Element),
	}
}

func (c *LFUCache) Get(key int) int {
	var (
		ele         *list.Element
		currentPair *Pair
	)

	ele = c.EleM[key]
	if ele == nil {
		return -1
	}

	currentPair = ele.Value.(*Pair)
	c.MinL[currentPair.Frequency].Remove(ele)
	currentPair.Frequency++
	c.EleM[key] = c.getMinList(currentPair.Frequency).PushFront(currentPair)

	if c.Min == currentPair.Frequency-1 && c.MinL[currentPair.Frequency-1].Len() == 0 {
		c.Min++
	}

	return currentPair.Value
}

func (c *LFUCache) getMinList(minIndex int) *list.List {
	if c.MinL[minIndex] == nil {
		c.MinL[minIndex] = list.New()
	}
	return c.MinL[minIndex]
}

func (c *LFUCache) Put(key int, value int) {
	if c.Cap == 0 {
		return
	}

	if ele, ok := c.EleM[key]; ok {
		pair := ele.Value.(*Pair)
		pair.Value = value
		c.Get(key)
	} else {
		if len(c.EleM) == c.Cap {
			if ele := c.MinL[c.Min].Back(); ele != nil {
				c.MinL[c.Min].Remove(ele)
				delete(c.EleM, ele.Value.(*Pair).Key)
			}
		}
		ele := c.getMinList(1).PushFront(&Pair{
			Key:       key,
			Value:     value,
			Frequency: 1,
		})
		c.EleM[key] = ele
		c.Min = 1
	}
}

```
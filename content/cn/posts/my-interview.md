---
title: "My Interview"
date: 2022-04-19T21:20:29+08:00
draft: false
hide: true
categories: ['Algorithm','Leetcode']
---

## 面试
* 2022-04-20 11:00 百度-个人云 * 2
* 2022-04-25 14:00 boss直聘
* 2022-04-26 11:00 探探
* 2022-04-27 14:30 Shopee * 2
* 2022-04-28 16:00 字节跳动-番茄小说
* 2022-05-09 全天面试 趣加
* 2022-05-09 19:00 商汤
* 2022-05-10 17:00 一点资讯

### 百度-个人云
* 链表排序，要求归并非递归版本实现
* GC算法

### boss直聘
* [HTTP 1.0、1.1、2.0、3.0区别](https://www.jianshu.com/p/cd70b8e90d00)
* docker单机 服务端的最大连接数
* tcp 为啥需要time_wait，以及丢失最后一次ack后，怎么处理
* grpc的4种实现方式：[既然有 HTTP 请求，为什么还要用 RPC 调用？](https://www.zhihu.com/question/41609070)
* map什么时候扩容，为什么会出现稀疏桶，syncMap和自己加锁实现有啥区别: [Go 1.9 sync.Map揭秘](https://colobu.com/2017/07/11/dive-into-sync-Map/)
* rr 如何解决幻读问题: [Innodb 中 RR 隔离级别能否防止幻读?](https://github.com/Yhzhtk/note/issues/42)
* [五分钟搞懂MySQL索引下推](https://www.cnblogs.com/three-fighter/p/15246577.html)
* 为啥Go 1.14解决了抢占式调度: [go1.14基于信号的抢占式调度实现原理](https://xiaorui.cc/archives/6535)
* 切片问题

{{< highlight go "linenos=table,linenostart=1" >}}
package main

import (
	"fmt"
	"sync"
)

func m1() {
	var a []int
	wg := sync.WaitGroup{}
	for i := 0; i < 10000; i++ {
		wg.Add(1)
		go func(i int) {
			a = append(a, i)
			wg.Done()
		}(i)
	}
	wg.Wait()
	fmt.Println(len(a))
}

func m2() {
	var a []int
	wg := sync.WaitGroup{}
	for i := 0; i < 10000; i++ {
		wg.Add(1)
		go func(i int, b []int) {
			b = append(b, i)
			wg.Done()
		}(i, a)
	}
	wg.Wait()
	fmt.Println(len(a))
}

func main() {
	m1()
	m2()
}

{{< / highlight >}}


### 探探
* 对于用户态和内核态的理解: [从根上理解用户态与内核态](https://segmentfault.com/a/1190000039774784)
* 什么时候发生上下文切换: [什么是上下文切换](https://luffy997.github.io/2021/07/19/%E4%BB%80%E4%B9%88%E6%98%AF%E4%B8%8A%E4%B8%8B%E6%96%87%E5%88%87%E6%8D%A2/#%E4%B8%8A%E4%B8%8B%E6%96%87)
* 进程和线程都是如何通信的: [进程间通信和线程间通信的几种方式](https://www.cnblogs.com/fanguangdexiaoyuer/p/10834737.html)
* 进程和线程自有的资源有哪些: [线程间到底共享了哪些进程资源](https://cloud.tencent.com/developer/article/1768025)
* 僵尸进程和孤儿进程的区别是啥: [僵尸进程和孤儿进程有了解过吗](https://xie.infoq.cn/article/3a980c8f6a5a0a7a26cc3d2e8)
* 对于fork的了解，以及父子进程如何通信和共享: [进程间通信](https://akaedu.github.io/book/ch30s04.html)
* 一致性hash，负载均衡算法了解哪些: [memcached一致性hash算法原理 ](https://www.cnblogs.com/hjwublog/p/5625275.html)
* 减少time_wait有哪些方法
* 关系型数据库和非关系型数据库的理解
* [myisam和innodb的区别](https://www.zhihu.com/question/20596402)
* 对于事务的理解
* 事务的acid属性分别是啥
* [B树、B+树、LSM树以及其典型应用场景](https://blog.csdn.net/u010853261/article/details/78217823)
* [WAL理解](https://www.cnblogs.com/xuwc/p/14037750.html)
* [TCP和Udp的区别是什么？](https://www.zhihu.com/question/47378601/answer/276353285)
* k个一组链表反转

{{< highlight go "linenos=table,linenostart=1" >}}
package main

import "log"

type ListNode struct {
	Val  int
	Next *ListNode
}

func _reverse(head, tail *ListNode) (*ListNode, *ListNode) {
	prev := tail.Next
	p := head
	for prev != tail {
		next := p.Next
		p.Next = prev
		prev = p
		p = next
	}
	return tail, head
}

func reverseKGroup(head *ListNode, k int) *ListNode {
	var dummyNode = &ListNode{Next: head}
	prev := dummyNode
	for head != nil {
		tail := prev
		for i := 0; i < k; i++ {
			tail = tail.Next
			if tail.Next == nil {
				return dummyNode.Next
			}
		}
		next := tail.Next
		head, tail = _reverse(head, tail)
		prev.Next = head
		tail.Next = next
		prev = tail
		head = tail.Next
	}
	return dummyNode.Next
}

func _createTestNodes(idx int, nodes []int) *ListNode {
	if idx >= len(nodes) {
		return nil
	}
	node := &ListNode{
		Val:  nodes[idx],
		Next: _createTestNodes(idx+1, nodes),
	}
	return node
}

func main() {
	topNodes := _createTestNodes(0, []int{1, 2, 3, 4, 5})
	o := reverseKGroup(topNodes, 2)
	log.Println(o)
}
{{< / highlight >}}



### 虾皮
* 数组和链表的区别 
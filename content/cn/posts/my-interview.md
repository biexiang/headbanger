---
title: "My Interview"
date: 2022-04-19T21:20:29+08:00
draft: false
categories: ['Algorithm','Leetcode']
---

## 面试
* 2022-04-20 11:00 百度-个人云
* 2022-04-25 14:00 boss直聘
* 2022-04-26 11:00 探探
* 2022-04-27 14:30 Shopee
* 2022-04-28 16:00 字节跳动-番茄小说

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
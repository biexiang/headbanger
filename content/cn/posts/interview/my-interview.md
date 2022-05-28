---
title: "面试常见问题"
date: 2022-04-17T22:54:18+08:00
categories: ['Interview']
draft: false
hide: true
---

{{% mdlist file="/layouts/partials/interview_list.md" %}}

### MySQL
* [MySQL八股文连环45问（背诵版）](https://zhuanlan.zhihu.com/p/403656116)
* [MyISAM与InnoDB 数据结构的区别](https://zhuanlan.zhihu.com/p/343746709)
* [聚簇索引和辅助索引](https://zhuanlan.zhihu.com/p/371167166)
* [Mysql存储引擎--MyISAM与InnoDB的底层数据结构](https://www.cnblogs.com/zhangdanyang95/p/11384785.html)
* [Mysql中数据类型括号中的数字代表的含义](https://www.cnblogs.com/loren-yang/p/7512258.html)
* [MySQL 性能优化神器 Explain 使用分析](https://segmentfault.com/a/1190000008131735)
* [布隆过滤器/HyperLogLog/位图](https://hogwartsrico.github.io/2020/06/08/BloomFilter-HyperLogLog-BitMap/index.html)
* [为什么 MySQL 使用 B+ 树](https://draveness.me/whys-the-design-mysql-b-plus-tree/)
* [什么是 MySQL 的全局锁、表锁、行锁](https://segmentfault.com/a/1190000039848201)
* [MySQL各种“Buffer”之Change Buffer](https://www.modb.pro/db/112469)
* [MySQL事务隔离级别和实现原理](https://zhuanlan.zhihu.com/p/117476959)
* [五分钟搞懂MySQL索引下推](https://www.cnblogs.com/three-fighter/p/15246577.html)
* [Mysql - 范围查询过程分析底层的锁实现机制](https://blog.csdn.net/it_lihongmin/article/details/115337587)
* [平衡二叉树、B树、B+树、B*树 理解其中一种你就都明白了](https://zhuanlan.zhihu.com/p/27700617)
* [数据库IO性能，及InnoDB与MyISAM引擎对比](https://blog.csdn.net/weixin_38744051/article/details/86466908)
* [Innodb 中 RR 隔离级别能否防止幻读?](https://github.com/Yhzhtk/note/issues/42)
* [分库分表常见玩法及跨库查询/事务等问题](https://www.jianshu.com/p/6f5662908dae)
* [关于电商平台订单分库分表那些事](https://www.cnblogs.com/ZJOE80/p/15763170.html)
* [B树、B+树、LSM树以及其典型应用场景](https://blog.csdn.net/u010853261/article/details/78217823)


### Redis
* [Redis，八股文！](https://jishuin.proginn.com/p/763bfbd66f14)
* [Redis 通过巧妙地使用数据结构节省内存空间](https://blog.csdn.net/qq_39751320/article/details/108862584)
* [redis的事务和watch](https://www.jianshu.com/p/361cb9cd13d5)
* [Redis 管道、事务、Lua 脚本对比](https://blog.csdn.net/qq_35787138/article/details/113741467)
* [如何应对缓存问题](https://gongfukangee.github.io/2019/04/02/Cache/#%E7%BC%93%E5%AD%98%E7%A9%BF%E9%80%8F)
* [《面试八股文》之 Redis 16卷](https://juejin.cn/post/6989153296808149029)
* [5个基础数据结构的实现原理]()
* [为什么 Redis 选择单线程模型](https://draveness.me/whys-the-design-redis-single-thread/)
* [为什么Redis要比Memcached更火？](https://cloud.tencent.com/developer/article/1697819)
* [Redis为什么用跳表而不用平衡树？](https://zhuanlan.zhihu.com/p/23370124)、[redis为什么选择了跳跃表而不是红黑树](https://blog.csdn.net/qq9808/article/details/104865385)
* [说一说Redis事务是否满足ACID以及WATCH监视命令的作用](https://blog.csdn.net/qq_39794062/article/details/120426301)
* [Redis中的hash扩容渐进式rehash过程](https://zhuanlan.zhihu.com/p/400625895)

### Nginx
* [都是事件驱动，为什么Nginx的性能远高于Redis？](https://www.taohui.tech/2020/12/14/nginx/%E9%83%BD%E6%98%AF%E4%BA%8B%E4%BB%B6%E9%A9%B1%E5%8A%A8%EF%BC%8C%E4%B8%BA%E4%BB%80%E4%B9%88nginx%E7%9A%84%E6%80%A7%E8%83%BD%E8%BF%9C%E9%AB%98%E4%BA%8Eredis%EF%BC%9F/)
* [深入剖析Nginx负载均衡算法](https://www.taohui.tech/2021/02/08/nginx/%E6%B7%B1%E5%85%A5%E5%89%96%E6%9E%90Nginx%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1%E7%AE%97%E6%B3%95/)

### ElasticSearch
* [倒排/全文索引](https://zhuanlan.zhihu.com/p/33671444)
* [跳表(SkipList)原理篇](https://www.cnblogs.com/Laymen/p/14084664.html)

### Kafka
* [《面试八股文》之kafka21卷](https://juejin.cn/post/6982851330234646565)
* [怎么理解 Kafka 消费者与消费组之间的关系?](https://segmentfault.com/a/1190000039125247)
* [图解 ZooKeeper 的选举机制](https://segmentfault.com/a/1190000039385874)
* [原来 8 张图，就可以搞懂「零拷贝」了](https://www.cnblogs.com/xiaolincoding/p/13719610.html)
* [kafka的副本同步机制 HW和LEO](https://www.cfanz.cn/mobile/resource/detail/lBAxElXWnQjmO)
* [消息队列面试热点：如何保证消息的顺序性？](https://juejin.cn/post/6844904000098140173)
* [kafka如何解决重复消费问题](https://blog.51cto.com/u_15281317/3007783)
* [kafka新增topic对应的partition数如何决策](https://juejin.cn/post/6988344277654847501)
* [图解削峰限流技术，RabbitMq 消息队列解决高并发，高并发下削峰限流技术，主流消息队列对比](https://blog.csdn.net/penggerhe/article/details/108404243)、[消息队列 解耦、异步、削峰、限流](https://blog.csdn.net/qq_36390914/article/details/108359791)、[削峰填谷，你只知道消息队列？](https://developer.51cto.com/article/676653.html?edm)

### Go
* GC算法
* [详解 Go 程序的启动流程，你知道 g0，m0 是什么吗？](https://segmentfault.com/a/1190000040181868)
* [Golang为什么需要内存逃逸，如果全部逃逸，或者全部不逃逸会怎么样](https://segmentfault.com/a/1190000040450335)
* [go怎么做内存分配]()
* [k8s相关原理]()
* [go1.14基于信号的抢占式调度实现原理](https://xiaorui.cc/archives/6535)
* [GPM原理](https://learnku.com/articles/41728)
* [深度解密Go语言之context](https://zhuanlan.zhihu.com/p/68792989)
* 服务发现 etcd consul zk
* raft paxos 一致性选举
* [23道 K8S 面试题](https://www.modb.pro/db/99717)
* [面试必备 (背)--Go 语言八股文系列！](https://xie.infoq.cn/article/ac87ac5f9e8def9f91b817bf9)
* [你说说互斥锁、自旋锁、读写锁、悲观锁、乐观锁的应用场景](https://cloud.tencent.com/developer/article/1700079)
* [协程究竟比线程能省多少开销？](https://segmentfault.com/a/1190000037676163)
* [Go 1.9 sync.Map揭秘](https://colobu.com/2017/07/11/dive-into-sync-Map/)
* 切片扩容

{{< highlight go "linenos=table,linenostart=1" >}}
package main

import "fmt"

func main() {
	s := []int{2, 3, 5, 7, 11, 13}
	printSlice(s) // len=6 cap=6 [2, 3, 5, 7, 11, 13]

	// Slice the slice to give it zero length.
	s = s[:0]
	printSlice(s) // len=0 cap=6 []

	// Extend its length.
	s = s[:4]
	printSlice(s) // len=4 cap=6 [2,3,5,7]

	// Drop its first two values.
	s = s[2:]
	printSlice(s) // len=2 cap=4 [5,7]

	ss := []int{2, 3, 5, 7, 11, 13}
	s = ss[2:3]
	printSlice(s) // len=1 cap=4 [5]
	s[0] = 20
	printSlice(ss) //  len=6 cap=6 [2,3,20,7,11,13]
	s = append(s, 15, 17, 19)
	printSlice(s)  // len=4 cap=4 [20,15,17,19]
	printSlice(ss) // len=6 cap=6 [2,3,20,15,17,19]
}

func printSlice(s []int) {
	fmt.Printf("len=%d cap=%d %v\n", len(s), cap(s), s)
}
{{< / highlight >}}


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

### 网络
* [说说UDP和TCP的区别及应用场景](https://segmentfault.com/a/1190000021815671)
* [哪5种IO模型？什么是select/poll/epoll？同步异步阻塞非阻塞有啥区别](https://www.cnblogs.com/yangjianyong-bky/articles/14608585.html)
* [彻底搞懂HTTPS的加密原理](https://zhuanlan.zhihu.com/p/43789231)
* [HTTP/2.0 原理！与 1.x 相比，到底优化了什么？](https://blog.csdn.net/plokmju88/article/details/118688462)
* [HTTP 1.0、1.1、2.0、3.0区别](https://www.jianshu.com/p/cd70b8e90d00)
* [既然有 HTTP 请求，为什么还要用 RPC 调用？](https://www.zhihu.com/question/41609070)

### 微服务
* [微服务注册中心原理，看这篇就够了！](https://www.cnblogs.com/haha12/p/11532910.html)
* [Kubernetes (K8S) 3 小时快速上手 + 实践，无废话纯干货](https://www.bilibili.com/video/BV1Tg411P7EB?p=7&spm_id_from=444.41.top_right_bar_window_history.content.click)
* [微服务下如何保证事务的一致性](https://zhuanlan.zhihu.com/p/130416752)


### 开放题
* [面试官问：如何设计一个高并发系统？](https://blog.51cto.com/u_15420855/4713743)
* [关于电商平台订单分库分表那些事](https://www.cnblogs.com/ZJOE80/p/15763170.html)
* ES+HBase的查询方案
* [系统设计 | 榜单系统](https://zhuanlan.zhihu.com/p/477724438)
* [别瞎搞了！微博、知乎就是这么设计Feed流系统的~](https://jishuin.proginn.com/p/763bfbd5468b)
* [微博feed流方案](https://utf8.hk/archives/php_redis_weibo_feed.html)
* 系统设计，自行数据结构设计一个排名系统，支持查询top 1000和自己的排名

### 其他
* [C语言 strlen(str)和sizeof(arr)的区别](https://blog.csdn.net/wang123456___/article/details/114736344)
* [“字节序”是个什么鬼？](https://zhuanlan.zhihu.com/p/21388517)
* [面试官：什么是死锁？怎么排查死锁？怎么避免死锁？](https://segmentfault.com/a/1190000039753805?utm_source=sf-similar-article)
* [操作系统调度如何实现？](https://www.zhihu.com/question/25527028)
* [高并发系统限流-漏桶算法和令牌桶算法](https://www.cnblogs.com/xuwc/p/9123078.html)
* [僵尸进程和孤儿进程有了解过吗](https://xie.infoq.cn/article/3a980c8f6a5a0a7a26cc3d2e8)
* [进程间通信](https://akaedu.github.io/book/ch30s04.html)
* [memcached一致性hash算法原理 ](https://www.cnblogs.com/hjwublog/p/5625275.html)
* [线程间到底共享了哪些进程资源](https://cloud.tencent.com/developer/article/1768025)
* [什么是上下文切换](https://luffy997.github.io/2021/07/19/%E4%BB%80%E4%B9%88%E6%98%AF%E4%B8%8A%E4%B8%8B%E6%96%87%E5%88%87%E6%8D%A2/#%E4%B8%8A%E4%B8%8B%E6%96%87)
* [从根上理解用户态与内核态](https://segmentfault.com/a/1190000039774784)
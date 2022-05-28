---
title: "【面试系列】CAP理论"
date: 2022-05-28T16:51:57+08:00
draft: false
hide: true
categories: ['Interview']
keywords: ['Interview', '面试']
description: ""
---

{{% mdlist file="/layouts/partials/interview_list.md" %}}


### 什么是CAP理论
Consistency、Availability、Partition tolerance，aka CAP ... 
一般分布式系统会聊到这个理论，因为部署在不同的机器上，所以分区容错上是必须保证的，在出现分区错误时，决定了C和A只能选择其中一个，即 CA or CP ...

* [一文看懂｜分布式系统之CAP理论](https://cloud.tencent.com/developer/article/1860632)
* [分布式系统的“脑裂”到底是个什么玩意？](https://segmentfault.com/a/1190000040420527)

### MySQL的CAP应用
* [CAP定理的平衡：从MySQL主从到集群](https://juejin.cn/post/6895761455358935053)

### NOSQL的CAP应用
* [redis的三种模式以及涉及到的CAP理论](https://blog.csdn.net/weixin_56503821/article/details/119573689)
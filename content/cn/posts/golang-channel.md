---
title: "Golang Channel"
date: 2022-01-16T22:07:24+08:00
draft: true
categories: ['Share','Golang']
asciinema: true
---

{{< quote >}}
Do not communicate by sharing memory; instead, share memory by communicating.
{{< /quote >}}

### 简单使用
<script id="asciicast-IbIPgjjcTd4JXenekt2RujWvR" src="https://asciinema.org/a/IbIPgjjcTd4JXenekt2RujWvR.js" async></script>

### 数据结构
[hchan](https://github.com/golang/go/blob/0d0193409492b96881be6407ad50123e3557fdfb/src/runtime/chan.go#L33)


#### 创建

#### 关闭

### 发送数据

#### 直接发送

#### 缓冲区

#### 阻塞发送


### 接受数据

#### 直接接收

#### 缓冲区

#### 阻塞接收

### Reference
* [channel底层的数据结构是什么](https://blog.frognew.com/2021/11/read-go-sources-channel-make.html)
* [Go 的并发：Channel 源码浅析](https://paulzhn.me/posts/go-channel.html)
* [Channel](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/#645-%E6%8E%A5%E6%94%B6%E6%95%B0%E6%8D%AE)
* [Delve调试器](https://chai2010.cn/advanced-go-programming-book/ch3-asm/ch3-09-debug.html)

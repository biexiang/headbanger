---
title: "Goroutine泄漏"
date: 2022-01-23T16:00:23+08:00
draft: false
categories: ['Golang','Investigation']
keywords: ['Golang','协程']
description: "如何定位协程泄漏"
---

{{< music id="211059" >}}

### 问题发现

[![pF0Ev5V.png](https://s11.ax1x.com/2024/03/01/pF0Ev5V.png)](https://imgse.com/i/pF0Ev5V)

压测期间，观察服务Grafana监控面板，发现某些机器内存使用突增，于是上机器进行pprof采样查看内存使用：
```
go tool pprof -inuse_space 127.0.0.1:8080/debug/pprof/heap
go tool pprof -alloc_space 127.0.0.1:8080/debug/pprof/heap
```

交互式命令行操作，通过top10，没有查找到与监控匹配的内存使用，实际inuse_space在1GB左右，但是RSS在4GB-5GB：

[![pF0EzCT.png](https://s11.ax1x.com/2024/03/01/pF0EzCT.png)](https://imgse.com/i/pF0EzCT)

查找一番资料发现，pprof是采样会有误差，而且pprof抓的是堆内存，不包括栈的，有可能没有被GC触发，也有可能还未scavenging归还给操作系统，可以通过获取memstats进行具体计算：
```
curl 127.0.0.1:8080/debug/pprof/heap?debug=1
HeapInuse // 堆上使用内存的大小
HeapIdle - HeapReleased // 可以归还但是没有归还的内存
Stack // 栈上
```
因为Stack占用的较多，所以担心会有协程泄漏，但是代码之前review过没有找到相关问题。

### 问题解决
同样是通过pprof，在非压测期间上机器采样：
```
go tool pprof 127.0.0.1:8080/debug/pprof/goroutine
```
[![pF0VS8U.png](https://s11.ax1x.com/2024/03/01/pF0VS8U.png)](https://imgse.com/i/pF0VS8U)

用traces从采样中可以发现头两个runtime.gopark的协程数量很多，同样去其他机器采样发现，没有这两个17w+的runtime.gopark，基本就可以确定是这里出的问题，结合后面的协程信息，发现是协程消费端逻辑问题导致
```
runtime.gopark
runtime.chansend
runtime.chansend1
git.garena.com/*/rate_limit/distribute/v2.(*Limiter).batchProcess.func2
```
子chansend阻塞导致整体消费端runtime.chanrecv阻塞，最终就是相关key的并发请求进来都会因为buffer chan满了，而被gopark起来...
暂时解决办法将源头的chan，改成长度为1的buffer chan。


### Reference
* [一次线上内存使用率异常问题排查](https://fanlv.fun/2022/06/02/golang-pprof-mem/#2-3-memory-scavenging)

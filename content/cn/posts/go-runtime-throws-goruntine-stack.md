---
title: "Go Runtime Throws Gorountine Stack Dumps"
date: 2023-02-20T21:18:07+08:00
categories: ['Golang','Troubleshooting']
keywords: ['Golang','Go Runtime Throws Gorountine Stack Dumps']
description: "Go Runtime Throws Gorountine Stack Dumps"
---

{{< music id="1878812258" >}}

同事抛来一个错误日志，让我通过日志定位问题，日志关键内容如下：

```
STDOUT detail: application crashes/exits early: application exits with code 2 ....blahblah...
STDOUT goroutine 2070 [IO wait]:
    ....blahblah...

STDOUT goroutine 2098 [chan receive]:
    ....blahblah...

STDOUT goroutine 1771 [chan receive]:
    ....blahblah...
```

以前也在线上环境看到过类似的日志，当时就是觉得和pprof采集的goroutine堆栈信息比较像，但是单看他发的这几个goroutine的堆栈代码，代码逻辑没有问题，于是开始怀疑他给的日志不全，主动上平台查看日志发现完整的日志文件被截断了，查看完整的文件发现打印了一堆协程的堆栈，其中第一行打印了具体的原因：

```
fatal error: concurrent map read and map write

goroutine 391 [running]:
runtime.throw({0x330ef38?, 0x2c82480?})
	/usr/local/go/src/runtime/panic.go:992 +0x71 fp=0xc003371488 sp=0xc003371458 pc=0x436491
runtime.mapaccess2_faststr(0x0?, 0x0?, {0xc0026e8497, 0x28})
	/usr/local/go/src/runtime/map_faststr.go:117 +0x3d4 fp=0xc0033714f0 sp=0xc003371488 pc=0x4133b4
git.blahblah.com/blahblah/internal/clients/rate_limit/distribute/v2.findLimiter({0xc0026e8497, 0x28})
```
继而查看业务代码，确实是在并发操作map时，可能存在写map的同时，有协程对map进行读取，导致fatal error，关于这个问题，之前在  [Golang Map 源码分析]({{<ref "/posts/golang-map#map为什么不安全以及syncmap为什么安全">}}) 中提到过。

那这些goroutine的日志是谁打印的呢，以上面的错误为例，查看源码，是runtime throw出来的：

```
throw("concurrent map read and map write")
```

[pkg.go.dev/runtime](https://pkg.go.dev/runtime) 有说：

```
The GOTRACEBACK variable controls the amount of output generated 
when a Go program fails due to an unrecovered panic or an unexpected runtime condition. 
By default, a failure prints a stack trace for the current goroutine, 
eliding functions internal to the run-time system, and then exits with exit code 2. 
The failure prints stack traces for all goroutines if there is no current goroutine or the failure is internal to the run-time. 
```

同时然后找到一个issue，[runtime: some runtime throws print all goroutine stack dumps by default](https://github.com/golang/go/issues/46995)，大概就是说panic会打印问题goroutine的堆栈信息，但是throw会把所有goroutine都打印出来方便对问题进行调查，但是在线上流量大的环境下，如果代码存在问题，会频繁dump出大量日志出来，不太友好，对于不熟悉的同学也不好定位问题... 官方未回复
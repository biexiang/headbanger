---
title: "Golang PProf工具使用"
date: 2023-02-27T17:02:32+08:00
categories: ['Golang','Troubleshooting']
keywords: ['Golang','pprof']
description: "通过Golang PProf工具定位问题、优化系统"
---

{{< music id="422428548" >}}

## 背景
工作中经常会遇到一些常见的Go问题，比如锁竞争、内存占用过高、CPU 占用过高、协程泄露、阻塞操作、GC频繁内存回收等，我们可以通过官方的PProf工具进行迅速排查定位问题，最近工作的2家公司线上服务也都集成了PProf采集和分析的WEB UI工具，所以这是一个非常实在的功能。

## 使用
一般的基本引入包自动init，并且开启http服务就会默认注册debug路由，如下所示：

```
import	_ "net/http/pprof" //nolint
// ...
_ = http.ListenAndServe(":8080", nil)
```

常用的性能分析指标有：

|  类型   | 描述  |
|  ----  | ----  |
| profile  | CPU占用情况 |
| heap  | 堆上内存的使用情况 |
| goroutine  | 当前所有协程的情况 |
| allocs  | 内存分配情况 |
| blocks  | 阻塞操作情况 |
| trace  | 程序运行跟踪情况 |


对于在线采集分析可以用下面的命令：

```
// cpu
go tool pprof  http://localhost:8080/debug/pprof/profile

// 堆内存
go tool pprof  http://localhost:8080/debug/pprof/heap

// 协程
go tool pprof  http://localhost:8080/debug/pprof/goroutine

// etc ...
```

命令行模式有top、list、traces、web这几个命令，traces和web都是在浏览器上分析，命令行可以通过top查看占用排行，用list找出具体哪一行代码造成的占用：

```
➜  play: go tool pprof http://127.0.0.1:8080/debug/pprof/goroutine
Fetching profile over HTTP from http://127.0.0.1:8080/debug/pprof/goroutine
Saved profile in /Users/ileopold/pprof/pprof.goroutine.001.pb.gz
Type: goroutine
Time: Feb 27, 2023 at 2:34pm (CST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 62, 100% of 62 total
Showing top 10 nodes out of 93
      flat  flat%   sum%        cum   cum%
        60 96.77% 96.77%         60 96.77%  runtime.gopark
         1  1.61% 98.39%          1  1.61%  runtime.goroutineProfileWithLabels
         1  1.61%   100%          1  1.61%  runtime.sigNoteSleep
         0     0%   100%          5  8.06%  bufio.(*Reader).Read
         0     0%   100%          1  1.61%  bufio.(*Reader).ReadLine
         0     0%   100%          1  1.61%  bufio.(*Reader).ReadSlice
         0     0%   100%          1  1.61%  bufio.(*Reader).fill
         0     0%   100%          4  6.45%  bufio.(*Scanner).Scan
         0     0%   100%          1  1.61%  database/sql.(*DB).connectionOpener
(pprof) list runtime.gopark
Total: 62
ROUTINE ======================== runtime.gopark in /usr/local/go/src/runtime/proc.go
        60         60 (flat, cum) 96.77% of Total
         .          .    358:	gp.waitreason = reason
         .          .    359:	mp.waittraceev = traceEv
         .          .    360:	mp.waittraceskip = traceskip
         .          .    361:	releasem(mp)
         .          .    362:	// can't do anything that might move the G between Ms here.
        60         60    363:	mcall(park_m)
         .          .    364:}
         .          .    365:
         .          .    366:// Puts the current goroutine into a waiting state and unlocks the lock.
         .          .    367:// The goroutine can be made runnable again by calling goready(gp).
         .          .    368:func goparkunlock(lock *mutex, reason waitReason, traceEv byte, traceskip int) {
ROUTINE ======================== runtime.goparkunlock in /usr/local/go/src/runtime/proc.go
         0          2 (flat, cum)  3.23% of Total
         .          .    364:}
         .          .    365:
         .          .    366:// Puts the current goroutine into a waiting state and unlocks the lock.
         .          .    367:// The goroutine can be made runnable again by calling goready(gp).
         .          .    368:func goparkunlock(lock *mutex, reason waitReason, traceEv byte, traceskip int) {
         .          2    369:	gopark(parkunlock_c, unsafe.Pointer(lock), reason, traceEv, traceskip)
         .          .    370:}
         .          .    371:
         .          .    372:func goready(gp *g, traceskip int) {
         .          .    373:	systemstack(func() {
         .          .    374:		ready(gp, traceskip, true)
(pprof)
```

通过这种手段，只需要在问题发生期间抓包，就能线下分析立即找出问题代码的位置，并且上线修复。

如果看不出来问题，还可以抓取正常时期的包以及异常时期的包，通过 -base 指定基准来确定问题代码。

## Reference
[性能调试利器使用](http://liuqh.icu/2021/11/27/go/package/31-trace/)
[Go 大杀器之跟踪剖析 trace](https://eddycjy.gitbook.io/golang/di-9-ke-gong-ju/go-tool-trace)
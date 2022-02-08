---
title: "Golang Escape Analysis"
date: 2022-01-30T17:30:52+08:00
categories: ['Golang','Share']
draft: true
keywords: ['Golang','escape analysis','源码阅读','逃逸分析']
description: "Golang"
---

## 什么是逃逸分析
C/C++没有垃圾回收机制，都是开发人员进行内存分配，要么分配到栈，要么分配到堆上。同时堆内存对象的生命周期管理给开发人员带来了心智负担，为了降低这方面的心智负担，编程语言支持了垃圾回收，当分配到堆上的对象不再有引用时，就会被回收。显然垃圾回收带来了便利，但是也带来了性能损耗，堆内存对象过多会给垃圾回收带来压力，所以需要尽量减少在堆上的内存分配。        
逃逸分析（escape analysis）就是在程序编译阶段根据程序代码中的数据流，对代码中哪些变量需要在栈上分配，哪些变量需要在堆上分配进行静态分析的方法。     

## 如何确定逃逸分析
`go build -gcflags "-m -l"` 传入-l是为了关闭inline，屏蔽掉inline对这个过程以及最终代码生成的影响。

### 哪些场景有逃逸分析

## 如何判断是否内联

### 什么情况下不应该内联


https://github.com/golang/go/blob/master/src/cmd/compile/internal/escape/escape.go

内联优化



https://www.bilibili.com/video/BV1Qi4y1K7ep?from=search&seid=15361282791032949971&spm_id_from=333.337.0.0
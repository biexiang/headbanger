---
title: "Golang Escape Analysis"
date: 2022-01-30T17:30:52+08:00
categories: ['Golang','Share']
draft: true
keywords: ['Golang','escape analysis','源码阅读','逃逸分析']
description: "Golang"
---

## 什么是逃逸分析

## 如何判断是否内联

## 如何确定逃逸分析

## 哪些场景有逃逸分析



https://github.com/golang/go/blob/master/src/cmd/compile/internal/escape/escape.go

分配到堆上 还是 栈上

内联优化
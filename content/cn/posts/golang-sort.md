---
title: "Golang Sort 源码阅读"
date: 2022-03-05T18:59:21+08:00
categories: ['Golang','Share']
draft: true
keywords: ['Golang','sort','源码阅读']
description: "Golang Sort包都用到了哪些排序算法？什么情况下用Sort函数？什么情况下用Stable函数？数据结构中排序为什么要考虑稳定性？"
---

## 复习知识
通过题[912. 排序数组](https://leetcode-cn.com/problems/sort-an-array/)，用常见排序算法复习下。
### 快速排序
有2种解法：挖坑填坑法和swap法，同时可以通过栈来实现迭代而不是递归，通过比较前中后的值大小，选择合适的pivot，当待排序长度比较短的时候使用插入排序等进行优化。        
具体代码参考：[quick_sort.go](https://github.com/biexiang/code-snippet/blob/main/sortalgo/quick_sort.go)

### 堆排序
两种办法，使用官方的`container/heap`包，或者自己实现堆。      
具体代码参考：[heap_sort.go](https://github.com/biexiang/code-snippet/blob/main/sortalgo/heap_sort.go)

### 插入和希尔排序
希尔排序复杂点，待排序列表分段，每段按照索引和前面的段进行插入排序。
具体代码参考：[insert_sort.go](https://github.com/biexiang/code-snippet/blob/main/sortalgo/insert_sort.go)

## 代码结构
快排怎么实现的，哪里用到了堆排序？
`maxDepth`获取完全二叉树的最大深度

## 代码阅读

### Sort函数
快排怎么实现的，哪里用到了堆排序？

### Stable函数


## Reference
* [sort 包源码分析](https://learnku.com/articles/30404)
* [数据结构中排序为什么要考虑稳定性？](https://www.zhihu.com/question/46809714)
* [...](https://zhuanlan.zhihu.com/p/97965012)
---
title: "Golang database/sql 源码分享"
date: 2021-12-23T16:19:36+08:00
categories: ['Golang','Share']
draft: false
keywords: ['Golang','database/sql','源码阅读','分享','benchmark']
description: "通过oneNote看源码，PPT串流程来分享Golang database/sql的原理，同时最后通过benchmark来进行检验"
---

起因，同事组织部门分享，于是有了这次的分享，分享形式是 PPT + oneNote 看源码。因为之前从来没有做过技术的分享，还算有趣的体验，这种被定了deadline的模式，感觉可以推动自己去学习进步。

PPT：     [Google Slide](https://docs.google.com/presentation/d/1hqpyg88yupIbQg8ZjM9yXVhj6EqdSOEF9hqnYRzlGQw/edit?usp=sharing)

benchmark：[go-sql-test](https://github.com/biexiang/go-sql-test)

oneNote：[read code](https://1drv.ms/u/s!Aj5EXSoGCuo4eBZb0BjbPiAMJGQ)
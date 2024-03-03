---
title: "Go netpool"
date: 2024-02-23T16:00:23+08:00
draft: true
categories: ['Golang','netpool']
keywords: ['Golang','epoll']
description: "netpoll原理"
---

### 笔记
netpoll使用I/O多路复用，linux使用的epoll，通信管道是非阻塞的
net.Listen
    create socket and FD
    bind addr and listen
    init FD
        netFD.init -> poll.FD.init -> FD.pollDesc.init
            runtime_pollServerInit
                epoll global init once
                epoll_create epoll_ctl+FD
                nonblockingpipe
            runtime_pollOpen(uintptr(fd.Sysfd))
                init fd with pollDesc
                set with edge triggered
            
net.Accept
    通过系统调用发现新连接，则包装为FD，设置为非阻塞返回连接
    因为listener也是非阻塞，如果返回EAGAIN，则waitRead进行阻塞
        一直没有可读/可写事件，则当前g gopark

netpool
    caller: startTheWorldWithSema / findrunnable / pollwork / sysmon
    go调度通过epoll_wait获取发生的事件类型、pollDesc得到gopark的g，g状态从waiting重新变成runnable，等待调度


### Reference
* [netpool](https://www.syst.top/posts/go/netpoll/)

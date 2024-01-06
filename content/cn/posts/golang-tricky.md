---
title: "你可能不知道的Golang用法"
date: 2022-03-02T23:11:39+08:00
draft: false
categories: ['Golang','Share']
hide: true
keywords: ['Golang', 'Tricky用法']
description: "Golang中Tricky用法合集，你可能不知道的Golang用法"
---

记录比较奇怪的Golang用法。

### #1 Defer
{{< highlight go "linenos=table,linenostart=1" >}}
func main(){
	f1()
}

func f1(){
	defer trace_leave(trace_enter("f1()"))
	fmt.Println("f1()程序逻辑")
}

func trace_enter(msg string) string{
	fmt.Println("enter: ",msg)
	return msg
}

func trace_leave(msg string) {
	fmt.Println("leave: ",msg)
}

// enter:  f1()
// f1()程序逻辑
// leave:  f1()
{{< / highlight >}}

如上代码片段所示，会先首先执行`trace_enter`，再处理逻辑代码，最后`trace_leave`。
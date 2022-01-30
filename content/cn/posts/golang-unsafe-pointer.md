---
title: "Golang unsafe Pointer 怎么用"
date: 2022-01-23T16:00:23+08:00
draft: false
categories: ['Golang','Share']
---

{{< music id="1003622" >}}

### 起源
Golang官方库[strings.Builder](https://github.com/golang/go/blob/master/src/strings/builder.go#L15)，高效处理字符串，最小化内存拷贝，其中获取字符串方法里，使用了`unsafe.Pointer`进行字节数组到字符串的转换：
```
func (b *Builder) String() string {
	return *(*string)(unsafe.Pointer(&b.buf))
}
```
既然官方都这么用了，写了个[]byte和string互相转换的[benchmark](https://github.com/biexiang/code-snippet/blob/main/str_byte/str_byte_test.go)，测试结果如下：
```
goos: darwin
goarch: amd64
pkg: github.com/biexiang/code-snippet/str_byte
BenchmarkStrByte/NormalBenchmarkStr2Byte-4         	100000000	        12.1 ns/op	       0 B/op	       0 allocs/op
BenchmarkStrByte/UnsafeBenchmarkStr2Byte-4         	1000000000	         0.322 ns/op	       0 B/op	       0 allocs/op
BenchmarkStrByte/NormalBenchmarkByte2Str-4         	204761924	         5.75 ns/op	       0 B/op	       0 allocs/op
BenchmarkStrByte/UnsafeBenchmarkByte2Str-4         	1000000000	         0.317 ns/op	       0 B/op	       0 allocs/op
PASS
```
可见，时间效率提升确实很高，但是包名叫unsafe，就很迷惑，我们到底什么时候可以用这个包呢？😂

### 简单介绍
Golang里的指针分为`类型安全指针`和`非类型安全指针`，前者就是`*T`这种基类型为T的指针，后者就是这里提到的`unsafe.Pointer`。

`类型安全指针`有以下限制：
* 不支持算术运算
* 不能被随意转换为另一个指针类型
* 不能和其他任一指针类型的值进行比较
* 不能被赋值给其他任意类型的指针值

`非类型安全指针`刚好支持如下操作：
* 任何类型的指针都可以转为`unsafe.Pointer`
* `unsafe.Pointer`可以转为任何类型的指针值
* `uintptr`可以转为`unsafe.Pointer`
* `unsafe.Pointer`可以转为`uintptr`

通过使用这些转换规则，我们可以将任意两个`类型安全指针`转换为对方的类型，我们也可以将一个安全指针值和一个`uintptr`值转换为对方的类型。

目前atomic包有loadPointer/storePointer的方法，也是因为不支持泛型的Go版本只能这么处理用户自定义的类型。unsafe包有`Offsetof`、`Sizeof`和`Alignof`这些API，想了半天，就想到修改不可见的私有变量这一个用途，代码如下：
```
package main

import (
	"fmt"
	"unsafe"
)

type private struct {
	caller, callee string
}

// 假设RpcLog在其他包，private成员属性因为小写对包外不可见
type RpcLog struct {
	URI string
	private
}

func main() {
	log := RpcLog{
		URI: "ileopold.cn",
		private: private{
			"chrome", "apache",
		},
	}
	callee := (*string)(unsafe.Pointer(uintptr(unsafe.Pointer(&log)) + unsafe.Offsetof(log.URI) + unsafe.Sizeof("")*2))
	*callee = "nginx"
	fmt.Println(log)
}
```

### 为什么不安全
因为`非类型安全指针`的值是指针，`uintptr`值是整数，如下面的代码，指针指向的内存如果被GC回收掉，但是`uintptr`值感知不到，再次转换成`unsafe.Pointer`去进行赋值时，内存块可能有了别的用途，所以不建议使用。

```
func why() {
	fnCreate := func() *int {
		return new(int)
	}
	i := fnCreate()
	iu := uintptr(unsafe.Pointer(i))
	*(*int)(unsafe.Pointer(iu)) = 3
}
```

### 如何正确使用
* 将类型`*T1`的一个值转换为`非类型安全指针`值，然后将此`非类型安全指针`值转换为类型`*T2`，前提是底层结构上T1包含T2，比如[]byte就比string多一个cap值
* 将一个非类型安全指针值转换为一个uintptr值，然后使用此uintptr值
* 将一个非类型安全指针转换为一个uintptr值，然后此uintptr值参与各种算术运算，再将算术运算的结果uintptr值转回非类型安全指针
* 将一个reflect.SliceHeader或者reflect.StringHeader值的Data字段转换为非类型安全指针，以及其逆转换，如[代码](https://github.com/biexiang/code-snippet/blob/main/str_byte/unsafe/unsafe.go#L27)

### Reference
* [指针](https://gfw.go101.org/article/pointer.html)
* [非类型安全指针](https://gfw.go101.org/article/unsafe.html)

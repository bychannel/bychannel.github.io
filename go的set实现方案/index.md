# go的set实现方案


Go 的设计是一种简单哲学，它摒弃了其他语言一些臃肿的功能和模块，以降低程序员的学习门槛，减少使用中的心智负担。

本文，我们来探讨 Go 中缺失的数据结构：Set，以及它的最佳实现方案。

## Set 语义与实现方案

Set 集合是其他语言中常见的数据结构。特性：集合中的对象不按特定的方式排序，并且没有重复对象。

学习 Go ，要记住：Go 没有包含的东西，不代表 Go 真的没有。根据 Set 特性，我们可以很轻松地想到使用 map 的实现方案（因为 map 的 key 是不重复的）：把对象当做 key 存入 map。

使用 map 来实现 Set，意味着我们只关心 key 的存在，其 value 值并不重要。有其他语言编程经验的人也许会选择 `bool` 来作为 value，因为它是其它语言中内存消耗最少的类型（1个字节）。但是在 Go 中，还有另一种选择：`struct{}`。

```go
fmt.Println(unsafe.Sizeof(struct {}{})) // output: 0
```

## 压测对比

为了探究哪种数据结构是作为 value 的最佳选择。我们选择了以下常用的类型作为 value 进行测试：`bool`、`int`、`interface{}`、`struct{}`。

```go
package main

import (
 "testing"
)

const num = int(1 << 24)

// 测试 bool 类型
func Benchmark_SetWithBoolValueWrite(b *testing.B) {
 set := make(map[int]bool)
 for i := 0; i < num; i++ {
  set[i] = true
 }
}

// 测试 interface{} 类型
func Benchmark_SetWithInterfaceValueWrite(b *testing.B) {
 set := make(map[int]interface{})
 for i := 0; i < num; i++ {
  set[i] = struct{}{}
 }
}

// 测试 int 类型
func Benchmark_SetWithIntValueWrite(b *testing.B) {
 set := make(map[int]int)
 for i := 0; i < num; i++ {
  set[i] = 0
 }
}

// 测试 struct{} 类型
func Benchmark_SetWithStructValueWrite(b *testing.B) {
 set := make(map[int]struct{})
 for i := 0; i < num; i++ {
  set[i] = struct{}{}
 }
}
```

我们运行以下命令，进行测试

```go
$ go test -v -bench=. -count=3 -benchmem | tee result.txt
goos: darwin
goarch: amd64
pkg: workspace/example/demoForSet
cpu: Intel(R) Core(TM) i5-8279U CPU @ 2.40GHz
Benchmark_SetWithBoolValueWrite
Benchmark_SetWithBoolValueWrite-8                1 3549312568 ns/op 883610264 B/op   614311 allocs/op
Benchmark_SetWithBoolValueWrite-8                1 3288521519 ns/op 883599440 B/op   614206 allocs/op
Benchmark_SetWithBoolValueWrite-8                1 3264097496 ns/op 883578624 B/op   614003 allocs/op
Benchmark_SetWithInterfaceValueWrite
Benchmark_SetWithInterfaceValueWrite-8           1 4397757645 ns/op 1981619632 B/op   614062 allocs/op
Benchmark_SetWithInterfaceValueWrite-8           1 4088301215 ns/op 1981553392 B/op   613743 allocs/op
Benchmark_SetWithInterfaceValueWrite-8           1 3990698218 ns/op 1981560880 B/op   613773 allocs/op
Benchmark_SetWithIntValueWrite
Benchmark_SetWithIntValueWrite-8                 1 3472910194 ns/op 1412326480 B/op   615131 allocs/op
Benchmark_SetWithIntValueWrite-8                 1 3519755137 ns/op 1412187928 B/op   614294 allocs/op
Benchmark_SetWithIntValueWrite-8                 1 3459182691 ns/op 1412057672 B/op   613390 allocs/op
Benchmark_SetWithStructValueWrite
Benchmark_SetWithStructValueWrite-8              1 3126746088 ns/op 802452368 B/op   614127 allocs/op
Benchmark_SetWithStructValueWrite-8              1 3161650835 ns/op 802431240 B/op   613632 allocs/op
Benchmark_SetWithStructValueWrite-8              1 3160410871 ns/op 802440552 B/op   613748 allocs/op
PASS
ok   workspace/example/demoForSet 42.660s
```

此时的结果看起来不太直观，这里推荐一个 benchmark 统计工具：Benchstat。通过以下命令进行安装

```go
$ go get -u golang.org/x/perf/cmd/benchstat
```

使用 benchstat 分析刚才得到的 benchmark 结果文件

```shell
$ benchstat result.txt
name                           time/op
_SetWithBoolValueWrite-8        3.37s ± 5%
_SetWithInterfaceValueWrite-8   4.16s ± 6%
_SetWithIntValueWrite-8         3.48s ± 1%
_SetWithStructValueWrite-8      3.15s ± 1%

name                           alloc/op
_SetWithBoolValueWrite-8        884MB ± 0%
_SetWithInterfaceValueWrite-8  1.98GB ± 0%
_SetWithIntValueWrite-8        1.41GB ± 0%
_SetWithStructValueWrite-8      802MB ± 0%

name                           allocs/op
_SetWithBoolValueWrite-8         614k ± 0%
_SetWithInterfaceValueWrite-8    614k ± 0%
_SetWithIntValueWrite-8          614k ± 0%
_SetWithStructValueWrite-8       614k ± 0%
```

从内存开销而言，`struct{}` 是最小的，反映在执行时间上也是最少的。由于 `bool` 类型仅占一个字节，它相较于空结构而言，相差的并不多。但是，如果使用 interface{} 类型，那差距就很明显了。

所以，毫无疑问，在 Set 的实现中， map 值类型应该选 struct{}。

## 总结

本文虽然讨论的是 Set 的实现方案，但本质是涉及空结构体 `struct{}{}` 的 零内存特性。

空结构体除了是实现 Set 的 value 值最佳方案，它还可以应用于以下方面：

- 通知信号的 channel：当 channel 只用于通知 goroutine 的执行事件，此时 channel 就不需要发送任何实质性的数据，选择使用 `chan struct{}` 。

- 没有状态数据的结构体：当对象只拥有方法，而不包含任何的属性字段时，选择使用空结构体定义该对象。


# go的nil用法


`nil` 是 Go 语言中经常使用的标识符，语义上代表了很多类型的零值。相信很多 Gopher 在实际使用中都或多或少踩过 nil 的坑，比如 interface 中 nil 的含义。这篇文章希望梳理一下`nil` 的用法和原理。

这是 buildin/buildin.go 中对于 `nil` 的定义

```go
// nil is a predeclared identifier representing the zero value for a 
// pointer, channel, func, interface, map, or slice type. 
var nil Type // Type must be a pointer, channel, func, interface, map, or slice type
```

### `nil` 是Go语言内预置的标识符

这意味着你可以直接使用，无需额外声明。

### `nil` 是很多类型的零值

- 指针
- Map
- Slice
- Function
- Channel
- Interface

### `nil` 无默认类型

这一点非常重要，目前除了`nil`以外，所有的Go语言预置标识符都有一个**默认类型**，如 iota 的默认类型为 `int`。但`nil` 是个例外，预置的 `nil` 是唯一一个无默认类型的值。编译器需要足够的信息来判断一个 `nil` 值对应的类型。

如下代码可以通过编译：

```go
    _ = (*struct{})(nil)
    _ = []int(nil)
    _ = map[int]bool(nil)
    _ = chan string(nil)
    _ = (func())(nil)
    _ = interface{}(nil)

    // 等价于
    var _ *struct{} = nil
    var _ []int = nil
    var _ map[int]bool = nil
    var _ chan string = nil
    var _ func() = nil
    var _ interface{} = nil
```

下方的代码无法通过编译：

```go
var _ = nil
```

### `nil` 并非 Go 语言的关键字

你会发现，类似下面的代码是可以通过编译的：

```go
package main

import "fmt"

func main() {
    nil := 123
    fmt.Println(nil) // 123
}
```

此时 nil 已经被覆盖，并非是原来的语义，变成了一个 int 类型，值为 123的变量。若在此变量范围中继续使用 `nil` ，将会一直维持这个语义。

### `nil` 所占内存大小随着类型变化而变化

某个类型下所有的变量都有同样的内存结构，`nil` 也不例外。

```go
package main

import (
    "fmt"
    "unsafe"
)

func main() {
    var p *struct{} = nil
    fmt.Println( unsafe.Sizeof( p ) ) // 8

    var s []int = nil
    fmt.Println( unsafe.Sizeof( s ) ) // 24

    var m map[int]bool = nil
    fmt.Println( unsafe.Sizeof( m ) ) // 8

    var c chan string = nil
    fmt.Println( unsafe.Sizeof( c ) ) // 8

    var f func() = nil
    fmt.Println( unsafe.Sizeof( f ) ) // 8

    var i interface{} = nil
    fmt.Println( unsafe.Sizeof( i ) ) // 16
}
```

需要注意的是，上面打印出的大小可能随着运行环境和编译器变化而改变。

### 两个 `nil` 值未必相等

先来看这段代码：

```go
package main

import "fmt"

type SomeStruct struct{}

func main() {
    var h *SomeStruct
    var wrapper interface{} = h
    fmt.Println(h == nil, wrapper == nil) // true, false
}
```

这其实是日常 Go 开发中经常遇到的坑，`h == nil` 返回 `true` 很好理解，但为什么只是用 interface 包装了一层，就不再 `== nil` 了呢？

> non-interface value will be [converted to the type of the interface value](https://link.juejin.cn?target=https%3A%2F%2Fgo101.org%2Farticle%2Finterface.html%23boxing "https://go101.org/article/interface.html#boxing") before making the comparison.

前面我们提到过，Go预置的 `nil` 是没有类型的，为了让 `wrapper` 和 `nil` 进行比较，编译器会首先将 `nil` 转化为一个 `interface{}`，然后进行比较。但是注意，因为 `nil` 无默认类型，即便转为 `interface{}`，它也是没有对应的动态类型的，跟 `wrapper` 的动态类型`*SomeStruct` 不匹配，所以会返回 `false`。

结论：**一个接口包括动态类型和动态值。如果一个接口的动态类型和动态值都为空，则这个接口为空的。如果两个被比较的 `nil` 值，一个是`interface{}`，另一个不是，那么即便可以通过编译，比较结果永远是`false`。**

这样就可以理解，为什么下面的比较结果是 `false`

```go
fmt.Println( (interface{})(nil) == (*int)(nil) ) // false
```

### interface 底层结构

根据 interface 是否包含有 method，底层实现上用两种 struct 来表示：`iface` 和 `eface`。`eface`表示不含 method 的 interface 结构，或者叫 empty interface。

#### eface

```go
type eface struct {
    _type *_type
    data  unsafe.Pointer
}

type _type struct {
    size       uintptr // type size
    ptrdata    uintptr // size of memory prefix holding all pointers
    hash       uint32  // hash of type; avoids computation in hash tables
    tflag      tflag   // extra type information flags
    align      uint8   // alignment of variable with this type
    fieldalign uint8   // alignment of struct field with this type
    kind       uint8   // enumeration for C
    alg        *typeAlg  // algorithm table
    gcdata    *byte    // garbage collection data
    str       nameOff  // string form
    ptrToThis typeOff  // type for pointer to this type, may be zero
}
```

#### iface

iface 表示 non-empty interface 的底层实现。相比于 empty interface，non-empty 要包含一些 method。method 的具体实现存放在 itab.fun 变量里。如果 interface 包含多个 method，这里只有一个 fun 变量怎么存呢？这个下面再细说。

```go
type iface struct {
    tab  *itab
    data unsafe.Pointer
}

// layout of Itab known to compilers
// allocated in non-garbage-collected memory
// Needs to be in sync with
// ../cmd/compile/internal/gc/reflect.go:/^func.dumptypestructs.
type itab struct {
    inter  *interfacetype
    _type  *_type
    link   *itab
    bad    int32
    inhash int32      // has this itab been added to hash?
    fun    [1]uintptr // variable sized
}
```

概括起来，接口对象由接口表 (interface table) 指针和数据指针组成，或者说由动态类型和动态值组成。

接口表存储元数据信息，包括接口类型、动态类型，以及实现接口的方法指针。无论是反射还是通过接口调用方法，都会用到这些信息。

### 怎么解决 interface 和 `nil` 的比较

上一节我们得出了结论

> 如果两个被比较的 `nil` 值，一个是`interface{}`，另一个不是，那么即便可以通过编译，比较结果永远是`false`。

但实际应用场景中，依然有 interface 和 `nil` 比较的诉求，很多时候我们不希望在意类型，只是希望明确当前这个 interface 动态值，是否为零值。用`==`显然是无法做到这一点。

这个时候我们可以借助反射的帮助来实现。

在 Golang relfect 包的文档中我们可以看到

```go
// IsNil reports whether its argument v is nil. 
// The argument must be a chan, func, interface, map, pointer, or slice value; if it is not, IsNil panics.
// Note that IsNil is not always equivalent to a regular comparison with nil in Go. For example, if v was created by calling ValueOf with an uninitialized interface variable i, i==nil will be true but v.IsNil will panic as v will be the zero Value.

func (v Value) IsNil() bool
```

先将 interface 值转化为 `reflect.Value`，然后借用`IsNil` 来判断是否为空即可。
示例代码：

```go
func isNil(i interface{}) bool {
    return i == nil || reflect.ValueOf(i).IsNil()
}
```

但事实上，使用reflect包下的方法一定要小心，此处入参 i 的类型为 `interface{}`，也就意味着任何类型的值传进来皆可，贸然使用反射，容易引发 `panic`。

如果 i 是一个普通的结构体，非指针类型。此处`IsNil`会直接抛`panic`，注意文档注释。

> The argument must be a chan, func, interface, map, pointer, or slice value; if it is not, IsNil panics

所以，修改代码逻辑如下：

```go
func isNilFixed(i interface{}) bool {
   if i == nil {
      return true
   }
   switch reflect.TypeOf(i).Kind() {
   case reflect.Ptr, reflect.Map, reflect.Array, reflect.Chan, reflect.Slice:
      return reflect.ValueOf(i).IsNil()
   }
   return false
}
```


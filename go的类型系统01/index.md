# go的类型系统01


本文介绍 类型转换(Conversion)、类型断言(type assertion) 和类型切换（type switch)。

这三个概念类似但是又完全不同。

### 类型转换 Conversion

将一个值x转换成特定类型T,格式为 `T(x)`,非常的简单，类型加小括号即可。

如果类型T以 *、<-、func(不带结果列表)，未避免造成歧义，需要将类型括号包裹起来： `(T)(x)`:

```go
*Point(p)        // same as *(Point(p))
(*Point)(p)      // p is converted to *Point
<-chan int(c)    // same as <-(chan int(c))
(<-chan int)(c)  // c is converted to <-chan int
func()(x)        // function signature func() x
(func())(x)      // x is converted to func()
(func() int)(x)  // x is converted to func() int
func() int(x)    // x is converted to func() int (unambiguous)
```

并不是任意的值都可以转换成类型T, 它需要遵循一定的规则，下面一一道来。

对于一个常量值x, 如果能转换成T类型的值，它需要满足下面的条件之一:

#### 转换常量值

- x 可以表达为T的值
- x 是浮点数值， T是浮点类型。 x 使用 IEEE 754 round-to-even 规则 且 IEEE -0.0 会进一步舍入到无符号的 0.0， 经过舍入后的x可以表示为T。这一条主要约束浮点数取整的规则，并不是完全按照四舍五入规则计算的。
- x是一个整数而T是字符串类型

```go
uint(iota)               // iota value of type uint
float32(2.718281828)     // 2.718281828 of type float32
complex128(1)            // 1.0 + 0.0i of type complex128
float32(0.49999999)      // 0.5 of type float32
float64(-1e-1000)        // 0.0 of type float64
string('x')              // "x" of type string
string(0x266c)           // "♬" of type string
MyString("foo" + "bar")  // "foobar" of type MyString
string([]byte{'a'})      // not a constant: []byte{'a'} is not a constant
(*int)(nil)              // not a constant: nil is not a constant, *int is not a boolean, numeric, or string type
int(1.2)                 // illegal: 1.2 cannot be represented as an int
string(65.0)             // illegal: 65.0 is not an integer constant
```

#### 转换变量值

对于一个常量值x, 如果能转换成T类型的值，它需要满足下面的条件之一:

- x可以[赋值](https://golang.org/ref/spec#Assignability)给 T
- x的类型和T的底层类型 类型一致
- x类型和T 都是未命名的指针类型，它们的指针指向的对象类型 类型一致
- x的类型和T都是整数或者浮点数
- x的类型和T都是复数
- x是整数、slice of byte、slice of rune, T是字符串类型
- x是字符串， T是slice of byte 或者slice of rune

数值类型和字符串之间的转换可能会改变x的呈现并且会带来运行时的花费。  
其它的转换只是改变x的类型，不会改变x的呈现。

并没有直接整数和指针之间的转换。但是在前面的章节中也举例了，指针可以通过曲折的方式转换成整数，  
它是通过包`unsafe`实现的， 甚至于你通过这种方式还可以访问struct未导出的字段。

类型不一致的两个变量不能赋值, 并且也没有什么强制类型转换的概念：

```go
var i1 int8 = 10
var i2 uint8 = i1 //错误
var i3 uint8 = (uint8)i1 //错误
```

比如在类型那一章讲的例子，也是通过这种类型转换实现的:

```go
x := [...]int{1, 2, 3, 4, 5}
p := &x[0]
//p = p + 1
index2Pointer := unsafe.Pointer(uintptr(unsafe.Pointer(p)) + unsafe.Sizeof(x[0]))
p = (*int)(index2Pointer) //x[1]
fmt.Printf("%d\n", *p)    //2
```

### 类型转换实践

这一节介绍常见类型一致的转换。

#### 数值类型之间的转换

非常量的数值之间的转换遵循下面三条原则:  
1、整数之间的转换时，如果值是有符号的整数，它的符号位会扩展无限大，否则零扩展，然后它会被删减以适合结果类型。怎么理解，看例子。  
对于无符号数v: `v := uint16(0x10F0)`,如果进行转换`uint32(int8(v))`,可以看到它的结果是`0xFFFFFFF0`，不会有溢出指示或者错误。

```go
v1 := uint16(0x10F0)
fmt.Printf("%d=%b\n", v1, v1) //4336=1000011110000
v2 := int8(v1)
fmt.Printf("%d=%b\n", v2, v2) //-16=-10000
v3 := uint16(v2)
fmt.Printf("%d=%b\n", v3, v3) //65520=1111111111110000
v4 := int16(v2)
fmt.Printf("%d=%b\n", v4, v4) //-16=-10000
```

介绍一下。 对于v1,它是一个无符号的整数， 要把它转为有符号的int8，那么我们只看v1的后8位:

```go
1111 0000
```

不幸的是，这个8位的最高位是1,我们会把它作为符号位，所以v2是个负数，那么`11110000`就是这个负数的补码，  
那么它的原码是多少呢，计算补码的补码就是负数的原码:`1001 0000`,所以它是-16。如果最高位是0，简单了，本身就是它的原码。

再看v2转v3， 也就是有符号整数转无符号整数。v2是负数，内部表示为`11110000`,因为要扩展为16位，将符号位1扩展到最高位`1111 1111 1111 0000`,因为它是无符号整数，所以这个值整数的值65520。

你可以把v1值的值改为0xff60看看输出是什么？此时转换不会符号位为负数的情况。

> 补码（two's complement) 指的是正数=原码，负数=反码加一  
> 反码（ones' complement) 指的就是通常所指的反码。  
> 对一个整数的补码再求补码，等于该整数自身。  
> 补码的正零与负零表示方法相同。

2、浮点数转换成整数时，小数部分被丢弃,也就是朝0方向舍入。

```go
var v1 float32 = 0.999999
fmt.Println(int(v1))
v1 = -0.999999
fmt.Println(int(v1))
```

3、当转换整数或者浮点数到浮点数的时候，或者一个复数到另一个复数， 结果值会被舍入到目标类型的精度。例如类型为float32的变量x可以通过附加的精度超过标准的IEEE-754 32-bit数， 但是float32(x)代表x的值舍入到 IEEE-754 32 bit的精度。类似地， x + 0.1 可以使用超过32 bit的精度，但是float32(x + 0.1) 肯定是32 bit的精度。

> the value of a variable x of type float32 may be stored using additional precision beyond that of an IEEE-754 32-bit number, but float32(x) represents the result of rounding x's value to 32-bit precision. Similarly, x + 0.1 may use more than 32 bits of precision, but float32(x + 0.1) does not.

关于浮点数格式IEEE-754, 随便一本计算机原理的教材中都会介绍，网上也有无数的文章介绍，它由三个域组成，float32中分别占1位、8位、和 23位,本文中就不详细介绍了。  
![float32](/images/float32.png)

#### 整数和bool之间的转换

虽然有人提议实现快速的整数和bool之间的转换，但是目前看起来还没有实现，所以下面的语句是不对的：

```go
i1 := 1
i2 := 0
fmt.Printf("%t %t\n", bool(i1), bool(i2))
```

但是你完全可以通过其它方式实现， 比如判断语句 `n > 0`, 或者利用一个定义好的表(map,数组等)进行查表转换。

参考

- https://github.com/golang/go/issues/6011
- [language: bool to numeric and numeric to bool type conversions · Issue #7657 · golang/go · GitHub](https://github.com/golang/go/issues/7657)

#### 基于字节的字符串的转换

字符串代表一串字节流，所以很容易的和slice of byte, slice of rune进行转换。  
1、无符号整数或者有符号整数通过它对应的UTF-8编码转换成字符串。合法的Unicode code之外的值都被转换成`\uFFFD`。这里的整数也包含rune.

```go
string('a')       // "a"
string(-1)        // "\ufffd" == "\xef\xbf\xbd"
string(0xf8)      // "\u00f8" == "ø" == "\xc3\xb8"
type MyString string
MyString(0x65e5)  // "\u65e5" == "日" == "\xe6\x97\xa5"
```

2、字节slice根据UTF-8编码产生字符串

```go
string([]byte{'h', 'e', 'l', 'l', '\xc3', '\xb8'})   // "hellø"
string([]byte{})                                     // ""
string([]byte(nil))                                  // ""
```

3、将rune slice转换成字符串相当于将rune连接起来

```go
string([]rune{0x9E1F, 0x7A9D})   // "\u9e1f\u7a9d" == "鸟窝"
string([]rune{})                         // ""
string([]rune(nil))                      // ""
```

4、将字符串转为byte slice会将字符串的字节流复制到一个byte slice

5、将一个字符串转为rune slice会将产生一个新的rune slice,包含字符串中每个rune

#### 字符串和基本类型之间的转换

包strconv提供了字符串和基本数据类型的转换。上面我们提到了字符串和整数之间的转换，但是有时候我们需要的是将 12转换成字符串 "12"，或者从字符串中解析处一个整数，这个时候就可以使用这个包。

首先它提供了一组往byte slice增加基本类型元素的方法：

```go
func AppendBool(dst []byte, b bool) []byte
func AppendFloat(dst []byte, f float64, fmt byte, prec, bitSize int) []byte
func AppendInt(dst []byte, i int64, base int) []byte
func AppendQuote(dst []byte, s string) []byte
func AppendQuoteRune(dst []byte, r rune) []byte
func AppendQuoteRuneToASCII(dst []byte, r rune) []byte
func AppendQuoteRuneToGraphic(dst []byte, r rune) []byte
func AppendQuoteToASCII(dst []byte, s string) []byte
func AppendQuoteToGraphic(dst []byte, s string) []byte
func AppendUint(dst []byte, i uint64, base int) []byte
```

一组从字符串中解析出基本类型的方法：

```go
func ParseBool(str string) (value bool, err error)
func ParseFloat(s string, bitSize int) (f float64, err error)
func ParseInt(s string, base int, bitSize int) (i int64, err error)
func ParseUint(s string, base int, bitSize int) (n uint64, err error)
```

一组为字符串或者rune加引号和剥离引号的方法:

```go
func Quote(s string) string
func QuoteRune(r rune) string
func QuoteRuneToASCII(r rune) string
func QuoteRuneToGraphic(r rune) string
func QuoteToASCII(s string) string
func QuoteToGraphic(s string) string
func Unquote(s string) (t string, err error)
func UnquoteChar(s string, quote byte) (value rune, multibyte bool, tail string, err error)
```

一组检查字符串或者rune为特定类型的方法：

```go
func CanBackquote(s string) bool
func IsGraphic(r rune) bool
func IsPrint(r rune) bool
```

一组格式化基本类型为字符串的方法:

```go
func FormatBool(b bool) string
func FormatFloat(f float64, fmt byte, prec, bitSize int) string
func FormatInt(i int64, base int) string
func FormatUint(i uint64, base int) string
```

重要的放在最后说，我们在编程中更多的用到的两个方法, 整数字面值和字符串之间的转换:

```go
func Atoi(s string) (i int, err error)
func Itoa(i int) string
```

参考

- https://golang.org/pkg/strconv/

#### 字节slice和整数之间的转换

包 encoding/binary实现了数值和字节序列之间的转换，包含变长int的各种编解码。

Go中的数值类型都是固定长度的位数(int8, uint8, int16, float32, complex64)，所以组成这些数组的bit可以转换成各种字节slice。

变长int (varint)经常用于节省空间，比如一个， Go实现的varint规范可以参考[proto-buff的实现](https://developers.google.com/protocol-buffers/docs/encoding)。很多编解码库中都使用了变长的int，这样对于大量的小数字我们可以用更少的字节来表示，对于网络传输来说很有好处。

这个包经常用在网络传输的序列化和反序列中。

另外一个值得注意的是数值是由多个字节组成的，这就涉及到字节序的问题，你必须指定使用小端序或大端序。

首先看一下定长的数值的转换，主要是`Read`和`Write`两个方法，底层还是通过移位操作实现的。

```go
func Read(r io.Reader, order ByteOrder, data interface{}) error
func Write(w io.Writer, order ByteOrder, data interface{}) error
```

例子：

```go
package main
import (
    "bytes"
    "encoding/binary"
    "fmt"
)
func main() {
    b := write()
    read(b)
}
func write() []byte {
    buf := new(bytes.Buffer)
    var data = []interface{}{
        uint16(61374), //efbe
        int8(-54),     //-36
        uint8(254),    //fe
    }
    for _, v := range data {
        err := binary.Write(buf, binary.BigEndian, v)
        if err != nil {
            fmt.Println("binary.Write failed:", err)
        }
    }
    fmt.Printf("%x\n", buf.Bytes()) //efbecafe
    return buf.Bytes()
}
func read(b []byte) {
    var i1 uint16
    var i2 int8
    var i3 uint8
    buf := bytes.NewReader(b)
    err := binary.Read(buf, binary.BigEndian, &i1)
    if err != nil {
        fmt.Println("binary.Read failed:", err)
    }
    err = binary.Read(buf, binary.BigEndian, &i2)
    if err != nil {
        fmt.Println("binary.Read failed:", err)
    }
    err = binary.Read(buf, binary.BigEndian, &i3)
    if err != nil {
        fmt.Println("binary.Read failed:", err)
    }
    fmt.Println(i1, i2, i3) //61374 -54 254
}
```

一种不通用的适合特定类型的转换也可以使用下面的方法：

```go
func readInt32(b []byte) int32 {
    // equivalnt of return int32(binary.LittleEndian.Uint32(b))
    return int32(uint32(b[0]) | uint32(b[1])<<8 | uint32(b[2])<<16 | uint32(b[3])<<24)
}
//或者
func ReadInt32Unsafe(b []byte) int32 {
    return *(*int32)(unsafe.Pointer(&b[0]))
}
```

变长int的操作函数:

```go
func PutUvarint(buf []byte, x uint64) int
func PutVarint(buf []byte, x int64) int
func Uvarint(buf []byte) (uint64, int)
func Varint(buf []byte) (int64, int)
func ReadUvarint(r io.ByteReader) (uint64, error)
func ReadVarint(r io.ByteReader) (int64, error)
```

以及一个对象被转换成多少字节的方法:

```go
func Size(v interface{}) int
```

参考

- https://golang.org/pkg/encoding/binary/
- [https://zh.wikipedia.org/wiki/字节序#.E5.AD.97.E8.8A.82.E9.A1.BA.E5.BA.8F](https://zh.wikipedia.org/wiki/%E5%AD%97%E8%8A%82%E5%BA%8F#.E5.AD.97.E8.8A.82.E9.A1.BA.E5.BA.8F)
- http://codereview.stackexchange.com/questions/15945/effectively-convert-little-endian-byte-slice-to-int32

#### 数组和slice之间的转换

数组转换成slice很简单，前面讲到了，利用索引运算：

```go
a[low:high]
a[low:high:max]
```

而slice转数组，我们可以好好分析一下。

slice的底层实现是数组，所以有一个"hack"方法，将slice的底层数组返回:

```go
import (
    "reflect"
    "unsafe"
)
const SIZEOF_INT32 = 4 // bytes
// Get the slice header
header := *(*reflect.SliceHeader)(unsafe.Pointer(&raw))
// The length and capacity of the slice are different.
header.Len /= SIZEOF_INT32
header.Cap /= SIZEOF_INT32
// Convert slice header to an []int32
data := *(*[]int32)(unsafe.Pointer(&header))
```

安全的方式是生成数组然后依次赋值，注意copy是不行的，因为copy的参数必须都是slice:

```go
import "encoding/binary"
const SIZEOF_INT32 = 4 // bytes
data := make([]int32, len(raw)/SIZEOF_INT32)
for i := range data {
    // assuming little endian
    data[i] = int32(binary.LittleEndian.Uint32(raw[i*SIZEOF_INT32:(i+1)*SIZEOF_INT32]))
}
```

参考

- [go - How do you convert a slice into an array? - Stack Overflow](http://stackoverflow.com/questions/19073769/in-golang-how-do-you-convert-a-slice-into-an-array)
- [go - Convert between slices of different types - Stack Overflow](http://stackoverflow.com/questions/11924196/convert-between-slices-of-different-types)
- https://groups.google.com/forum/#!topic/golang-nuts/yNis7bQG_rY

#### struct和字符串之间的转换

struct类型的值和字符串之间的转换我们称之为marshal和unmarshal。  
有非常多的库可以做这个事情，比如gob, encoding/json等。

Go序列化框架的性能比较可以参照我的一个开源项目: [gosercomp](https://github.com/smallnest/gosercomp)。

#### Java字符串和Go字符串之间的转换

Java字符串在内部是以UTF-16编码方式存在的，每个字符包含两个字节。而Go字符串在内部是以UTF-8格式存在的，每个字符串占用的字节数可能不同。

可以使用[unicode](https://colobu.com/2016/06/21/dive-into-go-6/golang.org/x/text/encoding/unicode)包进行转换，或者使用[unicode/utf16](https://golang.org/pkg/unicode/utf16)

参考

- https://jannewmarch.gitbooks.io/network-programming-with-go-golang-/content/encoding/utf-16_and_go.html

### 类型断言 type assertion

和上节的类型转换不同，类型断言是将接口类型的值x，转换成类型T。

格式为：

```go
x.(T)
v := x.(T)
v, ok := x.(T)
```

类型断言的必要条件是x是接口类型,非接口类型的x不能做类型断言:

```go
var i int = 10
v := i.(int) //错误
```

T可以是非接口类型，如果想断言合法，则T应该实现x的接口。

T也可以是接口，则x的动态类型也应该实现接口T。

```go
var x interface{} = 7  // x 的动态类型为int， 值为 7
i := x.(int)           // i 的类型为 int， 值为 7
type I interface { m() }
var y I
s := y.(string)        // 非法: string 没有实现接口 I (missing method m)
r := y.(io.Reader)     // y如果实现了接口io.Reader和I的情况下，  r的类型则为io.Reader
```

类型断言如果非法，运行时时候就会出现 impossible type assertion panic，为了避免这种情况，可以使用下面的语法:

```go
v, ok = x.(T)
v, ok := x.(T)
var v, ok = x.(T)
```

ok代表类型断言是否合法，如果非法ok =false,v为T的零值，这样就不会出现运行时panic了。

希望你能记住，类型转换和类型断言完全是两个概念。

### 类型切换 type switch

类型切换(暂且这么翻译吧，英语更准确)用来比较类型而不是对值进行比较。

switch语句虽然在下一章中去讲，但是对于读者来说，多少会一种或者几种常用的编程语言，switch是一个条件语句，它可以判断某个值是否匹配某个case clause。但是对于type switch，它检查的是值x的类型T是否匹配某个类型。

格式如下，类型类型断言，但是括号内的不是某个具体的类型，而是单词`type`:

```go
switch x.(type) {
// cases
}
```

type switch语句中可以有一个简写的变量声明，这种情况下，等价于这个变量声明在每个case clause隐式代码块的开始位置。如果case clause只列出了一个类型，则变量的类型就是这个类型，否则就是原始值的类型。

假设下面的例子中x的类型为x interface{}

```go
switch i := x.(type) {
case nil:
  printString("x is nil") // i的类型是 x的类型 (interface{})
case int:
  printInt(i) // i的类型 int
case float64:
  printFloat64(i) // i的类型是 float64
case func(int) float64:
  printFunction(i) // i的类型是 func(int) float64
case bool, string:
  printString("type is bool or string") // i的类型是 x (interface{})
default:
  printString("don't know the type") // i的类型是 x的类型 (interface{})
}
```

也许你已经看到上面的例子中有一个case clause中的类型是nil,它用来匹配x为nil的interface{}的情况。


# go的类型系统02


本文介绍 类型别名与类型定义。

### 类型别名

类型别名需要在别名和原类型之间加上赋值符号 = ，使用类型别名定义的类型与原类型等价，Go 语言内建的基本类型中就存在两个别名类型。

* byte 是 uint8 的别名类型；

* rune 是 int32 的别名类型；

类型别名定义形式如下：

```go
type MyString = string
```

上面代码表明 MyString 是 string 类型的别名类型。也就是说别名类型和源类型表示的是同一个目标，就譬如每个人的学名和乳名一样，都表示同一个人。

定义 string 类型的别名，示例代码：

```go
func main() {
 type MyString = string
 str := "hello"
 a := MyString(str)
 b := MyString("A" + str)
 fmt.Printf("str type is %T\n", str)
 fmt.Printf("a type is %T\n", a)
 fmt.Printf("a == str is %t\n", a == str)
 fmt.Printf("b > a is %t\n", b > a)
}
```

输出结果

```go
str type is string
a type is string
a == str is true
b > a is false
```

定义 []string 类型的别名，示例代码：

```go
func main() {
 type MyString = string
 strs := []string{"aa", "bb", "cc"}
 a := []MyString(strs)
fmt.Printf("strs type is %T\n", strs)
fmt.Printf("a type is %T\n", a)
fmt.Printf("a == nil is %t\n", a == nil)
fmt.Printf("strs == nil is %t\n", strs != nil)
}
```

运行结果为：

```go
strs type is []string
a type is []string
a == nil is false
strs == nil is true
```

从上面结果可以得出以下结论：

别名类型与源类型是完全相同的；
别名类型与源类型可以在源类型支持的条件下进行相等判断、比较判断、与 nil 是否相等判断等；

### 类型定义

类型定义是定义一种新的类型，它与源类型是不一样的。看下面代码：

```go
func main() {
 type MyString string
 str := "hello"
 a := MyString(str)
 b := MyString("A" + str)
 fmt.Printf("str type is %T\n", str)
 fmt.Printf("a type is %T\n", a)
 fmt.Printf("a value is %#v\n", a)
 fmt.Printf("b value is %#v\n", b)
 // fmt.Printf("a == str is %t\n", a == str)
 fmt.Printf("b > a is %t\n", b > a)
}
```

输出结果为：

```go
str type is string
a type is main.MyString
a value is "hello"
b value is "Ahello"
b > a is false
```

可以看到 MyString 类型为 main.MyString 而原有的 str 类型为 string，两者是不同的类型，如果使用下面的判断相等语句

```go
fmt.Printf("a == str is %t\n", a == str)
```

会有编译错误提示

```go
invalid operation: a == str (mismatched types MyString and string)
```

说明类型不匹配。

下面代码

```go
func main() {
 type MyString string
strs := []string{"E", "F", "G"}
myStrs := []MyString(strs)
fmt.Println(myStrs)
}
```

编译报错提示

```go
cannot convert strs (type []string) to type []MyString
```

对于这里的类型再定义来说，string 可以被称为 MyString2 的潜在类型。潜在类型相同的不同类型的值之间是可以进行类型转换的。

因此，MyString2 类型的值与 string 类型的值可以使用类型转换表达式进行互转。但对于集合类的类型[]MyString2 与 []string 来说这样做却是不合法的，因为 []MyString2 与 []string 的潜在类型不同，分别是 []MyString2 和 []string 。

另外，即使两个不同类型的潜在类型相同，它们的值之间也不能进行判等或比较，它们的变量之间也不能赋值。

```go
func main() {
 type MyString1 = string
 type MyString2 string
 str := "BCD"
 myStr1 := MyString1(str)
 myStr2 := MyString2(str)
 myStr1 = MyString1(myStr2)
 myStr2 = MyString2(myStr1)

myStr1 = str

myStr2 = str
// cannot use str (type string) as type MyString2 in assignment

myStr1 = myStr2
// cannot use myStr2 (type MyString2) as type string in assignment

myStr2 = myStr1
// cannot use myStr1 (type string) as type MyString2 in assignment
}
```

### 类型别名注意点

#### 类型循环

类型别名在定义的时候不允许出现循环定义别名的情况，如下面所示：

```go
type T1 = T2
type T2 = T1
```

上面的例子太明显，下面这个例子比较隐蔽，也是循环定义类型别名的情况，当然这些在编译代码的时候编译器会帮你检查，如果出现循环定义的情况会出错。

```go
type T1 = struct {
	next *T2
}
type T2 = T1
```

#### 可导出性

如果定义的类型别名是`exported` (首字母大写)的，那么别的包中就可以使用，它和原始类型是否可`exported`没关系。也就是说，你可以为`unexported`类型定义一个`exported`的类型别名，如下面的例子：

```go
type t1 struct {
	S string
}
type T2 = t1
```



#### 方法集

既然类型别名和原始类型是相同的，那么它们的方法集也是相同的。

下面的例子中`T1`和`T3`都有`say`和`greeting`方法。

```go
type T1 struct{}
type T3 = T1
func (t1 T1) say(){}
func (t3 *T3) greeting(){}
func main() {
	var t1 T1
	// var t2 T2
	var t3 T3
	t1.say()
	t1.greeting()
	t3.say()
	t3.greeting()
}
```



如果类型别名和原始类型定义了相同的方法，代码编译的时候会报错，因为有重复的方法定义。

另一个有趣的现象是 `embedded type`, 比如下面的例子， `T3`是`T1`的别名。在定义结构体`S`的时候，我们使用了匿名嵌入类型，那么这个时候调用`s.say`会怎么样呢？ 实际是你会编译出错，因为`s.say｀不知道该调用`s.T1.say`还是`s.T3.say`，所以这个时候你需要明确的调用。

```go
type T1 struct{}
type T3 = T1
func (t T1) say(){}
type S struct {
	T1
	T3
}
func main() {
	var s S
	s.say()
}
```

进一步想，这样是不是我们可以为其它库中的类型增加新的方法了， 比如为标准库的`time.Time`增加一个滴答方法:

```go
type NTime = time.Time
func (t NTime) Dida() {
	fmt.Println("嘀嗒嘀嗒")
}
func main() {
	t := time.Now()	
	t.Dida()
}
```

答案是: **NO**, 编译的时候会报错: `cannot define new methods on non-local type time.Time`。


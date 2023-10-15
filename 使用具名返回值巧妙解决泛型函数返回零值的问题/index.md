# 【转载】使用具名返回值巧妙解决泛型函数返回零值的问题


> 注：本文转载自 https://tonybai.com/2022/05/20/solving-problems-in-generic-function-implementation-using-named-return-values/
> 
> 如有侵权，联系删除

Go语言泛型语法特性在[Go 1.18版本](https://tonybai.com/2022/04/20/some-changes-in-go-1-18)落地后，不出所料，在github上看到大量的基础容器类型数据结构被用泛型重写。这种重写我觉得是很正常、很自然的，并且实现良好的通用数据结构改为泛型其实也不难，有些简单的结构可能分分钟就能搞定。

Go 1.18发布后，我一直没机会写泛型，今天在做[DSL](https://tonybai.com/2022/05/10/introduction-of-implement-dsl-using-antlr-and-go)语义模型提取时，多处用到Stack结构，于是想到使用泛型简单实现了一个通用的Stack结构。

在Go中，我们可以用一个切片来定义Stack。泛型Stack类型的定义如下：

```
type Stack[T any] []T
```

这里的Stack类型就是一个带有类型参数(type parameter)的泛型类型，它的类型参数的约束(constraints)为[any](https://tonybai.com/2021/12/18/replace-empty-interface-with-any-first-after-switching-to-go-1-18)，即允许任何类型作为Stack的元素类型。

Stack是最基础的数据结构，一般来说它具有的操作方法包括：

- Push：压栈；
- Pop：弹栈；
- Top：获取栈顶元素；
- Len：获取栈内元素个数。

对于以切片为底层存储的Stack而言，压栈Push操作就相当于对切片的追加(append)操作：

```
func (s *Stack[T]) Push(v T) {
    (*s) = append((*s), v)
}
```

不过，这里有两点要注意：

- 泛型类型的方法原型中，receiver部分的类型要带上类型参数，比如这里的*Stack[T]；
- 这里务必要用*Stack[T]，而不要像下面代码这样用Stack[T]，否则append方法改变的仅仅是Stack[T]的拷贝，而不是原Stack[T]类型的实例。

```
func (s Stack[T]) Push(v T) {
    s = append(s, v)
}
```

我们再来看看*Stack[T]的弹栈Pop方法：

```
func (s *Stack[T]) Pop() T {
    if len(*s) == 0 {
        return nil
    }     

    // Get the last element from the stack.
    t := (*s)[len(*s)-1]

    // Remove the last element from the stack.
    *s = (*s)[:len(*s)-1]
    return t
}
```

这样实现的Pop方法会提示return nil一行有错误：**cannot use nil as T value in return statement**。Go编译器错误信息提示我们：**nil不能作为T类型的值返回**。

Stack的类型参数的约束为any，即Stack的元素可以是任意类型，即可以是切片、map等复合类型，亦可以是int、string等值类型。如果将nil作为所有这些类型的零值的确不恰当。

那么当Stack为空时，应该如何返回呢？多亏[Go原生支持类型零值](https://www.imooc.com/read/87/article/2381)，我们可以声明一个类型零值并将其作为返回值返回：

```
func (s *Stack[T]) Pop() T {
    if len(*s) == 0 {
        var zero T
        return zero // 模拟类型零值
    }     

    // Get the last element from the stack.
    t := (*s)[len(*s)-1]

    // Remove the last element from the stack.
    *s = (*s)[:len(*s)-1]
    return t
}
```

虽然这种方法有效，但你是不是和我有一样的感觉：**不够优雅**。下面我们就来看一个更为优雅的小技巧：**利用函数的具名返回值**，看代码：

```
func (s *Stack[T]) Pop() (t T) {
    if len(*s) == 0 {
        return
    } 

    // Get the last element from the stack.
    t = (*s)[len(*s)-1]

    // Remove the last element from the stack.
    *s = (*s)[:len(*s)-1]
    return
}
```

我们看到：具名返回值(named return value)一出马，一切都变得自然而然了。当然这也要归功于Go的类型零值特性。

具名返回值日常使用的不多，从使用的频度来看，Go标准库以及多数项目的代码默认选择非具名返回值(unamed return value)。当函数使用defer且在deferred函数中修改外部函数返回值时，应用具名返回值可以让代码显得更清晰一些：

```
func Foo() (a int) {
    defer func() {
        a = 5
    }

    a = 6
}
```

其他情况，看项目编码规范一致性要求以及个人喜好了。不过，Go引入泛型后，针对上述的泛型函数返回零值的情况，相信**具名返回值**将得到更多的“出镜”的机会。


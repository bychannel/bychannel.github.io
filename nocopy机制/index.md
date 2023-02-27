# noCopy机制


最近在读Go源码时发现高频注释语句“**XXX must not be copied after first use**“。例如sync包下的Pool、Cond、WaitGroup、Mutex、Map和atomoic.Vaule、strings.Builder等，都有该句注释。

## 为什么注释文档中要强调no copy？

**安全！** 如果结构体对象包含指针字段，当该对象被拷贝时，会使得两个对象中的指针字段变得不再安全。

```go
type S struct {
 f1 int
 f2 *s
}

type s struct {
 name string
}

func main() {
 mOld := S{
  f1: 0,
  f2: &s{name: "mike"},
 }
 mNew := mOld //拷贝
 mNew.f1 = 1
 mNew.f2.name = "jane"

 fmt.Println(mOld.f1, mOld.f2) //输出：0 &{jane}
}
```

如上，结构体对象S中存在两个field，分别是f1和f2，其中f2是指向s类型的指针。当mNew复制了mOld之后，mNew对两个字段进行了改变，可以看到f1字段的更改，不会对mOld造成影响。但是，nNew中f2字段的修改也会把mOld中的f2字段修改掉，这引发了安全问题。

## Go是如何保证no copy的？

### runtime checking

- strings.Builder中copy检查

```go
func main() {
 var a strings.Builder
 a.Write([]byte("a"))
 b := a
 b.Write([]byte("b"))
}
// 运行报错：strings: illegal use of non-zero Builder copied by value
```

报错信息，来源于strings.Builder的copyCheck。

```go
type Builder struct {
 addr *Builder // of receiver, to detect copies by value
 buf  []byte
}

func (b *Builder) Write(p []byte) (int, error) {
 b.copyCheck()
 b.buf = append(b.buf, p...)
 return len(p), nil
}

func (b *Builder) copyCheck() {
 if b.addr == nil {
  b.addr = (*Builder)(noescape(unsafe.Pointer(b)))
 } else if b.addr != b {
  panic("strings: illegal use of non-zero Builder copied by value")
 }
}
```

在Builder中，addr是一个指向自身的指针。当对上文中的a复制给b时，a和b本身是不同的对象。因此，b.addr实际还是指向a的指针，这就会触发条件b.addr!=b，造成panic。

- sync.Cond中copy检查

在源码中，拥有copy检查机制的还有sync.Cond。

```go
type Cond struct {
 noCopy noCopy
 L Locker
 notify  notifyList
 checker copyChecker
}

func (c *Cond) Wait() {
 c.checker.check()
 ...
}

type copyChecker uintptr

func (c *copyChecker) check() {
 if uintptr(*c) != uintptr(unsafe.Pointer(c)) &&
  !atomic.CompareAndSwapUintptr((*uintptr)(c), 0, uintptr(unsafe.Pointer(c))) &&
  uintptr(*c) != uintptr(unsafe.Pointer(c)) {
  panic("sync.Cond is copied")
 }
}
```

这里的check函数初看不易明白。因此，定义一个相似的结构体对象，来探究这里的check函数究竟是如何做copy检查的。

```go
type cond struct {
 checker copyChecker
}

type copyChecker uintptr

func (c *copyChecker) check() {
 fmt.Printf("Before: c: %12v, *c: %12v, uintptr(*c): %12v, uintptr(unsafe.Pointer(c)): %12v\n", c, *c, uintptr(*c), uintptr(unsafe.Pointer(c)))
 swapped := atomic.CompareAndSwapUintptr((*uintptr)(c), 0, uintptr(unsafe.Pointer(c)))
 fmt.Printf("After : c: %12v, *c: %12v, uintptr(*c): %12v, uintptr(unsafe.Pointer(c)): %12v, swapped: %12v\n", c, *c, uintptr(*c), uintptr(unsafe.Pointer(c)), swapped)
}

func main() {
 var a cond
 a.checker.check()
 b := a
 b.checker.check()
}

// 输出
Before: c: 0xc0000b4008, *c:            0, uintptr(*c):            0, uintptr(unsafe.Pointer(c)): 824634458120
After : c: 0xc0000b4008, *c: 824634458120, uintptr(*c): 824634458120, uintptr(unsafe.Pointer(c)): 824634458120, swapped:         true
Before: c: 0xc0000b4040, *c: 824634458120, uintptr(*c): 824634458120, uintptr(unsafe.Pointer(c)): 824634458176
After : c: 0xc0000b4040, *c: 824634458120, uintptr(*c): 824634458120, uintptr(unsafe.Pointer(c)): 824634458176, swapped:        false
```

这下，sync.Cond的copy检查就很清晰了。当a被b copy之后，uintptr(*c)和uintptr(unsafe.Pointer(c))的值是不同的，通过uint对象的原子比较方法CompareAndSwapUintptr将返回false，它证明了对象a被copy过，从而调用panic保护sync.Cond不被复制。

### go vet checking

上述两个例子都是在程序编译后，runtime检查的。但是，正如文中开篇所述，sync包下的其他的对象如Pool、WaitGroup、Mutex、Map等，它们其实也需要copy检查机制，但是在源码中，却没有提供运行时检查。那该如何保证我们的代码中这些对象在使用中未被copy，从而避免潜在的安全问题呢？

Go在源代码src/sync/cond.go中的一段注释给了我们答案。

```go
// noCopy may be embedded into structs which must not be copied
// after the first use.
//
// See https://golang.org/issues/8005#issuecomment-190753527
// for details.
type noCopy struct{}

// Lock is a no-op used by -copylocks checker from `go vet`.
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
```

很明显，runtime时的copy检查虽然很重要，但是，该操作会影响程序的执行性能。Go官方目前只提供了strings.Builder和sync.Cond的runtime拷贝检查机制，对于其他需要nocopy对象类型来说，使用go vet工具来做静态编译检查。

具体实施来说，就是该对象，或对象中存在filed，它拥有Lock()和Unlock()方法，即实现sync.Locker接口。之后，可以通过go vet功能，来检查代码中该对象是否有被copy。

例如sync.Pool和sync.WaitGroup就内嵌了noCopy属性，sync.Mutex实现了sync.Locker接口，sync.Map内嵌了sync.Mutex。

- 静态检查

```go
// wg.go
package main

import "sync"

func main() {
    var sm sync.Mutex
    sm.Lock()
    sm.Unlock()
    sm2 := sm
    sm2.Lock()
}
```

如上，sm在first use后，被copy给sm2。注意：该代码运行时，不会报错，但是却存在安全隐患。

```go
 $ go vet wg.go
 # command-line-arguments
./wg.go:9:9: assignment copies lock value to sm2: sync.Mutex
```

通过以上命令，即可检查出sync.Mutex有被copy。因此，举一反三，如果在我们自己的项目开发中，定义某对象不能被copy，那么就可以参考Go源码中，嵌入noCopy结构体，最终通过go vet进行copy检查。

```go
type noCopy struct{}
func (*noCopy) Lock() {}
func (*noCopy) Unlock() {}
type MyType struct {
   noCopy noCopy
   ...
}
```


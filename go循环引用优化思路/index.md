# go循环引用优化思路


### 前言

golang是一门极具设计哲学的语言，其中一点是没有支持包循环导入，目的是迫使 Go 程序员更多地考虑程序的依赖关系。一方面保持依赖关系图的简洁，二是快速的程序构建。

通常来说，只要你的包规划得好，严格规范单向调用链（如控制层->业务层->数据层），一般不会出现包循环引用问题。当然现实业务往往不会这么理想，同层级之间的不同包经常需要互相引用，下面我就分享几种解决包循环引用的方案。

### 抽象接口

- package_i

```go
package package_i
 
type PackageAInterface interface {
 PrintA()
}
 
type PackageBInterface interface {
 PrintB()
}
```

- package_a

```go
package package_a
 
import (
 "cycle/package_i"
 "fmt"
)
 
type PackageA struct {
 B package_i.PackageBInterface
}
 
func (a PackageA) PrintA() {
 fmt.Println("I'm a!")
}
 
func (a PackageA) PrintAll() {
 a.PrintA()
 a.B.PrintB()
}
```

- package_b

```go
package package_b
 
import (
 "cycle/package_i"
 "fmt"
)
 
type PackageB struct {
 A package_i.PackageAInterface
}
 
func (b PackageB) PrintB() {
 fmt.Println("I'm b!")
}
 
func (b PackageB) PrintAll() {
 b.PrintB()
 b.A.PrintA()
}
```

- main

```go
package main
 
import (
 "cycle/package_a"
 "cycle/package_b"
)
 
func main() {
 a := new(package_a.PackageA)
 b := new(package_b.PackageB)
 a.B = b
 b.A = a
 a.PrintAll()
 b.PrintAll()
}
```

## 三、新建公共组合包（子包），在组合包中组合调用

- package_c

```go
package package_c
 
import (
	"cycle/package_a"
	"cycle/package_b"
)
 
type CombileAB struct {
	A *package_a.PackageA
	B *package_b.PackageB
}
 
func (c CombileAB) PrintAll() {
	c.A.PrintA()
	c.B.PrintB()
}
```

- main

```go
package main
 
import (
 "cycle/package_a"
 "cycle/package_b"
 "cycle/package_c"
)
 
func main() {
 a := new(package_a.PackageA)
 b := new(package_b.PackageB)
 c := new(package_c.CombileAB)
 c.A = a
 c.B = b
 c.PrintAll()
}
```

## 四、全局存储需要相互依赖的函数，通过关键字进行调用

- callback_mgr

```go
package callback_mgr
 
import (
 "fmt"
 "reflect"
)
 
var callBackMap map[string]interface{}
 
func init() {
 callBackMap = make(map[string]interface{})
}
 
func RegisterCallBack(key string, callBack interface{}) {
 callBackMap[key] = callBack
}
 
func CallBackFunc(key string, args ...interface{}) []interface{} {
 if callBack, ok := callBackMap[key]; ok {
 in := make([]reflect.Value, len(args))
 for i, arg := range args {
 in[i] = reflect.ValueOf(arg)
 }
 outList := reflect.ValueOf(callBack).Call(in)
 result := make([]interface{}, len(outList))
 for i, out := range outList {
 result[i] = out.Interface()
 }
 return result
 } else {
 panic(fmt.Errorf("callBack(%s) not found", key))
 }
}
```

- package_a

```go
package package_a
 
import (
 "cycle/callback_mgr"
 "fmt"
)
 
func init() {
 callback_mgr.RegisterCallBack("getA", new(PackageA).GetA)
}
 
type PackageA struct {
}
 
func (a PackageA) GetA() string {
 return "I'm a!"
}
 
func (a PackageA) PrintAll() {
 fmt.Println(a.GetA())
 fmt.Println(callback_mgr.CallBackFunc("getB")[0].(string))
}
```

- package_b

```go
package package_b
 
import (
 "cycle/callback_mgr"
 "fmt"
)
 
func init() {
 callback_mgr.RegisterCallBack("getB", new(PackageB).GetB)
}
 
type PackageB struct {
}
 
func (b PackageB) GetB() string {
 return "I'm b!"
}
 
func (b PackageB) PrintAll() {
 fmt.Println(b.GetB())
 fmt.Println(callback_mgr.CallBackFunc("getA")[0].(string))
}
```

- main

```go
package main
 
import (
 "cycle/package_a"
 "cycle/package_b"
)
 
func main() {
 a := new(package_a.PackageA)
 b := new(package_b.PackageB)
 a.PrintAll()
 b.PrintAll()
}
```

### 异步调用解耦

对于不需要执行结果的方法调用，可以采用事件总线进行异步调用。

- eventBus

```go
package eventBus
 
import (
 "github.com/asaskevich/EventBus"
)
 
var globalEventBus EventBus.Bus
 
func init() {
 globalEventBus = EventBus.New()
}
 
func Subscribe(topic string, fn interface{}) error {
 return globalEventBus.Subscribe(topic, fn)
}
 
func SubscribeAsync(topic string, fn interface{}, transactional bool) error {
 return globalEventBus.SubscribeAsync(topic, fn, transactional)
}
 
func Publish(topic string, args ...interface{}) {
 globalEventBus.Publish(topic, args...)
}
```

- package_a

```go
package package_a
 
import (
 "cycle/eventBus"
 "fmt"
)
 
func init() {
 eventBus.Subscribe("PrintA", new(PackageA).PrintA)
}
 
type PackageA struct {
}
 
func (a PackageA) PrintA() {
 fmt.Println("I'm a!")
}
 
func (a PackageA) PrintAll() {
 a.PrintA()
 eventBus.Publish("PrintB")
}
```

- package_b

```go
package package_b
 
import (
 "cycle/eventBus"
 "fmt"
)
 
func init() {
 eventBus.Subscribe("PrintB", new(PackageB).PrintB)
}
 
type PackageB struct {
}
 
func (b PackageB) PrintB() {
 fmt.Println("I'm b!")
}
 
func (b PackageB) PrintAll() {
 b.PrintB()
 eventBus.Publish("PrintA")
}
```


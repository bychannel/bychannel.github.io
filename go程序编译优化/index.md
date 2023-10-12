# go程序编译优化


## gcflags

go build 可以用 `-gcflags` 给 go 编译器传入参数，也就是传给 `go tool compile` 的参数，因此可以用 `go tool compile --help` 查看所有可用的参数。

常⽤参数

* -m 开启内存分配分析(逃逸)。可以帮助您找到可能存在的内存分配问题

* -N 禁⽤优化 (debug时⽤到）可以⽤于⽣成调试信息

* -l 禁⽌内联优化 (debug时⽤到）

* -L 错误信息中打印⽂件全名

gcflag传⼊的⽅式为： -gcflag="pattern= args",其中pattern代表取值分别为 main,all,std,...,⽤于指定编译参数作⽤的范围，args则为对应的编译参数

如果只在编译特定包时需要传递参数，格式应遵守“包名=参数列表”，如go build -gcflags -gcflags='log=-N -l' main.go

-gcflags "-N -l" 选项⽬的是在编译过程中禁⽌内联优化，加快编译速度减少开销

## Pattern

* main 指 main 函数所在的顶级包路径

* all 表⽰ GOPATH 中的所有包。如果在 modules 模式下，则表⽰主模块和它所有的依赖，包括 test ⽂件的依赖

* std 表⽰ Go 标准库中的所有包

## ldflags

go build用 `-ldflags` 给go链接器传入参数，实际是给 `go tool link` 的参数，可以用 `go tool link --help` 查看可用的参数。

* -w 去掉调试信息

* -s 去掉符号表，panic时候的stack trace就没有任何⽂件名/⾏号信息了

* -X 注⼊变量, 编译时赋值

-w 和 -s 通常⼀起使⽤，⽤来减少可执⾏⽂件的体积。但删除了调试信息后，可执⾏⽂件将⽆法使⽤ gdb/dlv 调试

-ldflags "-w -s" 选项⽬的是获取最⼩的⼆进制⽂件

-X 来指定版本号等编译时才决定的参数值，例如程序版本号

## 项⽬使⽤

* Develop 环境

```go
-gcflags "-l -N -m"
-ldflags ""
```

* Release 环境

```go
-gcflags ""
-ldflags "-w -s"
```


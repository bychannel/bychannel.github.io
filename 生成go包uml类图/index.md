# 如何生成go包的uml类图


GoLand内置的Diagrams功能不像IDEA一样强大，不支持生成类图。解决方案是利用github上的 PlantUML 类图生成器：[jfeliu007/goplantuml](https://github.com/jfeliu007/goplantuml) ，结合Goland的PlantUML插件，对于golang源码项目的类图生成足够用了。

### 安装goplantuml

```shell
go install github.com/jfeliu007/goplantuml/cmd/goplantuml@latest
```

将goplantuml可执行文件安装到本地`$GOBIN`目录下



### 集成goplantuml到goland

我们安装好可执行文件后，可以直接使用命令进行生成uml文件，诸如：

```shell
goplantuml [-recursive] path/to/gofiles path/to/gofiles2
或
goplantuml [-recursive] path/to/gofiles path/to/gofiles2 > diagram_file_name.puml
```

当然，这里会很麻烦，每次都需要打开命令行，然后定位到对应的路径下，敲一大串命令才能得到uml文件。这里，我们可以借助goland的`External Tools`，将命令集成到goland菜单中，我们需要使用的时候直接点击菜单按钮就生成了，非常快捷。

首先，打开`settings/Tools/External Tools`，点击＋号新增。

Name: `goplanuml`，或者随意取名；

Program：`C:\Users\wujunjie\go\bin\goplantuml.exe`，安装的goplantuml位置，请替换成你的GOBIN路径；

Arguments: `-recursive --output=$FileDir$.puml $FileDir$`，这样应该够用了，当然你也可以使用 `goplantuml -h` 查看更多命令；

working directory：`.`，当前路径就行。

保存后，选中一个go包，然后点击菜单按钮，输出了对应的uml文件表示成功。



### 安装PlantUML插件

生成uml文件了还是无法可视化查看，这里借助goland插件来生成类图，插件选择 `PlantUML Integration` ，直接在goland插件市场中安装即可。



### 其他

此外，作者还开发了一个在线版本的uml生成器 [www.dumels.com](http://www.dumels.com/)，可以直接生成github仓库的源码uml图，非常的方便，可以先行体验，当然，给的源码包层次不能太深了，否则一些大型项目生成会失败。


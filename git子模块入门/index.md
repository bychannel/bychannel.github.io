# 【转载】git子模块入门


> 注：本文转载自 [Git: submodule 子模块简明教程 | A Quest After Perspectives](https://iphysresearch.github.io/blog/post/programing/git/git_submodule/)
> 
> 如有侵权，联系删除

有种情况我们经常会遇到：某个工作中的项目需要包含并使用另一个项目。 也许是第三方库，或者你独立开发的，用于多个父项目的库。 现在问题来了：你想要把它们当做两个独立的项目，同时又想在一个项目中使用另一个。

Git 通过子模块来解决这个问题。 子模块允许你将一个 Git 仓库作为另一个 Git 仓库的子目录。 它能让你将另一个仓库克隆到自己的项目中，同时还保持提交的独立。

---

## 添加子模块

添加一个远程仓库项目 `https://github.com/iphysresearch/GWToolkit.git` 子模块到一个已有主仓库项目中。代码形式是 `git submodule add <url> <repo_name>`， 如下面的例子：

```bash
$ git submodule add https://github.com/iphysresearch/GWToolkit.git GWToolkit
```

这时，你会看到一个名为 `GWToolkit` 的文件夹在你的主仓库目录中。

> 如果你是旧版 Git 的话，你会发现 `./GWToolkit` 目录中是空的，你还需要在执行一步「更新子模块」，才可以把远程仓库项目中的内容下载下来。
> 
> ```bash
> $ git submodule update --init --recursive
> ```
> 
> 如果你不小心把路径写错了，可以用下面的代码来删掉，详细可查阅 `git help submodule`。
> 
> ```bash
> $ git rm --cached GWToolkit
> ```

添加子模块后，若运行 `git status`，可以看到主仓库目录中会增加一个文件 `.gitmodules`，这个文件用来保存子模块的信息。

```bash
$ git status
位于分支 main
您的分支与上游分支 'origin/main' 一致。

要提交的变更：
  （使用 "git restore --staged <文件>..." 以取消暂存）
    新文件：   .gitmodules
    新文件：   GWToolkit
```

另外，在 `.git/config` 中会多出一块关于子模块信息的内容：

```v
[submodule "GWToolkit"]
        url = https://github.com/iphysresearch/GWToolkit.git
        active = true
```

该配置文件保存了项目 URL 与已经拉取的本地目录之间的映射。如果有多个子模块，该文件中就会有多条记录。 要重点注意的是，该文件也像 `.gitignore` 文件一样受到（通过）版本控制。 它会和该项目的其他部分一同被拉取推送。 这就是克隆该项目的人知道去哪获得子模块的原因。

新生成的还有相关子模块的文件：`.git/modules/GWToolkit/`。

此时若把上述「添加子模块」的修改更新到主仓库的 GitHub 上去的话，会看到相应子模块仓库的文件夹图标会有些不同：

![](https://vip2.loli.io/2021/08/28/ktpsTaGFljSKfOc.png)

此时还要留意的是，在终端 Git 命令操作下，位于主仓库目录中除了子模块外的任何子目录下进行的 commit 操作，都会记到主仓库下。只有在子模块目录内的任何 commit 操作，才会记到子模块仓库下。如下面的示例：

```bash
$ cd ~/projects/<module>
$ git log # log shows commits from Project <module>
$ cd ~/projects/<module>/<sub_dir>
$ git log # still commits from Project <module>
$ cd ~/projects/<module>/<submodule>
$ git log # commits from <submodule>
```

---

## 查看子模块

```bash
$ git submodule
 13fe233bb134e25382693905cfb982fe58fa94c9 GWToolkit (heads/main)
```

---

## 更新子模块

更新项目内子模块到最新版本：

```bash
$ git submodule update
```

更新子模块为远程项目的最新版本

```bash
$ git submodule update --remote
```

---

## `Clone` 包含子模块的项目

对于你的主仓库项目合作者来说，如果只是 `git clone` 去下载主仓库的内容，那么你会发现子模块仓库的文件夹内是空的！

此时，你可以像上面「添加子模块」中说到的使用 `git submodule update --init --recursive` 来递归的初始化并下载子模块仓库的内容。

也可以分初始化和更新子模块两步走的方式来下载子模块仓库的内容：

```bash
$ git submodule init        # 初始化子模块
$ git submodule update    # 更新子模块
```

但是，如果你是第一次使用 `git clone` 下载主仓库的所有项目内容的话，我建议你可以使用如下的代码格式来把主仓库和其中子模块的所有内容，都**一步到位**的下载下来：

```bash
$ git clone --recursive <project url>
```

以后可以在子模块仓库目录下使用 `git pull origin main` 或者 `git push` 等来进行更新与合并等操作。

---

## 删除子模块

**删除子模块比较麻烦，需要手动删除相关的文件，否则在添加子模块时有可能出现错误** 同样以删除 `GWToolkit` 子模块仓库文件夹为例：

1. 删除子模块文件夹
   
   ```bash
   $ git rm --cached GWToolkit
   $ rm -rf GWToolkit
   ```

2. 删除 `.gitmodules` 文件中相关子模块的信息，类似于：
   
   ```v
   [submodule "GWToolkit"]
          path = GWToolkit
          url = https://github.com/iphysresearch/GWToolkit.git
   ```

3. 删除 `.git/config` 中相关子模块信息，类似于：
   
   ```v
   [submodule "GWToolkit"]
          url = https://github.com/iphysresearch/GWToolkit.git
          active = true
   ```

4. 删除 `.git` 文件夹中的相关子模块文件
   
   ```bash
   $ rm -rf .git/modules/GWToolkit
   ```

---

## 最后的话

- 虽然 Git 提供的子模块功能已足够方便好用，但仍请在为主仓库项目添加子模块之前确保这是非常必要的。毕竟有很多编程语言（如 [Go](https://golang.org/)）或其他依赖管理工具（如 Ruby’s [rubygems](http://guides.rubygems.org/), Node.js’ [npm](https://docs.npmjs.com/getting-started/what-is-npm), or Cocoa’s [CocoaPods](https://cocoapods.org/about) and [Carthage](https://github.com/Carthage/Carthage)）可以更好的 handle 类似的功能。
- 主仓库项目的合作者并不会**自动地**看到子模块仓库的更新通知的。所以，更新子模块后一定要记得提醒一下主仓库项目的合作者 `git submodule update`。


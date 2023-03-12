# 自动化博客搭建记录


这里记录一下hugo静态博客的搭建过程，本文假设你已经了解hugo、github、github pages的相关知识，不清楚可以先去google了解一下。

我的方案主要分为以下几个核心部分：

1. 个人博客源仓库，对博客配置及所有文章 `.md` 源文件进行版本管理，配合 GitHub Action 进行自动化部署，自动生成静态站点推送到 GitHub Pages 博客发布仓库。
2. GitHub Pages 博客发布仓库，以 `username.github.io` 形式命名的仓库，使用 GitHub Pages 实现网站部署，可以通过配置域名 CNAME 解析使用自定义域名。
3. Hugo 主题仓库，fork 喜欢的主题，并对自己的个人定制化改造配置进行版本管理，通过 `git submodule` 的方式链接到个人博客源仓库。

下文会对搭建、本地测试、自动化部署维护等过程进行详细讲解，希望对大家所有帮助。其他组件配置，如网站统计、评论系统等我们后面在慢慢补充。

### 博客源仓库

其实这个就是hugo的源工程，我们最终会把这个源工程提交到git仓库中进行版本管理，这样，我们的文章也就有版本了，见本站的[源工程](https://github.com/bychannel/blog)

#### 前提

在此之前，你现在的环境中需要一下工具：

* hugo，最好是带extend的扩展版本，在编译主题中带有scss文件会用上，[点此处](https://github.com/gohugoio/hugo/releases)

* git，部署到github用到的工具，没有安装的话，直接下最新版本就好了

#### hugo工程

使用命令 `hugo new site blog` 就可以创建初始化一个hugo项目

#### 主题

我们先[点这里](https://themes.gohugo.io)选择一个喜欢的主题，我的做法是fork了主题的github源码，自己如果有想法可以进行自定义修改，然后是把主题添加到hugo工程的themes目录下，这里建议使用子模块的方式进行添加：

```shell
git submodule add https://github.com/bychannel/LoveIt themes/LoveIt
```

进入`themes/LoveIt/exampleSite`，这是主题的使用案例，可以直接复制config.toml文件覆盖hugo工程的配置，然后修改其中的参数，并参照`exampleSite`下的资源路径配置，文章路径，直接照搬使用即可，这样操作下来博客的雏形就好了。

#### 新建文章

现在可以尝试写一篇测试文章看看，直接使用命令`hugo new posts/test.md`就能生成一篇新文章，文件路径在`content/posts`下，或者我们可以直接进入该路径下，复制一个markdown文章进行改写。

文章保存后，使用命令编译一下查看效果：

```shell
hugo server -D
```

打开`localhost:1313`就能进行预览，如果效果不好，我们还可以修改markdown或者toml配置文件，并能够在浏览器实时查看修改后的效果。

### pages仓库

每次使用`hugo server`命令进行编译，会在public目录下生成网站的静态资源，这些html、css等资源就是我们在浏览器中看到的页面了，我们需要单独把他发布出去。

#### 创建仓库

GitHub Pages 项目需要符合 `username.github.io` 的特殊命名格式，仓库建立完成后，可以在设置中配置自己注册的自定义域名来指向 GitHub Pages 生成的网址。此外，需要将博客站点配置文件 `config.toml` 中的 `baseURL` 改为自己的自定义域名，这样博客站点才能正常访问 GitHub Pages 生成的网站服务。

#### 域名配置

如果你购买了域名，不使用默认的`username.github.io`，那你需要在购买域名的后台系统进行域名重定向配置，这里我没有购买域名，就不再进行演示了，自己实操一下应该是没问题的。

#### 自动发布

能自动化处理的事情，我们绝不手动处理，不仅仅是重复的劳动，更是因为手动更加容易出错。

因为我们的博客基于 GitHub 与 GitHub Pages，可以通过官方提供的 GitHub Action 进行 CI 自动发布，下面我会进行详细讲解。GitHub Action 是一个持续集成和持续交付(CI/CD) 平台，可用于自动执行构建、测试和部署管道，目前已经有很多开发好的工作流，可以通过简单的配置即可直接使用。

配置在仓库目录 `.github/workflows` 下，以 `.yml` 为后缀。我的 GitHub Action [配置](https://github.com/bychannel/blog/blob/main/.github/workflows/deploy.yml)，自动发布示例配置如下

```yml
name: deploy

on:
    push:
    workflow_dispatch:
    schedule:
        # Runs everyday at 8:00 AM
        - cron: "0 0 * * *"

jobs:
    build:
        runs-on: ubuntu-latest
        steps:
            - name: Checkout
              uses: actions/checkout@v2
              with:
                  submodules: true
                  fetch-depth: 0

            - name: Setup Hugo
              uses: peaceiris/actions-hugo@v2
              with:
                  hugo-version: "latest"
                  extended: true

            - name: Build Web
              run: hugo

            - name: Deploy Web
              uses: peaceiris/actions-gh-pages@v3
              with:
                  PERSONAL_TOKEN: ${{ secrets.PERSONAL_TOKEN }}
                  EXTERNAL_REPOSITORY: bychannel/bychannel.github.io
                  PUBLISH_BRANCH: main
                  PUBLISH_DIR: ./public
                  commit_message: ${{ github.event.head_commit.message }}
```

`on` 表示 GitHub Action 触发条件，我设置了 `push`、`workflow_dispatch` 和 `schedule` 三个条件：

- `push`，当这个项目仓库发生推送动作后，执行 GitHub Action
- `workflow_dispatch`，可以在 GitHub 项目仓库的 Action 工具栏进行手动调用
- `schedule`，定时执行 GitHub Action，如我的设置为北京时间每天早上执行，主要是使用一些自动化统计 CI 来自动更新我博客的关于页面

`jobs` 表示 GitHub Action 中的任务，我们设置了一个 `build` 任务，`runs-on` 表示 GitHub Action 运行环境，我们选择了 `ubuntu-latest`。我们的 `build` 任务包含了 `Checkout`、`Setup Hugo`、`Build Web` 和 `Deploy Web` 四个主要步骤，其中 `run` 是执行的命令，`uses` 是 GitHub Action 中的一个插件，我们使用了 `peaceiris/actions-hugo@v2` 和 `peaceiris/actions-gh-pages@v3` 这两个插件。其中 `Checkout` 步骤中 `with` 中配置 `submodules` 值为 `true` 可以同步博客源仓库的子模块，即我们的主题模块。

大家可以根据自己的实际情况进行配置改写，然后加入到工程的对应路径下，github会按照配置自动执行。

#### token配置

因为我们需要从一个git仓库push到另一个git仓库，那么需要进行对应的token配置，以便推送的时候能正确的进行权限认证。

首先要在 GitHub 账户下 `Setting - Developer setting - Personal access tokens - Tokens(classic)` 下创建一个 Token。权限需要开启 `repo` 与 `workflow`，选择永不过期。

配置后复制生成的 Token（注：只会出现一次），然后在我们博客源仓库的 `Settings - Secrets and variables - Actions` 中，添加密钥 `PERSONAL_TOKEN` 为刚才的 Token，这样 GitHub Action 就可以获取到 Token 了。

完成上述配置后，推送代码至仓库，即可触发 GitHub Action，自动生成博客页面并推送至 GitHub Pages 仓库。

### 总结

以上就是我通过 Hugo 与 GitHub Action 实现的博客自动部署系统，我自己的实现仓库如下：

博客源码仓库 [https://github.com/bychannel/blog](https://github.com/bychannel/blog)，

主题仓库 [https://github.com/bychannel/LoveIt](https://github.com/bychannel/LoveIt) ，

发布仓库 [https://github.com/bychannel/bychannel.github.io](https://github.com/bychannel/bychannel.github.io)

这样整理下来，我们不仅可以随时修改整个博客，而且流程全自动化，我们只需要专注于产出文章，然后提交到源仓库就可以了。

如果还有不清楚或者没讲明白的地方，大家可以下面评论区提出来讨论一下。


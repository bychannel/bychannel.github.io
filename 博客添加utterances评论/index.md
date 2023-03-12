# 博客添加utterances评论


建立博客以来，试用了多款评论系统，不过最终还是用了 disqus，但是 disqus 令人讨厌的是默认添加很多烦人的广告，牛皮癣一样，而且在国内无法加载，必须翻墙才能显示。[utterances](https://utteranc.es/) 是一款基于 github issues 的评论系统，简洁，避免了 disqus 的广告以及加载问题。这篇文章总结如何给 LoveIt 主题添加 utterances 评论系统。

**注：该主题已经支持了utterances，所以我们只需要简单配置就能使用，如果你使用的主题还不支持，需要按照文档进行代码嵌入**

操作如下：

1. 建立一个 public GitHub 仓库，这个仓库的 issue 用来存放博客评论：如果自己的博客就是用 GitHub pages 建立的，那么就不需要额外建立仓库了，就用博客的仓库也可以。例如我的博客放在 [bychannel/bychannel.github.io](https://github.com/bychannel/bychannel.github.io)这个仓库，就可以用这个仓库存放博客的评论。

2. 安装 [utterances 的 app](https://github.com/apps/utterances)：install 以后，在 `Repository access`，选择 `Only select repositories`，然后选择自己建立的用来存放评论的仓库即可。这时，进入到对应仓库的设置界面，点击 `Integrations`，会看到 utterances 的 app。

3. Hugo 对应的设置：主题已经支持了 utterances，在博客的 `config.toml` 配置文件，加入下面的配置：
   
   ```toml
   [params.page.comment.utterances]
     enable = true
     # owner/repo
     repo = "bychannel/bychannel.github.io"
     issueTerm = "pathname"
     label = "comment"
     lightTheme = "github-light"
     darkTheme = "github-dark"
   ```

4. 重新 deploy Hugo 博客，等博客更新以后，utterances 评论系统即可生效。

要在博客文章下面评论，只需要登录自己的 GitHub 账号即可。


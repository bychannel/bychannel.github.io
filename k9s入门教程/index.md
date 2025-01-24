# k9s入门教程


K9s是一个基于终端的K8s UI管理工具。只需要一个二进制文件，就可以在任意的命令行终端上对K8s进行管理。这个工具的操作逻辑是基于Vim，熟悉Vim的同学几分钟就能上手。

## 安装
K9s支持Linux，macOS和Windows平台。推荐到官方[release](https://github.com/derailed/k9s/releases)界面直接下载二进制文件。
以CentOS为例，选择`k9s_linux_amd64.rpm`下载，然后上传到远程节点机器，使用`rpm -i k9s_linux_amd64.rpm`即可。

与 kubectl 相同，K9s 启动时会读取默认的 kubeconfig。
```
[root@localhost ~]# k9s
```
如果有多个 config 文件，可以通过 --kubeconfig 指定。
```
[root@localhost ~]# k9s --kubeconfig=/srv/.kube/config
```

## 操作演示
下面使用官方提供的deployment来演示一下K9s的常用操作
```
kubectl create -f https://k8s.io/examples/application/deployment.yaml
```
创建好后，我们想要查看这个Deployment的状态。

在终端输入k9s，进入Context的界面。

![](/images/k9s1.png)

通过方向键↑↓或者使用Vim的jk来选择部署的集群，ENTER进入集群。

如果你看到的不是这个界面，你可以输入:ctx，再按下ENTER。这样也能达到上述的界面。

K9s的基本命令与Vim的命令类似，通过冒号加资源类型:resource来定位到不同资源的浏览界面。

Context的界面下一层是Pod。如果你的命名空间中的Pod太多，可以通过 /filter 的方式来过滤，例如输入 /nginx。

![](/images/k9s2.png)

当然你也可以先输入:deploy 跳转到Deployment的界面，选择Nginx对应的Deployment，再按ENTER进入查看Nginx下面所有的Pod。

K9s内置了命令自动补全，例如输入de后，按Tab即可自动补全为deploy。

![](/images/k9s3.png)

此时我们可以看到Pod或者Deployment的状态。如果要看更详细的信息，可以选择快捷键d或者y，查看资源的详细信息或者是资源的Yaml描述文件

![](/images/k9s4.png)

浏览的时候也可以使用Page Down、Page Up、gG跳转到文件头尾等常用Vim操作，这里过滤器仍然适用

查看完毕后使用ESC退回上一层。

如果发现部署的配置不对，可以使用快捷键e进入Vim来修改资源的Yaml定义。

几乎所有资源类型都能进行编辑。如果修改后的语法不正确，K9s会提示修改失败，修改不会生效。

![](/images/k9s5.png)

确认部署成功后我想看Nginx的日志，现在可以通过在Pod界面ENTER进入下一层的日志界面，又或者在Deployment或Pod界面使用快捷键l快速进入日志界面。

因为示例的Nginx只会显示访问日志，这时候界面会提示Waiting for logs....

如果我们快速访问Nginx，该怎么做？在K9s里面有两种方式

第一种是使用Shell，该命令等价于`kubectl exec pod /bin/sh`。先用ESC退回Pod界面，再按下s，就会进入到容器的Shell命令行。如果一个Pod包含了多个Container，则会进入Container界面让你选择要进入的Container。

这种方式是就是当初K9s吸引我使用的地方，对比自己去拼命令，这个要快捷方便很多。

接着输入`curl localhost:80`，但可惜的是镜像没有自带curl。

那只有用第二种方法，port-forward。exit退出命令行界面输入快捷键Shift+f进入PortForward界面。

![](/images/k9s6.png)

这个就是对应`kubectl port-forward`命令。此时使用Tab进行上下切换。确认后本地开一个新的终端访问 `curl localhost:80`。

port-forward在K9s关闭后就会失效

现在回到日志界面就会看到新的日志。

![](/images/k9s7.png)

在日志界面的浏览和Vim浏览也一样，移动方式、过滤和上下跳转等操作都能适用。

我一般在查看容器日志的时候会按 0和 w，0代表查看日志的尾部，即最新日志，w代表日志自动换行。

除了上面的基本功能外，K9s还支持Node Shell(在Node界面按s进入该主机的容器)、xray(目录树的方式展示K8s资源)、压测等等功能，有兴趣可以到官网查看。

以上就是我使用K9s常用到的操作技巧，最后放上K9s命令列表的中文翻译。

## 命令
|动作	|命令|	备注|	
|-------|-------|-------|
|快捷键记录界面|	?	|	|
|显示集群所有的资源和它们的缩写|	ctrl-a 或 :alias	|例如service的缩写是svc	|
|退出	|:q 或 ctrl-c	|	
|用资源名或者缩写来浏览某一类资源|	:po	|接受单数、复数、缩写或者别名。例如po,pod,pods,v1/pods	|
|在指定的ns里面浏览资源|	:po namespace		|
|过滤某种资源	|/filter	|支持Regex2标准，例如`fred	blee`过滤名字是fred或者是blee的资源|
|逆过滤器|	/!filter|	去除所有匹配的资源。不支持日志过滤	|
|根据labels过滤资源|	/-l label-selector	|	
|模糊匹配过滤	|/-f filter|		
|退出 浏览/命令/过滤 模式|	esc		|
|常用快捷键，describe, view,edit,view logs,…	|d,v,e,l,…		|
|切换集群上下文	|:ctx	|	
|切换集群上下文|	:ctx context-name	|	
|切换namespace|	:ns |		
|浏览所有保存的资源	|:screendump or sd	|	
|删除资源(TAB和ENTER来确认)|	ctrl-d		|
|终止一种资源(不会有确认的对话框)|	ctrl-k		|
|扩宽显示栏|	ctrl-w	|等价于kubectl ... -o wide	|
|浏览在错误状态的资源|	ctrl-z	|	
|增强界面	|:pulses or pu |	用命令行实现类似GUI的监控界面	|
|XRay界面|	:xray RESOURCE [NAMESPACE]|	用目录树的结构来展示相关资源	|
|Popeye界面|	:popeye or pop |	Popeye是一个k8s清理工具，帮助找出有潜在问题的资源和配置，详见https://popeyecli.io/ |



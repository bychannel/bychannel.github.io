# Containerd配置镜像加速


众所周知，长城防火墙的存在使的k8s镜像加载变得异常困难，不使用特殊手段几乎是无法正常拉取到镜像，k8s1.24以后的默认容器运行时是containerd，也就是说k8s会直接使用containerd去拉取镜像，并启动容器，而不再是使用docker去拉取镜像，所以我们需要配置containerd的镜像加速器。
下面记录一下相关的配置方法：

## 旧版本配置方法
在containerd1.5版本前，配置方法如下：

### 1. 生成containerd配置文件
如果配置文件不存在，则执行下面这条命令，否则跳过
```
containerd config default > /etc/containerd/config.toml
```

### 2. 修改配置文件
```
vim /etc/containerd/config.toml
```
需要找到这一行，并添加2行
```
[plugins."io.containerd.grpc.v1.cri".registry.mirrors]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
    endpoint = ["镜像加速器地址1","镜像加速器地址2"]
```
写完之后重启containerd就可以了

## 新版本配置方法

新版本的containerd镜像仓库配置都是建议放在一个单独的文件夹当中，并且在/etc/containerd/config.toml配置文件当中打开config_path配置，指向镜像仓库配置目录即可。这种方式只需要在第一次修改/etc/containerd/config.toml文件打开config_path配置时需要重启containerd，后续我们增加镜像仓库配置都无需重启containerd，非常方便。

官方文档：[https://github.com/containerd/containerd/blob/v1.6.24/docs/cri/registry.md](https://github.com/containerd/containerd/blob/v1.6.24/docs/cri/registry.md)

### 1. 修改config.toml
```
[root@master1 ~]# vim /etc/containerd/config.toml

# 修改内容
[plugins."io.containerd.grpc.v1.cri".registry]
   config_path = "/etc/containerd/certs.d"
```

### 2. 配置镜像加速
```
# docker hub镜像加速
mkdir -p /etc/containerd/certs.d/docker.io
cat > /etc/containerd/certs.d/docker.io/hosts.toml << EOF
server = "https://docker.io"
[host."https://dockerproxy.com"]
  capabilities = ["pull", "resolve"]

[host."https://docker.m.daocloud.io"]
  capabilities = ["pull", "resolve"]

[host."https://reg-mirror.qiniu.com"]
  capabilities = ["pull", "resolve"]

[host."https://registry.docker-cn.com"]
  capabilities = ["pull", "resolve"]

[host."http://hub-mirror.c.163.com"]
  capabilities = ["pull", "resolve"]

EOF
```

```
# registry.k8s.io镜像加速
mkdir -p /etc/containerd/certs.d/registry.k8s.io
tee /etc/containerd/certs.d/registry.k8s.io/hosts.toml << 'EOF'
server = "https://registry.k8s.io"

[host."https://k8s.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF
```
```
# docker.elastic.co镜像加速
mkdir -p /etc/containerd/certs.d/docker.elastic.co
tee /etc/containerd/certs.d/docker.elastic.co/hosts.toml << 'EOF'
server = "https://docker.elastic.co"

[host."https://elastic.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF
```
```
# gcr.io镜像加速
mkdir -p /etc/containerd/certs.d/gcr.io
tee /etc/containerd/certs.d/gcr.io/hosts.toml << 'EOF'
server = "https://gcr.io"

[host."https://gcr.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF
```
```
# ghcr.io镜像加速
mkdir -p /etc/containerd/certs.d/ghcr.io
tee /etc/containerd/certs.d/ghcr.io/hosts.toml << 'EOF'
server = "https://ghcr.io"

[host."https://ghcr.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF
```
```
# k8s.gcr.io镜像加速
mkdir -p /etc/containerd/certs.d/k8s.gcr.io
tee /etc/containerd/certs.d/k8s.gcr.io/hosts.toml << 'EOF'
server = "https://k8s.gcr.io"

[host."https://k8s-gcr.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF
```
```
# mcr.microsoft.com镜像加速
mkdir -p /etc/containerd/certs.d/mcr.microsoft.com
tee /etc/containerd/certs.d/mcr.microsoft.com/hosts.toml << 'EOF'
server = "https://mcr.microsoft.com"

[host."https://mcr.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF
```
```
# nvcr.io镜像加速
mkdir -p /etc/containerd/certs.d/nvcr.io
tee /etc/containerd/certs.d/nvcr.io/hosts.toml << 'EOF'
server = "https://nvcr.io"

[host."https://nvcr.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF
```
```
# quay.io镜像加速
mkdir -p /etc/containerd/certs.d/quay.io
tee /etc/containerd/certs.d/quay.io/hosts.toml << 'EOF'
server = "https://quay.io"

[host."https://quay.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF
```
```
# registry.jujucharms.com镜像加速
mkdir -p /etc/containerd/certs.d/registry.jujucharms.com
tee /etc/containerd/certs.d/registry.jujucharms.com/hosts.toml << 'EOF'
server = "https://registry.jujucharms.com"

[host."https://jujucharms.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF
```
```
# rocks.canonical.com镜像加速
mkdir -p /etc/containerd/certs.d/rocks.canonical.com
tee /etc/containerd/certs.d/rocks.canonical.com/hosts.toml << 'EOF'
server = "https://rocks.canonical.com"

[host."https://rocks-canonical.m.daocloud.io"]
  capabilities = ["pull", "resolve", "push"]
EOF
```

### 3. 重启containerd
```
[root@master1 ~]# systemctl restart containerd
[root@master1 ~]# systemctl status containerd
```

### 4. 验证
crictl 命令， 会自动使用/etc/containerd/certs.d目录下的配置镜像加速 （推荐）
使用crictl 命令拉取
```
[root@node2 certs.d]# crictl --debug=true  pull docker.io/library/ubuntu:20.04
DEBU[0000] get image connection
DEBU[0000] PullImageRequest: &PullImageRequest{Image:&ImageSpec{Image:docker.io/library/ubuntu:20.04,Annotations:map[string]string{},UserSpecifiedImage:,RuntimeHandler:,},Auth:nil,SandboxConfig:nil,}
DEBU[0022] PullImageResponse: &PullImageResponse{ImageRef:sha256:ba6acccedd2923aee4c2acc6a23780b14ed4b8a5fa4e14e252a23b846df9b6c1,}
Image is up to date for sha256:ba6acccedd2923aee4c2acc6a23780b14ed4b8a5fa4e14e252a23b846df9b6c1
[root@node2 certs.d]# crictl images
IMAGE                                        TAG                    IMAGE ID            SIZE
docker.io/library/ubuntu                     20.04                  ba6acccedd292       28.6MB
docker.io/rancher/klipper-helm               v0.8.3-build20240228   0929b4140ada6       256MB
docker.io/rancher/klipper-lb                 v0.4.7                 edc812b8e25d0       12.2MB
docker.io/rancher/local-path-provisioner     v0.0.26                c54dcef6214cb       48.7MB
docker.io/rancher/mirrored-coredns-coredns   1.10.1                 ead0a4a53df89       53.6MB
docker.io/rancher/mirrored-library-busybox   1.36.1                 65ad0d468eb1c       4.5MB
docker.io/rancher/mirrored-library-traefik   2.10.7                 ee69e8120b64a       154MB
docker.io/rancher/mirrored-metrics-server    v0.7.0                 b9a5a1927366a       68.2MB
docker.io/rancher/mirrored-pause             3.6                    6270bb605e12e       686kB
```

是对于ctr命令，需要指定--hosts-dir=/etc/containerd/certs.d，举个栗子：
```
ctr i pull --hosts-dir=/etc/containerd/certs.d registry.k8s.io/sig-storage/csi-provisioner:v3.5.0
```
如果要确定此命令是否真的使用了镜像加速，可以增加--debug=true参数，譬如：
```
ctr --debug=true i pull --hosts-dir=/etc/containerd/certs.d registry.k8s.io/sig-storage/csi-provisioner:v3.5.0
```

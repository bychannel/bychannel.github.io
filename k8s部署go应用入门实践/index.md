# k8s部署go应用入门实践


k8s作为目前容器化管理和编排最热门的项目，我们无疑需要马上去学习它，而实践是能够帮助我们快速入门一项技术，在实践的过程中，我们可以理解大致的使用，了解相关的概念，并在过程中遇到大量的问题，从而不断的深入去了解k8s，废话少说，下面我们通过如下几步，可以将自己的go应用部署到k8s上。

## 准备go项目

需要先准备一个go的项目，用于部署到k8s中，然后我们再对项目进行访问，从而对部署结果进行验证。如果自己已经有可以使用的项目了，可以直接使用自己的项目，我用gin创建了一个简单的[演示项目](https://github.com/bychannel/hello-gin)，大家自取。

该项目提供了两个访问接口：

* / 用于返回固定的hello

* /test 用于返回当前的时间

## 编译镜像

接下来，编写Dockerfile将项目编译成一个docker image，内容如下

```dockerfile
FROM golang:1.19.1-alpine as builder
ENV GOPROXY=https://goproxy.cn
WORKDIR /build
COPY . .
RUN go mod tidy
RUN go build -o helloGin main.go

FROM  alpine:latest
RUN mkdir -p /cmd
WORKDIR  /cmd
COPY  --from=builder /build  .
EXPOSE 8888
CMD ["./helloGin"]
```

生成镜像，执行命令

```shell
docker build -t hellogin:1.0.0 -f Dockerfile .
```

执行

```shell
docker images
```

可以看到生成的hellogin:1.0.0镜像已经存在了

## 推送镜像

因为k8s集群的pod在启动的时候，需要先拉取到目标镜像，所以需要把自己的镜像推到镜像服务器，我这里选用阿里的容器镜像服务个人版。自己进行开通此处不再展开，或者使用其他的镜像服务都可以，不影响后续操作。

先登陆->将自己生成的镜像打tag->推到镜像服务器

```shell
$ docker login --username=阿里云账户 registry.cn-shanghai.aliyuncs.com
$ docker tag hellogin:1.0.0 registry.cn-shanghai.aliyuncs.com/命名空间/hellogin:1.0.0
$ docker push registry.cn-shanghai.aliyuncs.com/命名空间/hellogin:1.0.0
```

## 编写部署配置yaml

编写deployment和service配置文件，用于将image镜像如何部署到k8s中。

deployment.yaml文件内容如下：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellogin
spec:
  replicas: 3 # 启动3个pod
  selector:
    matchLabels:
      app: hello-gin
  template:
    metadata:
      labels:
        app: hello-gin
    spec:
      containers:
        - name: go-app-container
          image: registry.cn-shanghai.aliyuncs.com/bysimon/hellogin:1.0.0
          imagePullPolicy: IfNotPresent
          resources:
            limits:
              memory: "512Mi"
              cpu: "500m"
            requests:
              memory: "64Mi"
              cpu: "100m"
          ports:
            - containerPort: 8888 # 程序端口
      imagePullSecrets:
        - name: aliyun-registry # aliyun镜像拉取的秘钥
```

service.yaml内容如下

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hellogin-service
spec:
  type: NodePort
  ports:
    - port: 8888
      nodePort: 30088
  selector:
    app: hello-gin
```

需要对外提供服务所以type选NodePort
port：8888是对应pod的端口号
nodePort：30088是service对外提供的调用端口号

另外，如果使用的是阿里云的镜像服务，在部署之前需要创建镜像拉取的秘钥，否则会拉取镜像失败，导致pod启动失败

```shell
kubectl create secret docker-registry aliyun-registry  \
    --namespace=default \
    --docker-server=registry.cn-shanghai.aliyuncs.com \
    --docker-username=aliyun账户 \
    --docker-password=密码 \
    --docker-email=邮箱(非必选)
```

创建成功后，使用 `kubectl get secrets` 查看。

## 部署到k8s

配置文件准备就绪后，可以开始部署到k8s中，使用命令:

```shell
kubectl create -f ./k8s/deployment.yaml
kubectl create -f ./k8s/service.yaml
```

查看pod和service状况

```shell
kubectl get pod,svc -n default
```

可以看到如下

```shell
NAME                            READY   STATUS    RESTARTS   AGE
pod/hellogin-599968cc59-69qk6   1/1     Running   0          61m
pod/hellogin-599968cc59-bmdkb   1/1     Running   0          61m
pod/hellogin-599968cc59-tnqjz   1/1     Running   0          61m

NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/hellogin-service   NodePort    10.109.104.154   <none>        8888:30088/TCP   60m
service/kubernetes         ClusterIP   10.96.0.1        <none>        443/TCP          69m
```

三个pod(hellogin-xxxxxx-xxxx)处在Running状态，service的Port30088对外提供服务

## 访问确认

查看任何一个node的物理IP地址，通过访问node主机的IP:30088/test会返回当前时间，表示整个流程已经跑通


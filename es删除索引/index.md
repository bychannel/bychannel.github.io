# es删除索引


## 删除指定索引

```shell
# curl -XDELETE -u elastic:changeme http://localhost:9200/acc-apply-2018.08.09

{"acknowledged":true}
```

## 删除多个指定索引

```shell
# 中间用逗号分隔
# curl -XDELETE -u elastic:changeme http://localhost:9200/acc-apply-2018.08.09,acc-apply-2018.08.10
```

## 模糊匹配删除

```shell
# curl -XDELETE -u elastic:changeme http://localhost:9200/acc-apply-*

{"acknowledged":true}
```

## 删除所有的索引

```shell
curl -XDELETE http://localhost:9200/_all
或 curl -XDELETE http://localhost:9200/*
```

_all，* 通配所有的索引，通常不建议使用通配符，误删了后果就很严重了，所有的index都被删除了，为了安全起见，可以在elasticsearch.yml配置文件中设置禁用_all和 * 通配符action.destructive_requires_name = true，这样就不能使用_all和 * 了

## 获取当前索引

```shell
# curl -u elastic:changeme 'localhost:9200/_cat/indices?v'
```

## 定时删除

如果存储不够可以设置定时删除，下面是保留3天的日志

```shell
30 2 * * * /usr/bin/curl -XDELETE -u elastic:changeme http://localhost:9200/*-$(date -d '-3days' +'%Y.%m.%d') >/dev/null 2>&1
```

以下是定时删除脚本：

```shell
#!/bin/bash
time=$(date -d '-3days' +'%Y.%m.%d')
curl -XDELETE -u elastic:changeme http://localhost:9200/*-${time}
```


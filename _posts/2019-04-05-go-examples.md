---
layout: post
title: 用到过的 Go Examples, 持续更新
---

## Go 设置

#### 开启 go mod 和使用 goproxy

使用 goproxy.io 就不用担心一些 go package 被墙了

```
export GO111MODULE=on
export GOPROXY=https://goproxy.io
```

## 数据库

[使用 pq](https://github.com/baya/go-examples/blob/master/pg_example.go)

[使用 mongo-driver ](https://github.com/baya/go-examples/blob/master/mongodb_example.go)

[使用 mysql-driver ](https://github.com/baya/go-examples/blob/master/mysql_example.go)

[使用 go-redis ](https://github.com/baya/go-examples/blob/master/redis_example.go)


## HTML

[goquery 遍历html 节点](https://github.com/baya/go-examples/blob/master/goquery_exmaple.go)

## 分布式服务

[使用 go etcd 发现服务](https://github.com/baya/go-examples/blob/master/etcd_examples) 例子来源于 https://github.com/daizuozhuo/etcd-service-discovery, 我做了一些改动，迁移到了 go mod.


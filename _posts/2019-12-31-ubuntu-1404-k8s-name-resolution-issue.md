---
layout: post
title: ubuntu 14.04 解析 K8S 服务名称问题排查
categories: kubernetes ubuntu
---

最近把项目部署从 docker-compose 迁移到了 Kubernetes 集群, 并且使用集群[DNS 服务](https://kubernetes.io/docs/concepts/services-networking/service/#dns)作为数据库的访问方式。但是应用启动后，日志里有大量的数据库域名无法解析的错误。

```
django.db.utils.OperationalError: could not translate host name "postgres.namespace.svc.cluster.local" to address: No address associated with hostname
```


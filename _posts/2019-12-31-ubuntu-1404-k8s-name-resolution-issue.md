---
title: 记一次 Kubernetes 服务解析问题排查
last_modified_at: 2020-01-02T15:20:02+08:00
classes: wide
categories:
  - 2019-12
tags:
  - kubernetes
  - ubuntu
---

最近把项目部署从 docker-compose 迁移到 Kubernetes 集群后，遇到了域名解析失败的问题。这个问题不大常见，google 能找到的资料不多，所以在这里记录下排障的思路和方法，供他人参考。


## 问题症状

Python Web 应用启动后，日志里有大量数据库域名无法解析的错误:

```
django.db.utils.OperationalError: could not translate host name "postgres.namespace.svc.cluster.local" to address: No address associated with hostname
```

应用使用集群[DNS 服务](https://kubernetes.io/docs/concepts/services-networking/service/#dns) 访问数据库（postgres），服务的部署类型是 ClusterIP。

集群的部署情况大致如下：
一台 master 节点，3台工作节点，操作系统：ubuntu 18.04
kubernetes 版本：v1.15， 使用 rancher 管理，只用作开发测试，集群负载很低。


## 问题初查

启动 `dnstools` pod，通过 nslookup 连续多次测试数据库服务域名，均正常。

```bash
$ kubectl run -it --rm --restart=Never --image=infoblox/dnstools:latest dnstools

dnstools# nslookup postgres
Server:         10.43.155.58
Address:        10.43.155.58#53

Name:   postgres.namespace.svc.cluster.local
Address: 10.43.241.110
```

这样看来，DNS 服务应该没问题。重新再看了下日志，发现除了数据库，rabbitMQ 服务的域名也无法解析，从错误日志看，是在调用 [socket.getaddrinfo](https://github.com/pika/pika/blob/b907f91415169b7f590174ab5d228e75a1b273e6/pika/adapters/base_connection.py#L124) 出错。

OK～看来应该是客户端的问题。接下来写个小脚本测试下连接。

## 测试脚本

```python
import socket

db = 'postgres'
addr = socket.getaddrinfo(db, 5432, 0, socket.SOCK_STREAM, socket.IPPROTO_TCP)
print(addr)
```

连续测试几次后，有意思的问题出现了。域名并不是一直无法解析，而是间断性可以解析，中间间隔30秒。

```bash
root@ubuntu1404-648966b7fd-wx87s:/# python3 sock.py 
[(<AddressFamily.AF_INET: 2>, <SocketKind.SOCK_STREAM: 1>, 6, '', ('10.43.241.110', 5432))]
root@ubuntu1404-648966b7fd-wx87s:/# python3 sock.py 
Traceback (most recent call last):
  File "sock.py", line 5, in <module>
    addr = socket.getaddrinfo(db, 5432, 0, socket.SOCK_STREAM, socket.IPPROTO_TCP)
  File "/usr/lib/python3.4/socket.py", line 533, in getaddrinfo
    for res in _socket.getaddrinfo(host, port, family, type, proto, flags):
socket.gaierror: [Errno -2] Name or service not known
root@ubuntu1404-648966b7fd-wx87s:/# python3 sock.py
[(<AddressFamily.AF_INET: 2>, <SocketKind.SOCK_STREAM: 1>, 6, '', ('10.43.241.110', 5432))]
root@ubuntu1404-648966b7fd-wx87s:/# 
```

30秒内只能解析成功一次？这什么鬼？！像是某种缓存之类的，30秒过期。因为应用跑在容器里，没有额外的服务可以缓存 DNS，应用这边也没有缓存。好吧。。似乎可以排除客户端的问题了。

从个人经验来说，缓存不大可能会影响程序的正确性，还是看看 CoreDNS 的配置和日志吧。

## CoreDNS

```
.:53 {
  errors
  health
  ready
  kubernetes cluster.local in-addr.arpa ip6.arpa {
    pods insecure
    fallthrough in-addr.arpa ip6.arpa
  }
  prometheus :9153
  forward . "/etc/resolv.conf"
  cache 30
  loop
  reload
  loadbalance
}
```

配置里有 `cache 30`,查了下[coredns cache 文档](https://coredns.io/plugins/cache/)

> With cache enabled, all records except zone transfers and metadata records will be cached for up to 3600s. Caching is mostly useful in a scenario when fetching data from the backend (upstream, database, etc.) is expensive.

cache 插件把从上游组件，数据库之类过来的 dns 记录缓存 30 秒，否则每次都重新去获取数据，很影响应用的性能。
按理说从缓存返回的结果应该和正常返回的没什么不同。

接下来只能看下日志了，如果日志里也没有价值的信息，也可以用 tcpdump 抓下包试试，对比解析成功和失败的响应数据包有没有不同。

coredns 默认不开启日志，如果需要开启，在配置里加上 `log` 插件就可以了。但是改配置会影响集群里其他的应用，可以新开个 coredns 服务，把应用容器的 [/etc/resolv.conf nameserver](http://man7.org/linux/man-pages/man5/resolv.conf.5.html) 指向新的服务。

## /etc/resolv.conf

查了下当前的 coredns 版本是 1.3.1, 用 helm 安装该版本：

```bash
$ helm install test-coredns --namespace=default stable/coredns --version v1.3.1
```

然后进入应用容器，修改 /etc/resolv.conf:

```
nameserver 10.43.155.58
search namespace.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

保存后再运行测试脚本，此时，coredns 里有日志输出：

```
2019-12-31T06:19:27.566Z [INFO] 10.42.2.16:40232 - 4189 "A IN postgres.namespace.svc.cluster.local. udp 58 false 512" NOERROR qr,aa,rd 114 0.000427746s
2019-12-31T06:19:27.566Z [INFO] 10.42.2.16:40232 - 47817 "AAAA IN postgres.namespace.svc.cluster.local. udp 58 false 512" NOERROR qr,aa,rd 151 0.000625568s

2019-12-31T06:19:28.193Z [INFO] 10.42.2.16:47156 - 25454 "AAAA IN postgres.namespace.svc.cluster.local. udp 58 false 512" NOERROR qr,rd 151 0.000155542s
2019-12-31T06:19:28.194Z [INFO] 10.42.2.16:47156 - 30511 "A IN postgres.namespace.svc.cluster.local. udp 58 false 512" NOERROR qr,rd 114 0.000374902s
2019-12-31T06:19:28.195Z [INFO] 10.42.2.16:49445 - 20345 "AAAA IN postgres.namespace.svc.cluster.local. udp 58 false 512" NOERROR qr,rd 151 0.00030268s
2019-12-31T06:19:28.195Z [INFO] 10.42.2.16:49445 - 25967 "A IN postgres.namespace.svc.cluster.local. udp 58 false 512" NOERROR qr,rd 114 0.000448206s
2019-12-31T06:19:28.195Z [INFO] 10.42.2.16:44987 - 30511 "A IN postgres.namespace.svc.cluster.local. udp 58 false 512" NOERROR qr,rd 114 0.000720891s
2019-12-31T06:19:28.196Z [INFO] 10.42.2.16:44987 - 25454 "AAAA IN postgres.namespace.svc.cluster.local. udp 58 false 512" NOERROR qr,rd 151 0.001649s
2019-12-31T06:19:28.196Z [INFO] 10.42.2.16:59892 - 25967 "A IN postgres.namespace.svc.cluster.local. udp 58 false 512" NOERROR qr,rd 114 0.000335876s
2019-12-31T06:19:28.197Z [INFO] 10.42.2.16:59892 - 20345 "AAAA IN postgres.namespace.svc.cluster.local. udp 58 false 512" NOERROR qr,rd 151 0.000172475s
```

前两行是执行第一次成功的日志输出，后面几行是下一次失败的日志结果。
从日志里看，服务端返回的结果都是 `NOERROR`，也就是说 coredns 都成功返回了。但是客户端似乎不认可从 cache 返回的结果，又重试了3次才放弃。
另外，从 cache 返回的响应头部只有 `qr,rd`, 没有设置 `aa` 返回位。难道是这个原因？

查了下[rfc文档](https://tools.ietf.org/html/rfc1035)和一些[博客资料](https://danielmiessler.com/study/dns/#authoritative)：

> Authoritative vs. Non-authoritative Responses
> Authoritative responses are responses that come directly from a nameserver that has authority over the record in question. Non-authoritative answers come second-hand (or more), i.e., from another server or through a cache.

从 cache 里返回的结果确实不应该设置`aa`位。
或许我应该看看 python `socket.getaddrinfo` 的源码?


## getaddrinfo

看了下 python socket 模块的[代码](https://github.com/python/cpython/blob/3.6/Modules/socketmodule.c#L6061)，它实际调用的是系统函数 [getaddrinfo](http://man7.org/linux/man-pages/man3/getaddrinfo.3.html)。所以，不是 python 应用的问题，而是 getaddrinfo 这个系统函数的问题。

于是写了一段小程序，测试下 `getaddrinfo`:

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <netdb.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>

int
lookup_host (const char *host)
{
  struct addrinfo hints, *res;
  int errcode;
  char addrstr[100];
  void *ptr;

  memset (&hints, 0, sizeof (hints));
  hints.ai_family = PF_UNSPEC;
  hints.ai_socktype = SOCK_STREAM;
  hints.ai_flags |= AI_CANONNAME;

  errcode = getaddrinfo (host, NULL, &hints, &res);
  if (errcode != 0)
    {
      printf("err code: %d\n", errcode);
      printf("err: %s\n", gai_strerror(errcode));
      return -1;
    }

  printf ("Host: %s\n", host);
  while (res)
    {
      inet_ntop (res->ai_family, res->ai_addr->sa_data, addrstr, 100);

      switch (res->ai_family)
        {
        case AF_INET:
          ptr = &((struct sockaddr_in *) res->ai_addr)->sin_addr;
          break;
        case AF_INET6:
          ptr = &((struct sockaddr_in6 *) res->ai_addr)->sin6_addr;
          break;
        }
      inet_ntop (res->ai_family, ptr, addrstr, 100);
      printf ("IPv%d address: %s (%s)\n", res->ai_family == PF_INET6 ? 6 : 4,
              addrstr, res->ai_canonname);
      res = res->ai_next;
    }

  return 0;
}

int main (void)
{
  char inbuf[] = "postgres.namespace.svc.cluster.local";

  lookup_host (inbuf);
}
```

执行结果：

```bash
root@ff7587c75-5wllf:/home/ubuntu# ./a.out
Host: postgres.namespace.svc.cluster.local
IPv4 address: 10.43.241.110 (postgres.namespace.svc.cluster.local)
root@ff7587c75-5wllf:/home/ubuntu# ./a.out
err code: -5
err: No address associated with hostname
```

OK 应该是 glibc 版本太老了，应用镜像当前是基于 ubuntu 14.04， glibc 版本 2.19。于是把基础镜像改成 ubuntu 18.04 后，这个问题就解决了。


## 额外收获

1. 在测试 coredns 时候，发现最新版（1.7.3）并没有这个问题。后来多次降级后，发现在 [1.5.1](https://coredns.io/2019/06/26/coredns-1.5.1-release/) 里修改了代码，cache 返回的响应里也设置 `aa` 位，原因可以参考[这个PR](https://github.com/coredns/coredns/pull/2885)。 当然，目前这种方式只是一种 hack，为了兼容老的系统，后续可能会再调整。我也顺带帮作者完善了下[代码注释](https://github.com/coredns/coredns/pull/3573)。

2. [这篇文章](https://pracucci.com/kubernetes-dns-resolution-ndots-options-and-why-it-may-affect-application-performances.html)介绍了 `ndots:5` 对性能的影响。总的来说，如果 `/etc/resolv.conf`配置了 `ndots:5`, 那么使用 service name `postgres` 或者 fully qualified name `postgres.namespace.svc.cluster.local.`（注意最后的点）。否则系统会进行多次无效的 dns 查询，影响应用的性能。


## 阅读材料

- [https://discover.curve.app/a/mind-of-a-problem-solver](https://discover.curve.app/a/mind-of-a-problem-solver)

- [https://tech.xing.com/a-reason-for-unexplained-connection-timeouts-on-kubernetes-docker-abd041cf7e02](https://tech.xing.com/a-reason-for-unexplained-connection-timeouts-on-kubernetes-docker-abd041cf7e02)

- [https://blog.quentin-machu.fr/2018/06/24/5-15s-dns-lookups-on-kubernetes/](https://blog.quentin-machu.fr/2018/06/24/5-15s-dns-lookups-on-kubernetes/)

- [https://sysdig.com/blog/understanding-how-kubernetes-services-dns-work/](https://sysdig.com/blog/understanding-how-kubernetes-services-dns-work/)

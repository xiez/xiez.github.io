---
title: Helm command cheatsheet
classes: wide
categories:
  - 2019-12
tags:
  - helm
  - cheatsheet
---

This post is a cheatsheet of the `helm` commands. [Helm](https://helm.sh/) is a package manager for Kubernetes, like apt/yum/homebrew for Kubernetes.

## .bash_aliases

```bash
$ alias h='helm'
```

## Basic

### Add repo

```bash
# use custom repo in China, see https://github.com/cloudnativeapp/charts
$ h repo add apphub https://apphub.aliyuncs.com
```

### Search a package

```bash
$ h search repo mariadb
```

### Install a package

```bash
$ h install happy-panda apphub/mariadb

$ h install my-coredns --namespace=default stable/coredns --version v1.7.3
```

**output:**

```bash
NAME: happy-panda
LAST DEPLOYED: Tue Dec  3 15:28:49 2019
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
Please be patient while the chart is being deployed

Tip:

  Watch the deployment status using the command: kubectl get pods -w --namespace default -l release=happy-panda

Services:

  echo Master: happy-panda-mariadb.default.svc.cluster.local:3306
  echo Slave:  happy-panda-mariadb-slave.default.svc.cluster.local:3306

Administrator credentials:

  Username: root
  Password : $(kubectl get secret --namespace default happy-panda-mariadb -o jsonpath="{.data.mariadb-root-password}" | base64 --decode)

To connect to your database:

  1. Run a pod that you can use as a client:

      kubectl run happy-panda-mariadb-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mariadb:10.3.20-debian-9-r0 --namespace default --command -- bash

  2. To connect to master service (read/write):

      mysql -h happy-panda-mariadb.default.svc.cluster.local -uroot -p my_database

  3. To connect to slave service (read-only):

      mysql -h happy-panda-mariadb-slave.default.svc.cluster.local -uroot -p my_database

To upgrade this helm chart:

  1. Obtain the password as described on the 'Administrator credentials' section and set the 'rootUser.password' parameter as shown below:

      ROOT_PASSWORD=$(kubectl get secret --namespace default happy-panda-mariadb -o jsonpath="{.data.mariadb-root-password}" | base64 --decode)
      helm upgrade happy-panda stable/mariadb --set rootUser.password=$ROOT_PASSWORD
```

### Release status

```bash
$ h status happy-panda
```

### Show values

```bash
$ h show values apphub/mariadb
```

### Override values

```bash
$ echo '{mariadbUser: user0, mariadbDatabase: user0db}' > config.yaml
$ h install happy-panda -f config.yaml apphub/mariadb
```

### Upgrade/rollback release

```bash
$ h upgrade -f config.yaml happy-panda apphub/mariadb
$ h rollback happy-panda 1
```

### Release history

```bash
$ h history happy-panda
```

## Advance

### Create chart

```bash
$ h create deis-workflow
```

### Package a chart

```bash
$ h package deis-workflow
```

### Install zip package

```bash
$ h install deis-workflow ./deis-workflow-0.1.0.tgz
```

### Get release manifest

```bash
$ h get manifest happy-panda
```

### Install dry-run

```bash
$ h install clunky2 --debug --dry-run ./mychart/
```

### Lint a chart

```bash
$ h lint dnc-cloud-db/
```

### Pull a chart(helm v3 only)

```bash
$ h pull stable/coredns --version v1.7.2
```

### Push a chart

```bash
$ h push ./coredns/ chartmuseum-dev
```

### List & update dependency

```bash
$ h dep list
$ h dep update
```

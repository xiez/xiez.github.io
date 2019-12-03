---
layout: post
title: Helm command cheatsheet
categories: helm cheatsheet
---

This post is a cheatsheet of the `helm` commands. [Helm](https://helm.sh/) is a package manager for Kubernetes, like apt/yum/homebrew for Kubernetes.

## Basic

### Add repo

```

# use custom repo in China, see https://github.com/cloudnativeapp/charts
helm repo add apphub https://apphub.aliyuncs.com

```

### Search a package

```
helm search repo mariadb

```

### Install a package

```
helm install happy-panda apphub/mariadb
```

**output:**

```
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

```
helm status happy-panda

```

### Show values

```
helm show values apphub/mariadb
```

### Override values

```
echo '{mariadbUser: user0, mariadbDatabase: user0db}' > config.yaml
helm install happy-panda -f config.yaml apphub/mariadb
```

### Upgrade/rollback release

```
helm upgrade -f config.yaml happy-panda apphub/mariadb

helm rollback happy-panda 1
```

### Release history

```
helm history happy-panda
```

## Advance

### Create chart

```
helm create deis-workflow
```

### Package a chart

```
helm package deis-workflow
```

### Install zip package

```
helm install deis-workflow ./deis-workflow-0.1.0.tgz
```

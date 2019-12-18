---
title: K8S command cheatsheet
classes: wide
categories:
  - 2019-11
tags:
  - kubernetes
  - cheatsheet
---

This post is a cheatsheet of the `kubectl` and `minikube` commands.

## .bash_aliases

```bash
$ alias k='kubectl'
$ alias mk='minikube'
```


## kubectl

### Get/Describe Resource(s)

```bash
$ k get all --all-namespaces

# output format
$ k get pods -o wide

# get in default namespace
$ k get pods
$ k get services
$ k get deployments

# get in 'kube-system' namespace
$ k get pod coredns-9b8997588-87vtz --namespace kube-system

# describe a resource
$ k describe services/nginx-deployment
```

### Expose Resource(s)

```bash
# create service
$ k expose deployment/nginx-deployment  --type="NodePort" --port 80
```

### Curl via Node Port

```bash
$ export NODE_PORT=$(kubectl get services/nginx-deployment -o go-template='{{(index .spec.ports 0).nodePort}}')
$ echo NODE_PORT=$NODE_PORT
$ curl -v $(minikube ip):$NODE_PORT
```

### Port forward

```bash
# grafana
$ k --namespace default port-forward grafana-1575089321-6f4b7f7875-lds7b 3000

# nginx
$ export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=deis-workflow,app.kubernetes.io/instance=deis-workflow" -o jsonpath="{.items[0].metadata.name}")
$ echo "Visit http://127.0.0.1:8080 to use your application"
$ k --namespace default port-forward $POD_NAME 8080:80
```

### Scale Resource(s)

```bash
$ k scale deployments/nginx-deployment --replicas=4
```

### Set Resource(s)

```bash
$ k set image deployments/nginx-deployment nginx=nginx:latest
```

### Rollout Resource(s)

```bash
$ k rollout status deployments/nginx-deployment

# undo in case failure
$ k rollout undo deployments/nginx-deployment
```

### Exec commands

```bash
$ k exec -it kubernetes-dashboard-57f4cb4545-7ltsv --namespace=kubernetes-dashboard  sh
```

## minikube

```bash
# use custom image repo in case k8s.gcr.io is blocked, see https://github.com/kubernetes/minikube/pull/3714#issuecomment-514186245
$ mk start --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers

$ mk ip

$ mk ssh

$ mk docker-env

$ mk dashboard

# scp file to minikube virtual box
$ scp -i ~/.minikube/machines/minikube/id_rsa  /tmp/dj.tar docker@$(minikube ip):~

$ mk service django-service

# set local docker CLI to point to the minikube docker daemon
$ eval $(minikube docker-env)

# set local docker CLI point back to local docker daemon
$ eval $(minikube docker-env -u)
```

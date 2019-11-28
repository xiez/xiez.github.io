---
layout: post
title: k8s-command-cheatsheet
categories: kubernetes cheatsheet
---

## kubectl

### Get/Describe Resource(s)

```
kubectl get all --all-namespaces

# output format
kubectl get pods -o wide

# get in default namespace
kubectl get pods
kubectl get services
kubectl get deployments

# get in 'kube-system' namespace
kubectl get pod coredns-9b8997588-87vtz --namespace kube-system

# describe a resource
kubectl describe services/nginx-deployment

```

### Expose Resource(s)

```
# create service
kubectl expose deployment/nginx-deployment  --type="NodePort" --port 80

```

### Curl via Node Port
```
export NODE_PORT=$(kubectl get services/nginx-deployment -o go-template='{{(index .spec.ports 0).nodePort}}')
echo NODE_PORT=$NODE_PORT
curl -v $(minikube ip):$NODE_PORT
```

### Scale Resource(s)

```
kubectl scale deployments/nginx-deployment --replicas=4
```

### Set Resource(s)

```
kubectl set image deployments/nginx-deployment nginx=nginx:latest
```

### Rollout Resource(s)

```
kubectl rollout status deployments/nginx-deployment

# undo in case failure
kubectl rollout undo deployments/nginx-deployment

```

### Exec commands

```
kubectl exec -it kubernetes-dashboard-57f4cb4545-7ltsv --namespace=kubernetes-dashboard  sh
```

## minikube

```
# use custom image repo in case k8s.gcr.io is blocked, see https://github.com/kubernetes/minikube/pull/3714#issuecomment-514186245
minikube start --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers

minikube ip

minikube ssh

minikube docker-env

minikube dashboard

# scp file to minikube virtual box
scp -i ~/.minikube/machines/minikube/id_rsa  /tmp/dj.tar docker@$(minikube ip):~

```

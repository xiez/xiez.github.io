---
title:  Prometheus 监控系统搭建
classes: wide
categories:
  - 2020-09
tags:
  - kubernetes
  - prometheus
---

**本文目标:** 在内网 K8S 集群内部署 Prometheus ，展示机器CPU内存等各项指标以及应用请求延迟。

## Why Prometheus ?

- 基于 pull 的指标数据获取方式，相比于 push 方式，可靠性和扩展性更好。

- 作为数据源与 Grafana 完美集成。

- [Cloud-Native](https://landscape.cncf.io/selected=prometheus)

## Why in K8S ?

- 沿用已有的基础设施，最大化资源利用率。

- 高可用，可扩展。

- 一键部署 Prometheus & Grafana

## 架构图

![prometheus arch](/assets/images/2020/09/prometheus.jpg)

## 部署步骤

### 1. 安装 [Promethus Operator](https://github.com/prometheus-operator/prometheus-operator)

```
helm install prometheus-operator apphub/prometheus-operator --namespace monitor
```

### 2. 验证安装完成

```
ubuntu@master:~/node-exporter/oms-st$ kubectl  --namespace=monitor  get po
NAME                                                      READY   STATUS    RESTARTS   AGE
alertmanager-prometheus-operator-alertmanager-0           2/2     Running   0          25h
prometheus-operator-grafana-cffcfd7b7-lx8dd               2/2     Running   0          25h
prometheus-operator-kube-state-metrics-5855d94d57-ppjxj   1/1     Running   0          25h
prometheus-operator-operator-589b8cf785-ck8nm             2/2     Running   0          25h
prometheus-operator-prometheus-node-exporter-69vbq        1/1     Running   0          25h
prometheus-operator-prometheus-node-exporter-gtc6f        1/1     Running   0          25h
prometheus-operator-prometheus-node-exporter-qnv8k        1/1     Running   0          25h
prometheus-prometheus-operator-prometheus-0               3/3     Running   1          25h
```

### 3. 访问 Prometheus & Grafana

```
ubuntu@master:~$ kubectl port-forward -n monitor prometheus-prometheus-operator-prometheus-0 9090 --address=0.0.0.0
ubuntu@master:~$ kubectl port-forward $(kubectl get pods --selector=app=grafana -n monitor --output=jsonpath="{.items..metadata.name}") -n monitor 3000 --address=0.0.0.0
```

### 4. EC2 机器上安装 [node-exporter](https://github.com/prometheus/node_exporter)

```
docker run -d --pid="host" -p 0.0.0.0:9100:9100 -v "/:/host:ro,rslave" registry.cyai.com/monitor/node-exporter:v0.18.0 --path.rootfs=/host
```

### 5. Prometheus Operator [配置外部服务](https://jpweber.io/blog/monitor-external-services-with-the-prometheus-operator/)

分别创建3类资源（endpoint, service, service monitor）

```
apiVersion: v1
kind: Endpoints
metadata:
  name: oms-node-metrics
  labels:
    k8s-app: oms-node-metrics
subsets:
  - addresses:
    - ip: 0.0.114.1
    ports:
      - name: metrics
        port: 9100
        protocol: TCP

---

apiVersion: v1
kind: Service
metadata:
  name: oms-node-metrics
  namespace: monitor
  labels:
    k8s-app: oms-node-metrics
spec:
  type: ExternalName
  externalName: 0.0.114.1
  clusterIP: ""
  ports:
    - name: metrics
      port: 9100
      protocol: TCP
      targetPort: 9100

---

apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: oms-node-metrics-sm
  labels:
    k8s-app: oms-node-metrics
    release: prometheus-operator
spec:
  selector:
    matchLabels:
      k8s-app: oms-node-metrics
  endpoints:
    - port: metrics
      interval: 10s
      honorLabels: true
```

验证资源创建成功：

```
ubuntu@master:~/node-exporter/oms$ kubectl  --namespace=monitor  describe  svc/oms-node-metrics
Name:              oms-node-metrics
Namespace:         monitor
Labels:            k8s-app=oms-node-metrics
Annotations:       Selector:  <none>
Type:              ExternalName
IP:                
External Name:     0.0.114.1
Port:              metrics  9100/TCP
TargetPort:        9100/TCP
Endpoints:         0.0.114.1:9100
Session Affinity:  None
Events:            <none>
```

查看 targets

![targets](/assets/images/2020/09/targets.png)


最后，在 Grafana 界面上配置数据源。

![grafana](/assets/images/2020/09/grafana.png)

## 参考资料

- [https://prometheus.io/blog/2016/07/23/pull-does-not-scale-or-does-it/](https://prometheus.io/blog/2016/07/23/pull-does-not-scale-or-does-it/)

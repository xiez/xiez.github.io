---
title:  Ansible 笔记
classes: wide
categories:
  - 2021-02
tags:
  - ansible
---

Ansible 作为常见的配置管理（[Configuration Management](https://en.wikipedia.org/wiki/Configuration_management)）工具，相比 chef/puppet 的优势：

- Ansible offers a simple architecture that doesn’t require special software to be installed on nodes.

- It also provides a robust set of features and built-in modules which facilitate writing automation scripts.

- 使用 python 和声明式的 yaml 语法，简单易懂。

## 安装

参考这篇 [DO 文章](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-ansible-on-ubuntu-18-04#step-2-%E2%80%94-setting-up-the-inventory-file)

安装完成后, 配置下 `hosts`:

```
xiez@xiez-Vostro-3470:~$ cat /etc/ansible/hosts
# This is the default ansible 'hosts' file.
#
# It should live in /etc/ansible/hosts
#
#   - Comments begin with the '#' character
#   - Blank lines are ignored
#   - Groups of hosts are delimited by [header] elements
#   - You can enter hostnames or ip addresses
#   - A hostname/ip can be a member of multiple groups

[servers]
server1 ansible_host=10.6.2.21
server2 ansible_host=10.6.2.22
server3 ansible_host=10.6.2.23

[servers:vars]
ansible_python_interpreter=/usr/bin/python3
```

测试能否ping通:

```
xiez@xiez-Vostro-3470:~$ ansible all -m ping -u ubuntu
server3 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
server1 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
server2 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
```

## 常用命令 & Cheetsheet

```
ansible servers -a "uptime" -u ubuntu

ansible servers --become -m apt -a "name=vim state=latest" -u ubuntu
```

Cheetsheet: [https://www.edureka.co/blog/cheatsheets/ansible-cheat-sheet-guide/](https://www.edureka.co/blog/cheatsheets/ansible-cheat-sheet-guide/)

## Case study 1

**通过 playbook 在安装 docker**

参考这篇 [DO 教程](https://www.digitalocean.com/community/tutorials/how-to-use-ansible-to-install-and-set-up-docker-on-ubuntu-18-04)

```
git clone https://github.com/do-community/ansible-playbooks.git

ansible-playbook playbook.yml -l server1 -u ubuntu

```


## Case study 2

**通过 playbook 在3台全新的 ubuntu18.04 系统上安装 K8S 集群**


### 1. Edit ansible hosts

使用本地的 hosts 文件: [https://github.com/xiez/ansible-playbooks/blob/master/hosts](https://github.com/xiez/ansible-playbooks/blob/master/hosts)

```
xiez@xiez-Vostro-3470:~/dev/ansible-playbooks$ cat hosts
[servers]
server1 ansible_host=10.6.2.21
server2 ansible_host=10.6.2.22
server3 ansible_host=10.6.2.23

[servers:vars]
ansible_python_interpreter=/usr/bin/python3
```

### 2. Setup Ubuntu

#### 2.1 新增 ubuntu 用户, 开启公钥登录, 禁用密码登录

[playbook](https://github.com/xiez/ansible-playbooks/blob/master/01-setup_ubuntu1804/playbook.yml)

```
ansible-playbook -i hosts 01-setup_ubuntu1804/playbook.yml --extra-vars "ansible_user=cyuser ansible_password=123456 ansible_sudo_pass=123456"
```

#### 2.2 修改 apt 源为 aliyun

[playbook](https://github.com/xiez/ansible-playbooks/blob/master/02-ubuntu_apt_source/playbook.yml)

```
ansible-playbook -i hosts 02-ubuntu_apt_source/playbook.yml
```

#### 2.3 安装 docker, 修改 pip 源为 aliyun

[playbook](https://github.com/xiez/ansible-playbooks/blob/master/03-docker_ubuntu18.04/playbook.yml)

```
ansible-playbook -i hosts 03-docker_ubuntu18.04/playbook.yml
```

### 3. Install Kubernetes v1.18.0

#### 3.1 安装依赖（kubelet, kubeadmin, kubectl）

```
ansible-playbook -i hosts  04-kube_deps/playbook.yml
```

#### 3.2 节点额外配置（禁用swap，docker cgroupdriver改成systemd, 更改系统hostname）

```
ansible-playbook -i hosts  05-kube_pre_requirements/playbook.yml
```

#### 3.3 配置 master node（pull kubernetes v1.18 images, kubeadmin init, 设置 flannel 网络接口插件）

```
ansible-playbook -i hosts  06-setup_kube_master/playbook.yml
```

#### 3.4 配置 worker node（pull kubernetes v1.18 images, join cluser）

```
ansible-playbook -i hosts  07-setup_kube_worker/playbook.yml
```

#### 3.5 测试集群节点状态

在 master 节点上执行 `get node`

```
ubuntu@master:~$ kubectl get node
NAME      STATUS   ROLES    AGE   VERSION
master    Ready    master   17h   v1.18.0
worker1   Ready    <none>   17h   v1.18.0
worker2   Ready    <none>   17h   v1.18.0
```

### Test nginx deploy

```
kubectl create deployment nginx --image=nginx

kubectl expose deploy nginx --port 80 --target-port 80 --type NodePort
```

```
kubectl get services

NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
kubernetes ClusterIP 10.96.0.1 <none> 443/TCP 18h
nginx NodePort 10.102.213.64 <none> 80:30854/TCP 7m58s
```

Test with curl:

```
xiez@xiez-Vostro-3470:~/dev/ansible-playbooks$ curl http://10.6.2.21:30854/
xiez@xiez-Vostro-3470:~/dev/ansible-playbooks$ curl http://10.6.2.22:30854/
xiez@xiez-Vostro-3470:~/dev/ansible-playbooks$ curl http://10.6.2.23:30854/
```

## 参考资料

- [https://www.digitalocean.com/community/tutorials/how-to-create-a-kubernetes-cluster-using-kubeadm-on-ubuntu-18-04](https://www.digitalocean.com/community/tutorials/how-to-create-a-kubernetes-cluster-using-kubeadm-on-ubuntu-18-04)

- [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

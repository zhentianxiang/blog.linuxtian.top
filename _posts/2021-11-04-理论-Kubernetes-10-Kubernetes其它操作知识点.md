---
layout: post
title: 理论-Kubernetes-10-Kubernetes其它操作知识点
date: 2021-11-05
tags: 理论-Kubernetes
---

## 一、集群内操作

### 1. 调整k8s默认端口范围

```sh
[root@k8s-master ~]# vim /etc/kubernetes/manifests/kube-apiserver.yaml
...................
- --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
- --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
- --requestheader-allowed-names=front-proxy-client
- --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
- --requestheader-extra-headers-prefix=X-Remote-Extra-
- --requestheader-group-headers=X-Remote-Group
- --requestheader-username-headers=X-Remote-User
- --secure-port=6443
- --service-account-key-file=/etc/kubernetes/pki/sa.pub
- --service-cluster-ip-range=10.96.0.0/12
- --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
- --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
- --service-node-port-range=1-65535  ###添加此行内容
...........................
```

### 2. 开启默认StorageClass

以下仅仅是配置了一个默认的Storage Class，要想配合pvc的使用，还需要配置后端存储。

如：利用nfs来作为后端存储

可以参考这一章内容 https://blog.linuxtian.top/2021/08/Linux-Kubernetes-34-交付EFK到K8S

```sh
[root@k8s-master ~]# vim /etc/kubernetes/manifests/kube-apiserver.yaml
......................
- --service-cluster-ip-range=10.96.0.0/12
- --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
- --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
- --service-node-port-range=1-65535
- --enable-admission-plugins=NodeRestriction,DefaultStorageClass  ###添加此行内容
```
编辑一个storage class
```sh
[root@k8s-master ~]# vim /etc/kubernetes/manifests/defaultclass01.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: do-block-storage
  annotations:
    storageclass.beta.kubernetes.io/is-default-class: "true"
provisioner: example.com/nfs
parameters:
  type: pd-ssd
[root@k8s-master ~]# kubectl apply -f /etc/kubernetes/manifests/defaultclass01.yaml
通过kubectl create命令创建成功后，查看StorageClass列表，可以看到名为gold的StorageClass被标记为default
[root@k8s-master ~]# kubectl get sc
NAME                         PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
do-block-storage (default)   example.com/nfs   Delete          Immediate           false                  29m
```

## 二、集群外操作

### 1. 修改kubelet工作目录

用为像docker和kubelet默认的工作目录目录在/var/lib/下面，后期工作中可能会遇到磁盘空间不足导致集群出问题，因此我们在初始化集群初期可以修改kubelet的工作目录

根据 /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf 加载文件，只需要修改 /etc/sysconfig/kubelet 即可。
```sh
[root@k8s-master ~]# mkdir -p /data/k8s/lib/kubelet
[root@k8s-master ~]# cp -a /var/lib/kubelet/* /data/k8s/lib/kubelet
[root@k8s-master ~]# systemctl stop kubelet
[root@k8s-master ~]# vim /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--root-dir=/data/k8s/lib/kubelet"
[root@k8s-master ~]# vim /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf

# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/data/k8s/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/data/k8s/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/sysconfig/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
```

如果集群是搭建好之后修改kubelet工作目录的话，需要将源文件拷贝至新目录

```sh
[root@k8s-master ~]# cp -rp /data/k8s/lib/kubelet/pki/ /var/lib/kubelet/     ##后面可能会报证书路径问题，故将证书拷贝回原路径如果不报错，则不用此命令
[root@k8s-master ~]# systemctl daemon-reload
[root@k8s-master ~]# systemctl restart kubelet
```

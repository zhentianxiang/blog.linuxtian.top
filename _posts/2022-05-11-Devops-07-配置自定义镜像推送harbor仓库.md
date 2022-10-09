---
layout: post
title: Devops-07-配置自定义镜像推送harbor仓库
date: 2022-05-11
tags: Devops
---

### 1. 安装harbor

这里我使用的机器还是jenkins构建项目用来发布部署的机器（资源紧张）

```sh
[root@localhost ~]# ls
anaconda-ks.cfg   harbor-offline-installer-v2.1.5.tgz
[root@localhost ~]# tar xvf harbor-offline-installer-v2.1.5.tgz -C /usr/local/
[root@localhost harbor]# mkdir cert
[root@localhost harbor]# ls
cert  common  common.sh  data  docker-compose.yml  harbor.v2.1.5.tar.gz  harbor.yml  harbor.yml.tmpl  install.sh  LICENSE  prepare
[root@localhost cert]# openssl req  -newkey rsa:4096 -nodes -sha256 -keyout ca.key -x509 -days 365 -out ca.crt -subj "/C=CN/L=Beijing/O=lisea/CN=harbor-registry"
[root@localhost cert]# openssl req -newkey rsa:4096 -nodes -sha256 -keyout devops.key -out server.csr -subj "/C=CN/L=Beijing/O=lisea/CN=devops.com"
[root@localhost cert]# openssl x509 -req -days 365 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out devops.crt
[root@localhost cert]# ls
ca.crt  ca.key  ca.srl  server.csr  devops.crt  devops.key
[root@localhost cert]# cd ..
[root@localhost harbor]# cp harbor.yml.tmpl harbor.yml
[root@localhost harbor]# vim harbor.yml

# Configuration file of Harbor

# The IP address or hostname to access admin UI and registry service.
# DO NOT use localhost or 127.0.0.1, because Harbor needs to be accessed by external clients.
hostname: 192.168.20.54       # 修改此位置,IP 和 域名随意写，自己开心就行

# http related config
#http:  # 注释此位置
  # port for http, default is 80. If https enabled, this port will redirect to https port
  #port: 80  # 注释此位置

# https related config
https:
  # https port for harbor, default is 443
  port: 443
  # The path of cert and key files for nginx
  certificate: /usr/local/harbor/cert/devops.crt  # 修改此位置
  private_key: /usr/local/harbor/cert/devops.key  # 修改此位置
[root@localhost harbor]# ./prepare && ./install.sh && docker-compose ps
      Name                     Command                  State                          Ports                   
---------------------------------------------------------------------------------------------------------------
clair               ./docker-entrypoint.sh           Up (healthy)                                              
clair-adapter       /home/clair-adapter/entryp ...   Up (healthy)                                              
harbor-core         /harbor/entrypoint.sh            Up (healthy)                                              
harbor-db           /docker-entrypoint.sh            Up (healthy)                                              
harbor-jobservice   /harbor/entrypoint.sh            Up (healthy)                                              
harbor-log          /bin/sh -c /usr/local/bin/ ...   Up (healthy)   127.0.0.1:1514->10514/tcp                  
harbor-portal       nginx -g daemon off;             Up (healthy)                                              
nginx               nginx -g daemon off;             Up (healthy)   0.0.0.0:80->8080/tcp, 0.0.0.0:443->8443/tcp
redis               redis-server /etc/redis.conf     Up (healthy)                                              
registry            /home/harbor/entrypoint.sh       Up (healthy)                                              
registryctl         /home/harbor/start.sh            Up (healthy)
```

### 2. docker 配置私有仓库

```sh
[root@localhost harbor]# vim /etc/docker/daemon.json
{
"registry-mirrors": ["https://9w1hl6qt.mirror.aliyuncs.com"],
"insecure-registries": ["registry.access.redhat.com","quay.io","192.168.20.54"]
[root@localhost harbor]# systemctl restart docker
# 重启docker后建议重启一下harbor
[root@localhost harbor]# docker-compose down && docker-compose up -d
```

![](/images/posts/Devops/配置自定义镜像推送harbor仓库/1.png)

### 3. jenkins 配置

![2](/images/posts/Devops/配置自定义镜像推送harbor仓库/2.png)

![3](/images/posts/Devops/配置自定义镜像推送harbor仓库/3.png)

![7](/images/posts/Devops/配置自定义镜像推送harbor仓库/7.png)

![8](/images/posts/Devops/配置自定义镜像推送harbor仓库/8.png)

![9](/images/posts/Devops/配置自定义镜像推送harbor仓库/9.png)

![10](/images/posts/Devops/配置自定义镜像推送harbor仓库/10.png)

![11](/images/posts/Devops/配置自定义镜像推送harbor仓库/11.png)

![12](/images/posts/Devops/配置自定义镜像推送harbor仓库/12.png)

![13](/images/posts/Devops/配置自定义镜像推送harbor仓库/13.png)

![14](/images/posts/Devops/配置自定义镜像推送harbor仓库/14.png)

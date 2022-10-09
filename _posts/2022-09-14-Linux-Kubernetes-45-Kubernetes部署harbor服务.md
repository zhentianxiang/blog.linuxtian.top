---
layout: post
title: Linux-Kubernetes-45-Kubernetes部署harbor服务
date: 2022-09-14
tags: 实战-Kubernetes
---

# Kubernetes部署harbor服务

## 一、实验环境

```sh
[root@kubesphere ~]# kubectl get node -o wide
NAME         STATUS   ROLES                  AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
k8s-node01   Ready    <none>                 14h   v1.20.1   192.168.20.121   <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.15
kubesphere   Ready    control-plane,master   89d   v1.20.1   192.168.20.120   <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   docker://19.3.15
[root@kubesphere ~]# docker info |grep Version
 Server Version: 19.03.15
 Kernel Version: 3.10.0-1160.el7.x86_64
[root@kubesphere ~]# helm version
version.BuildInfo{Version:"v3.9.3", GitCommit:"414ff28d4029ae8c8b05d62aa06c7fe3dee2bc58", GitTreeState:"clean", GoVersion:"go1.17.13"}
```

> helm-harbor.tar.gz: 链接：https://pan.baidu.com/s/1MSmDJ7L01bI0Sa--SlA_ng?pwd=c1j5
提取码：c1j5
>
> helm-v3.9.3-linux-amd64.tar.gz: 链接：https://pan.baidu.com/s/1BEG3VPVDrPgQmVxdCPcTgQ?pwd=jw1b
提取码：jw1b



### 1. 替换所有镜像

以下只是harbor的镜像为掩饰，pgo和redis的一样操作，所以不再重复演示

```sh
[root@kubesphere helm-harbor]# tar xvf helm-harbor.tar.gz
[root@kubesphere helm-harbor]# ls
harbor  images  pgo  redis-ha
[root@kubesphere helm-harbor]# cd harbor/harbor-1.8.3/
[root@kubesphere harbor-1.8.3]# cat values.yaml |grep '^ *repository*'
    repository: goharbor/nginx-photon
    repository: goharbor/harbor-portal
    repository: goharbor/harbor-core
    repository: goharbor/harbor-jobservice
      repository: goharbor/registry-photon
      repository: goharbor/harbor-registryctl
    repository: goharbor/chartmuseum-photon
    repository: goharbor/trivy-adapter-photon
      repository: goharbor/notary-server-photon
      repository: goharbor/notary-signer-photon
      repository: goharbor/harbor-db
      repository: goharbor/redis-photon
      repository: goharbor/harbor-exporter
```

替换

```sh
[root@kubesphere harbor-1.8.3]# sed -i "s/repository\: goharbor/repository\: 192.168.20.120\:1443\/goharbor/g" values.yaml
[root@kubesphere harbor-1.8.3]# cat values.yaml |grep '^ *repository*'
    repository: 192.168.20.120:1443/goharbor/nginx-photon
    repository: 192.168.20.120:1443/goharbor/harbor-portal
    repository: 192.168.20.120:1443/goharbor/harbor-core
    repository: 192.168.20.120:1443/goharbor/harbor-jobservice
      repository: 192.168.20.120:1443/goharbor/registry-photon
      repository: 192.168.20.120:1443/goharbor/harbor-registryctl
    repository: 192.168.20.120:1443/goharbor/chartmuseum-photon
    repository: 192.168.20.120:1443/goharbor/trivy-adapter-photon
      repository: 192.168.20.120:1443/goharbor/notary-server-photon
      repository: 192.168.20.120:1443/goharbor/notary-signer-photon
      repository: 192.168.20.120:1443/goharbor/harbor-db
      repository: 192.168.20.120:1443/goharbor/redis-photon
      repository: 192.168.20.120:1443/goharbor/harbor-exporter
```

推送镜像

```sh
[root@kubesphere helm-harbor]# ls
harbor  images  k8s_harbor_v2.4.3.tar.xz  pgo  redis-ha
[root@kubesphere helm-harbor]# cd images/
[root@kubesphere images]# ls
harbor  pgo  redis
[root@kubesphere images]# cd harbor/
[root@kubesphere harbor]# ls
export.sh  hack  harbor.tar  import.sh
[root@kubesphere harbor]# docker load -i harbor.tar
[root@kubesphere harbor]# docker images |grep v2.4.3
goharbor/harbor-portal                                                             v2.4.3                         5e3dfb1a5059        5 weeks ago         151MB
goharbor/nginx-photon                                                              v2.4.3                         7d1eeb7adf34        6 weeks ago         142MB
goharbor/chartmuseum-photon                                                        v2.4.3                         f39a9694988d        6 weeks ago         172MB
goharbor/trivy-adapter-photon                                                      v2.4.3                         a406a715461c        6 weeks ago         251MB
goharbor/notary-server-photon                                                      v2.4.3                         da89404c7cf9        6 weeks ago         109MB
goharbor/notary-signer-photon                                                      v2.4.3                         38468ac13836        6 weeks ago         107MB
goharbor/harbor-registryctl                                                        v2.4.3                         61243a84642b        6 weeks ago         135MB
goharbor/registry-photon                                                           v2.4.3                         9855479dd6fa        6 weeks ago         77.9MB
goharbor/harbor-jobservice                                                         v2.4.3                         7fea87c4b884        6 weeks ago         219MB
goharbor/harbor-core                                                               v2.4.3                         d864774a3b8f        6 weeks ago         197MB
goharbor/harbor-db                                                                 v2.4.3                         7693d44a2ad6        6 weeks ago         225MB

# 修改tag
[root@kubesphere harbor]# docker images |grep v2.4.3 | sed 's/goharbor/192.168.20.120\:1443\/goharbor/g' | awk '{print "docker tag"" " $3" "$1":"$2}'
docker tag 5e3dfb1a5059 192.168.20.120:1443/goharbor/harbor-portal:v2.4.3
docker tag 7d1eeb7adf34 192.168.20.120:1443/goharbor/nginx-photon:v2.4.3
docker tag f39a9694988d 192.168.20.120:1443/goharbor/chartmuseum-photon:v2.4.3
docker tag a406a715461c 192.168.20.120:1443/goharbor/trivy-adapter-photon:v2.4.3
docker tag da89404c7cf9 192.168.20.120:1443/goharbor/notary-server-photon:v2.4.3
docker tag 38468ac13836 192.168.20.120:1443/goharbor/notary-signer-photon:v2.4.3
docker tag 61243a84642b 192.168.20.120:1443/goharbor/harbor-registryctl:v2.4.3
docker tag 9855479dd6fa 192.168.20.120:1443/goharbor/registry-photon:v2.4.3
docker tag 7fea87c4b884 192.168.20.120:1443/goharbor/harbor-jobservice:v2.4.3
docker tag d864774a3b8f 192.168.20.120:1443/goharbor/harbor-core:v2.4.3
docker tag 7693d44a2ad6 192.168.20.120:1443/goharbor/harbor-db:v2.4.3
[root@kubesphere harbor]# docker images |grep v2.4.3 | sed 's/goharbor/192.168.20.120\:1443\/gohorbor/g' | awk '{print "docker tag"" " $3" "$1":"$2}'|bash

# 创建镜像库
[root@kubesphere pgo]# curl -k -u 'admin:Harbor12345' -XPOST -H "Content-Type:application/json" -d '{"project_name": "goharbor", "metadata": {"public": "true"}, "storage_limit": -1}' "https://192.168.20.120:1443/api/v2.0/projects"

# 推送
[root@kubesphere harbor]# docker images | grep "192.168.20.120\:1443\/goharbor" | awk '{print "docker push "$1":"$2}'
docker push 192.168.20.120:1443/goharbor/harbor-portal:v2.4.3
docker push 192.168.20.120:1443/goharbor/nginx-photon:v2.4.3
docker push 192.168.20.120:1443/goharbor/chartmuseum-photon:v2.4.3
docker push 192.168.20.120:1443/goharbor/trivy-adapter-photon:v2.4.3
docker push 192.168.20.120:1443/goharbor/notary-server-photon:v2.4.3
docker push 192.168.20.120:1443/goharbor/notary-signer-photon:v2.4.3
docker push 192.168.20.120:1443/goharbor/harbor-registryctl:v2.4.3
docker push 192.168.20.120:1443/goharbor/registry-photon:v2.4.3
docker push 192.168.20.120:1443/goharbor/harbor-jobservice:v2.4.3
docker push 192.168.20.120:1443/goharbor/harbor-core:v2.4.3
docker push 192.168.20.120:1443/goharbor/harbor-db:v2.4.3
[root@kubesphere harbor]# docker images | grep "192.168.20.120\:1443\/gohorbor" | awk '{print "docker push "$1":"$2}'|bash
```

## 二、准备部署服务

> 为了方便编写，我这里直接写好了所有的value.yaml文件，大家直接下载即可
>
> https://blog.linuxtian.top/data/helm-harbor/pgo/values.yaml
>
> https://blog.linuxtian.top/data/helm-harbor/redis/values.yaml
>
> https://blog.linuxtian.top/data/helm-harbor/harbor/values.yaml

### 1. 部署pgo和postgres

> 我们使⽤ pgo 部署 postgres ，采⽤helm安装，服务供harbor调⽤
>
> 通过创建 CR 的⽅式部署 postgres数据库，[详细 CR 参考](https://access.crunchydata.com/documentation/postgres-operator/v5/references/crd/)

```sh
[root@kubesphere helm-harbor]# vim pgo/postgres_cluster/ha-postgres.yaml

  instances:
    - name: harbor-ha-instance
      replicas: 2
      dataVolumeClaimSpec:
       # storageClassName: nfs-sc  # 使用的存储类，默认不填时为默认的存储类
        accessModes:
          - "ReadWriteOnce"
        resources:
          requests:
            storage: 15Gi
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                topologyKey: kubernetes.io/hostname
                labelSelector:
                  matchLabels:
                    postgres-operator.crunchydata.com/instance-set: harbor_ha_instance
  backups:
    pgbackrest:
      image: 192.168.20.120:1443/crunchydata/crunchy-pgbackrest:ubi8-2.38-2
      repos:
        - name: repo1
          volume:
            volumeClaimSpec:
             # storageClassName: nfs-sc  # 使用的存储类，默认不填时为默认的存储类
              accessModes:
                - "ReadWriteOnce"
              resources:
                requests:
                  storage: 15Gi
  proxy:
    pgBouncer:
      image: 192.168.20.120:1443/crunchydata/crunchy-pgbouncer:ubi8-1.16-4
      replicas: 2
      service:
        type: NodePort
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 1
              podAffinityTerm:
                topologyKey: kubernetes.io/hostname
                labelSelector:
                  matchLabels:
                    postgres-operator.crunchydata.com/cluster: harbor
                    postgres-operator.crunchydata.com/role: pgbouncer
  users:
    - name: harbor
      databases:
        - harbor
  service:
    type: NodePort
```

准备启动

```sh
[root@kubesphere helm-harbor]# cd pgo/pgo_install/
[root@kubesphere pgo_install]# helm install -n pgo pgo .
NAME: pgo
LAST DEPLOYED: Wed Sep 14 16:42:09 2022
NAMESPACE: pgo
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for deploying PGO v5.1.2!

                          ((((((((((((((((((((((
                    (((((((((((((%%%%%%%(((((((((((((((
                (((((((((((%%%             %%%%((((((((((((
            (((((((((((%%(   (((( (            %%%(((((((((((
          (((((((((((((%%     (( ,((               %%%(((((((((((
        (((((((((((((((%%         *%%/            %%%%%%%((((((((((
      (((((((((((((((((((%%(( %%%%%%%%%%#(((((%%%%%%%%%%#((((((((((((
    ((((((((((((((((((%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%((((((((((((((
  *((((((((((((((((((((%%%%%%     /%%%%%%%%%%%%%%%%%%%((((((((((((((((
  (((((((((((((((((((((((%%%/      .%,             %%%((((((((((((((((((,
  ((((((((((((((((((((((%                             %#(((((((((((((((((
(((((((((((((((%%%%%%                                 #%(((((((((((((((((
((((((((((((((%%                                         %%(((((((((((((((,
((((((((((((%%%#%                                     %   %%(((((((((((((((
((((((((((((%.                      %                 %     #((((((((((((((
(((((((((((%%                        %               %%*     %(((((((((((((
#(###(###(#%%                      %%%    %%        %%%      #%%#(###(###(#
###########%%%%%   /%%%%%%%%%%%%%       %%       %%%%%         ,%%#######
###############%%       %%%%%%        %%%    %%%%%%%%             %%#####
  ################%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%   %%               %%##
  ################%%        %%%%%%%%%%%%%%%%%      %%%%               %
    ##############%#        %%   (%%%%%%%           %%%%%%
    #############%     %%%%%                      %%%%%%%%%%%
      ###########%       %%%%%%%%%%%            %%%%%%%%%
        #########%%     %%            %%%%%%%%%%%%%%%#
          ########%%   %%                  %%%%%%%%%
              ######%% %%                      %%%%%%
                ####%%%                        %%%%%  %
                      %%                         %%%%
```

查看启动情况

```sh
[root@kubesphere pgo_install]# kubectl get pods -n pgo -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP               NODE         NOMINATED NODE   READINESS GATES
pgo-6f45cd8db9-ch6gw           1/1     Running   0          33s   10.100.148.131   kubesphere   <none>           <none>
pgo-6f45cd8db9-llmkx           1/1     Running   0          33s   10.100.85.251    k8s-node01   <none>           <none>
pgo-upgrade-6dc84658b4-9cp84   1/1     Running   0          33s   10.100.85.252    k8s-node01   <none>           <none>
pgo-upgrade-6dc84658b4-9gx4t   1/1     Running   0          33s   10.100.148.163   kubesphere   <none>           <none>
```

创建 CR namespace 以及启动CR

```sh
[root@kubesphere pgo_install]# kubectl create ns postgres
namespace/postgres created
[root@kubesphere pgo_install]# kubectl -n postgres apply -f ../postgres_cluster/ha-postgres.yaml
postgrescluster.postgres-operator.crunchydata.com/harbor created
```

查看启动情况

```sh
[root@kubesphere pgo_install]# kubectl get pods -n postgres
harbor-backup-zzm6-qt8ts           0/1     Completed   0          38s
harbor-harbor-ha-instance-qg5d-0   4/4     Running     0          6m16s
harbor-harbor-ha-instance-rptd-0   4/4     Running     0          6m12s
harbor-pgbouncer-78cbd46bd-hc9nh   2/2     Running     0          6m16s
harbor-pgbouncer-78cbd46bd-ls7v8   2/2     Running     0          6m16s
harbor-repo-host-0                 2/2     Running     0          6m16s
```

### 2. 部署redis服务

> Redis 采⽤哨兵模式启动，使⽤helm 安装，服务供harbor调⽤
>
> Redis 的主要配置是 values.yaml ，
>
> 检查事项： 检查副本数量，默认是3 ，masterGroupName参数设置，默认值是mymaster

```
[root@kubesphere pgo_install]# cd ../../
[root@kubesphere helm-harbor]# vim redis-ha/values.yaml
```

启动服务

```sh
[root@kubesphere helm-harbor]# kubectl create ns redis
namespace/redis created
[root@kubesphere helm-harbor]# cd redis-ha/
[root@kubesphere redis-ha]# helm install -n redis redis .
NAME: redis
LAST DEPLOYED: Wed Sep 14 16:56:12 2022
NAMESPACE: redis
STATUS: deployed
REVISION: 1
NOTES:
Redis can be accessed via port 6379   and Sentinel can be accessed via port 26379    on the following DNS name from within your cluster:
redis-redis-ha.redis.svc.cluster.local

To connect to your Redis server:
1. Run a Redis pod that you can use as a client:

   kubectl exec -it redis-redis-ha-server-0 sh -n redis

2. Connect using the Redis CLI:

  redis-cli -h redis-redis-ha.redis.svc.cluster.local
```

查看启动情况

```sh
[root@kubesphere helm-harbor]# kubectl get pods -n redis
NAME                      READY   STATUS            RESTARTS   AGE
redis-redis-ha-server-0   3/3     Running           0          2m46s
redis-redis-ha-server-1   3/3     Running           0          77s
redis-redis-ha-server-2   0/3     Running           0          20s

```

### 3. 部署harbor

> 采⽤helm部署⽅式，调⽤外置postgres 和redis ,提供镜像仓库服务
>
> 主要配置⽂件是 value.yaml ⽂件,部署harbor 需要获取postgres 和redis 的信息
>
> Postgres 数据库访问信息全部保存在 secret 中，获取⽅法如下:

- 获取password

```sh
[root@kubesphere helm-harbor]# kubectl get secrets -n postgres harbor-pguser-harbor  -o yaml |grep password
  password: LVV8PmdhSzJkbF1iQlY2RkFZZUxnOUdq
        f:password: {}
[root@kubesphere helm-harbor]# echo 'LVV8PmdhSzJkbF1iQlY2RkFZZUxnOUdq' |base64 --decode
-U|>gaK2dl]bBV6FAYeLg9Gj[root@kubesphere helm-harbor]#
```

- 获取 host 的 svc

```sh
[root@kubesphere helm-harbor]# kubectl get secrets -n postgres harbor-pguser-harbor  -o yaml |grep host
  host: aGFyYm9yLXByaW1hcnkucG9zdGdyZXMuc3Zj
  pgbouncer-host: aGFyYm9yLXBnYm91bmNlci5wb3N0Z3Jlcy5zdmM=
        f:host: {}
        f:pgbouncer-host: {}
[root@kubesphere helm-harbor]# echo 'aGFyYm9yLXBnYm91bmNlci5wb3N0Z3Jlcy5zdmM=' |base64 --decode
harbor-pgbouncer.postgres.svc[root@kubesphere helm-harbor]#
```

```sh
[root@kubesphere helm-harbor]# kubectl get svc -n redis
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)              AGE
redis-redis-ha              ClusterIP   None             <none>        6379/TCP,26379/TCP   11m
redis-redis-ha-announce-0   ClusterIP   10.105.195.214   <none>        6379/TCP,26379/TCP   11m
redis-redis-ha-announce-1   ClusterIP   10.110.75.243    <none>        6379/TCP,26379/TCP   11m
redis-redis-ha-announce-2   ClusterIP   10.108.121.23    <none>        6379/TCP,26379/TCP   11m
```

获取sentinelMasterSet

```sh
[root@kubesphere helm-harbor]# grep 'masterGroupName' redis-ha/values.yaml
  masterGroupName: "mymaster"       # must match ^[\\w-\\.]+$) and can be templated
```

```sh
检查事项：

1. Postgres 对接类型选择type: external ，默认值是internal

2. Postgres 的host配置: "harbor-primary.postgres.svc"， 与上⾯获取到pg host 信息保持⼀致

3. Postgres 的password配置: "-U|>gaK2dl]bBV6FAYeLg9Gj"， 与上⾯获取到pg password信息保

持⼀致

4. Postgres 的sslmode配置: "require"

5. Redis 对接类型选择type: external ，默认值是internal6. Redis 的addr 配置: "redis-redis-ha.redis.svc:26379"， 与上⾯获取到 redis svc信息保持⼀致

7. Redis 的sentinelMasterSet配置: "mymaster"， 与上⾯获取到 redis sentinelMasterSet信息保持⼀致

8. commonName: "192.168.20.120"   IP 地址和域名自己随意配置，后期 docker login  你喜欢用地址就写 IP，喜欢用域名就写域名

9. externalURL: https://192.168.20.120:30003      因为nodeport是30003，所以得加端口

如果外部访问方式是 ingress的话，那么会与需要以下配置

expose:
  type: ingress
  enabled: true
  certSource: auto
    #commonName: "k8s.harbor.com"
  secret:
    secretName: "k8s.harbor.com"
    notarySecretName: "k8s.harbor.com"
ingress:
  hosts:
    core: k8s.harbor.com
    notary: k8s.harbor.com

externalURL: https://k8s.harbor.com


```
准备部署

```sh
[root@kubesphere helm-harbor]# cd harbor/harbor-1.8.3
[root@kubesphere harbor-1.8.3]# helm install -n harbor harbor .
NAME: harbor
LAST DEPLOYED: Wed Sep 14 17:39:20 2022
NAMESPACE: harbor
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Please wait for several minutes for Harbor deployment to complete.
Then you should be able to visit the Harbor portal at https://127.0.0.1:30002
For more details, please visit https://github.com/goharbor/harbor
```

查看全部 chart

```sh
[root@kubesphere harbor-1.8.3]# helm ls --all-namespaces
NAME                         	NAMESPACE                   	REVISION	UPDATED                                	STATUS  	CHART                      	APP VERSION                 
elasticsearch-logging        	kubesphere-logging-system   	1       	2022-06-16 15:38:13.973770347 +0800 CST	deployed	elasticsearch-1.22.1       	6.7.0-0217                  
elasticsearch-logging-curator	kubesphere-logging-system   	1       	2022-06-16 15:38:15.344425265 +0800 CST	deployed	elasticsearch-curator-1.3.3	5.5.4-0217                  
harbor                       	harbor                      	1       	2022-09-14 17:39:20.797994887 +0800 CST	deployed	harbor-1.8.3               	2.4.3                       
jaeger-operator              	istio-system                	1       	2022-06-16 15:45:00.046575544 +0800 CST	deployed	jaeger-operator-2.26.0     	1.27.0                      
kiali-operator               	istio-system                	1       	2022-06-16 15:45:04.271879212 +0800 CST	deployed	kiali-operator-1.38.1      	v1.38.1                     
ks-core                      	kubesphere-system           	13      	2022-09-14 14:43:47.381365127 +0800 CST	deployed	ks-core-0.1.0              	v3.1.0                      
ks-events                    	kubesphere-logging-system   	1       	2022-06-16 15:40:57.237116725 +0800 CST	deployed	kube-events-0.3.0          	0.3.0                       
ks-minio                     	kubesphere-system           	1       	2022-06-16 15:36:59.762278651 +0800 CST	deployed	minio-2.5.16               	RELEASE.2019-08-07T01-59-21Z
ks-openldap                  	kubesphere-system           	1       	2022-06-16 15:36:50.49643425 +0800 CST 	deployed	openldap-ha-0.1.0          	1.0                         
kube-auditing                	kubesphere-logging-system   	1       	2022-06-16 15:40:31.352086304 +0800 CST	deployed	kube-auditing-0.2.0        	0.2.0                       
logsidecar-injector          	kubesphere-logging-system   	1       	2022-06-16 15:41:02.245932944 +0800 CST	deployed	logsidecar-injector-0.1.0  	0.1.0                       
notification-manager         	kubesphere-monitoring-system	1       	2022-06-16 15:42:35.048344949 +0800 CST	deployed	notification-manager-1.4.0 	1.4.0                       
pgo                          	pgo                         	1       	2022-09-14 16:42:09.543545043 +0800 CST	deployed	pgo-0.3.2                  	5.1.2                       
redis                        	redis                       	1       	2022-09-14 16:56:12.576117485 +0800 CST	deployed	redis-ha-4.17.4            	6.2.5                       
snapshot-controller          	kube-system                 	13      	2022-09-14 14:42:49.925999527 +0800 CST	deployed	snapshot-controller-0.2.0  	4.0.0
```

检查服务启动状况

```sh
[root@kubesphere ~]# kubectl get pods -n harbor -o wide
NAME                                 READY   STATUS    RESTARTS   AGE     IP               NODE         NOMINATED NODE   READINESS GATES
harbor-core-7758ff849b-g4g96         1/1     Running   0          9m36s   10.100.148.157   kubesphere   <none>           <none>
harbor-core-7758ff849b-gh9js         1/1     Running   0          9m36s   10.100.148.187   kubesphere   <none>           <none>
harbor-jobservice-54954ddddf-cf54d   1/1     Running   0          9m36s   10.100.148.158   kubesphere   <none>           <none>
harbor-jobservice-54954ddddf-wxks7   1/1     Running   0          9m36s   10.100.148.175   kubesphere   <none>           <none>
harbor-nginx-58b9b9779b-6zmrn        1/1     Running   0          9m36s   10.100.148.180   kubesphere   <none>           <none>
harbor-nginx-58b9b9779b-qpx8c        1/1     Running   0          9m36s   10.100.148.179   kubesphere   <none>           <none>
harbor-portal-76c5df56f-k2zdm        1/1     Running   0          9m36s   10.100.148.143   kubesphere   <none>           <none>
harbor-portal-76c5df56f-m2cc9        1/1     Running   0          9m36s   10.100.148.186   kubesphere   <none>           <none>
harbor-registry-5b6b787575-hkpcl     2/2     Running   0          9m36s   10.100.85.214    k8s-node01   <none>           <none>
harbor-registry-5b6b787575-n7k22     2/2     Running   0          9m36s   10.100.85.211    k8s-node01   <none>           <none>
```
![](/images/posts/Linux-Kubernetes/Kubernetes部署harbor服务/1.png)
![](/images/posts/Linux-Kubernetes/Kubernetes部署harbor服务/2.png)
![](/images/posts/Linux-Kubernetes/Kubernetes部署harbor服务/3.png)

### 4. docker login 证书配置

```sh
[root@kubesphere harbor-1.8.3]# cd /etc/docker/certs.d/192.168.20.120\:30003/
[root@kubesphere 192.168.20.120:30003]# kubectl get secrets  -n harbor harbor-nginx
NAME           TYPE     DATA   AGE
harbor-nginx   Opaque   3      9m32s
[root@kubesphere 192.168.20.120:30003]# kubectl get secrets  -n harbor harbor-nginx -o jsonpath="{.data.ca\.crt}" | base64 --decode
-----BEGIN CERTIFICATE-----
MIIDFDCCAfygAwIBAgIRALQ13zWYIFx4XJO+EXXVsvUwDQYJKoZIhvcNAQELBQAw
FDESMBAGA1UEAxMJaGFyYm9yLWNhMB4XDTIyMDkxNzA4NDMzNloXDTIzMDkxNzA4
NDMzNlowFDESMBAGA1UEAxMJaGFyYm9yLWNhMIIBIjANBgkqhkiG9w0BAQEFAAOC
AQ8AMIIBCgKCAQEA5dbCMx5aqJyNxkpHiPKCltP68VH2+r/1y/t767IiQ0l5jgAs
E/lKBTQZuYraNrb5vFsqADtH1CRxDqTEeJQaVSofK4qTIt3qM/4wWX8wJ6M1928M
zkonyikivwzz299G1L9oKU8KuHFcl48TtkCvH0esDfdbzztiHVCErOzwE82J6knG
A4m/4KUR9jA/501GtMZYXW4avRIXtuVg4QzxYkt2OBpJVBnZtiPzx4HpuCo5+gDn
1/auplT7r4bDHHPc2C6OaLMKcoSifXjyXmukyVJ7ftXCw9Y8Q584/Tr2U3c0OFG8
D619ibiotlrCEivsTF7b1Pbzi8Rl6ro8CREo3QIDAQABo2EwXzAOBgNVHQ8BAf8E
BAMCAqQwHQYDVR0lBBYwFAYIKwYBBQUHAwEGCCsGAQUFBwMCMA8GA1UdEwEB/wQF
MAMBAf8wHQYDVR0OBBYEFLiavau6ZHjUcKKpwZoKbtD89IfGMA0GCSqGSIb3DQEB
CwUAA4IBAQDivGbR6xZQN7A818Mj0CgXy8uOfTNV0HgGpp+2CVEVvQxCRmNfmgBW
YUbLAEgMfwBDukqxzdcG1BLKRgOOY9JPcwpq+XdnuctLcCtuzJzU1uvme61imELt
iiZaDMcCj1YvaA03fWw/cNeDp5p7POyrZGrGNoz0sEWc9ju3tV6i7bHM0/U59xHB
UhIhVhDknXLjOeM9/g8m49joPUhZSijTdk7OoL8MJbmpkxfGUYgD7ZenpVLOTc03
YBMWndP4qOGskxrPFi1tepB4nnEoLKlXuFvur6YdJ/5karaAkSGEPR3xVJBWOUh7
oM4F1rUw7ZCMdodvab7DC4vdjj/6yMuc
-----END CERTIFICATE-----
[root@kubesphere 192.168.20.120:30003]# vim ca.crt
-----BEGIN CERTIFICATE-----
MIIDFDCCAfygAwIBAgIRALQ13zWYIFx4XJO+EXXVsvUwDQYJKoZIhvcNAQELBQAw
FDESMBAGA1UEAxMJaGFyYm9yLWNhMB4XDTIyMDkxNzA4NDMzNloXDTIzMDkxNzA4
NDMzNlowFDESMBAGA1UEAxMJaGFyYm9yLWNhMIIBIjANBgkqhkiG9w0BAQEFAAOC
AQ8AMIIBCgKCAQEA5dbCMx5aqJyNxkpHiPKCltP68VH2+r/1y/t767IiQ0l5jgAs
E/lKBTQZuYraNrb5vFsqADtH1CRxDqTEeJQaVSofK4qTIt3qM/4wWX8wJ6M1928M
zkonyikivwzz299G1L9oKU8KuHFcl48TtkCvH0esDfdbzztiHVCErOzwE82J6knG
A4m/4KUR9jA/501GtMZYXW4avRIXtuVg4QzxYkt2OBpJVBnZtiPzx4HpuCo5+gDn
1/auplT7r4bDHHPc2C6OaLMKcoSifXjyXmukyVJ7ftXCw9Y8Q584/Tr2U3c0OFG8
D619ibiotlrCEivsTF7b1Pbzi8Rl6ro8CREo3QIDAQABo2EwXzAOBgNVHQ8BAf8E
BAMCAqQwHQYDVR0lBBYwFAYIKwYBBQUHAwEGCCsGAQUFBwMCMA8GA1UdEwEB/wQF
MAMBAf8wHQYDVR0OBBYEFLiavau6ZHjUcKKpwZoKbtD89IfGMA0GCSqGSIb3DQEB
CwUAA4IBAQDivGbR6xZQN7A818Mj0CgXy8uOfTNV0HgGpp+2CVEVvQxCRmNfmgBW
YUbLAEgMfwBDukqxzdcG1BLKRgOOY9JPcwpq+XdnuctLcCtuzJzU1uvme61imELt
iiZaDMcCj1YvaA03fWw/cNeDp5p7POyrZGrGNoz0sEWc9ju3tV6i7bHM0/U59xHB
UhIhVhDknXLjOeM9/g8m49joPUhZSijTdk7OoL8MJbmpkxfGUYgD7ZenpVLOTc03
YBMWndP4qOGskxrPFi1tepB4nnEoLKlXuFvur6YdJ/5karaAkSGEPR3xVJBWOUh7
oM4F1rUw7ZCMdodvab7DC4vdjj/6yMuc
-----END CERTIFICATE-----
[root@kubesphere 192.168.20.120:30003]# docker login 192.168.20.120:30003 -u admin -p Harbor12345
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

ingress 访问公钥

```sh
[root@kubesphere k8s.harbor.com:443]# kubectl get secrets -n harbor harbor-ingress -o jsonpath="{.data.ca\.crt}" | base64 --decode > ca.crt
[root@kubesphere k8s.harbor.com:443]# docker login  k8s.harbor.com:443 -u admin -p Harbor12345
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
[root@kubesphere ~]# docker tag nginx:latest k8s.harbor.com:443/library/nginx:latest
[root@kubesphere ~]# docker push k8s.harbor.com:443/library/nginx:latest
The push refers to repository [k8s.harbor.com:443/library/nginx]
d874fd2bc83b: Pushed
32ce5f6a5106: Pushed
f1db227348d0: Pushed
b8d6e692a25e: Pushed
e379e8aedd4d: Pushing [==================================================>]     62MB
2edcec3590a4: Pushing [======================================>            ]  61.93MB/80.37MB
```

### 5. CoreDNS 添加 hosts 解析

```sh
[root@kubesphere ~]# kubectl edit configmap -n kube-system coredns
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }

        hosts {
          192.168.20.120 k8s.harbor.com
          192.168.20.121 k8s.harbor.com
          fallthrough
.........................
```

重启 pod 进行验证

```sh
[root@kubesphere ~]# kubectl delete pods -n kube-system `kubectl get pods -n kube-system |grep coredns |awk {'print $1'}`
[root@kubesphere ~]# kubectl exec -it jekyll-6589bc94c7-fgttl -- bash
root@jekyll-6589bc94c7-fgttl:/# apt-get install -y inetutils-ping
root@jekyll-6589bc94c7-fgttl:/# ping k8s.harbor.com
PING k8s.harbor.com (192.168.20.120): 56 data bytes
64 bytes from 192.168.20.120: icmp_seq=0 ttl=64 time=0.166 ms
64 bytes from 192.168.20.120: icmp_seq=1 ttl=64 time=0.060 ms
64 bytes from 192.168.20.120: icmp_seq=2 ttl=64 time=0.135 ms
64 bytes from 192.168.20.120: icmp_seq=3 ttl=64 time=0.052 ms
^C--- k8s.harbor.com ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.052/0.103/0.166/0.049 ms
```

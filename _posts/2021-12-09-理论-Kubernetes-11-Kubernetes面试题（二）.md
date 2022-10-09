---
layout: post
title: 理论-Kubernetes-11-Kubernetes面试题（二）
date: 2021-12-09
tags: 理论-Kubernetes
---

## docker

1、dockerfile 怎么写？ 如何在镜像内指定某个用户去启动

> FROM nginx:latest
>
> USER nginx

2、容器老是自动重启，怎么处理

> - no，默认策略，在容器退出时不重启容器
> - on-failure，在容器非正常退出时（退出状态非0），才会重启容器
> - on-failure:3，在容器非正常退出时重启容器，最多重启3次
> - always，在容器退出时总是重启容器
> - unless-stopped，在容器退出时总是重启容器，但是不考虑在Docker守护进程启动时就已经停止了的容器

3、镜像特别大，应该怎么优化

> 制作镜像的时候减少镜像层，如：
>

```sh
RUN mkdir abc && cd abc && .....&&.......，一条指令是一层镜像，打包镜像的时候可以使用 docker save nginx:latest |gzip > nginx_latest.tar.gz
```

4、我自己搭建了一个harbor仓库，我想要拉取这个仓库里的镜像报错，说这个仓库使用的不是http协议，不让拉，应该怎么处理

> 编辑harbor.yml配置文件，注释掉https和443端口的配置

5、启动了一个容器，里面的应用是有日志的，但是通过 docker logs 看不到日志，是什么原因

> 可能是日志驱动造成的，需要配置daemon.json配置文件，添加"log-driver": "json-file"，或者启动容器的时候指定--log-driver=json-file

---

## k8s

1、service的port、nodeport、targetport有什么区别

> - port是service暴露出来pod的端口
> - nodeport是service暴露出到宿主机的端口
> - targetport是容器的端口

2、我的pod是host网络，发现他无法解析service，可能是什么原因

> ```sh
> ···
> spec:
>   template:
>     spec:
>       dnsPolicy: ClusterFirstWithHostNet     # 调整策略
> ···
> ```

**存储类**

4、有一天发现我的业务平台突然访问不了了，查看pod状态发现都是驱逐的状态，可能是什么原因导致的

> 可能是内存空间不足或者存储空间不足，多数情况下是存储空间不足，解决方法，扩建目录存储空间，或删除无用数据或镜像文件，或调整kubelet驱逐策略

5、某天，机器A上面save了一个镜像，拷贝到机器B并load进去，发现这个镜像没有tag，是什么原因

> docker save nginx:latest ea335eea17ab
>
> save的时候指定的镜像id导致的

6、configmap 怎么使用？secret 怎么使用，通过什么算法加密的？如何解码

> configmap也可以理解为存储，只是说挂载进去的是服务的配置文件，这个配置文件是自己在外面定义的k8s的资源，然后通过configmap挂载进去的。
>
> secret也是用来存储数据的，不过这个存储数据是把pod要访问的数据存储到了etcd中，然后用户通过pod的容器挂载获取到etcd中的信息

```sh
base64加密，解码：echo "xxxxx123123" |base64 --decode
```

7、podA 无法通过 service 访问 podB 了，这个要怎么排查？ 如果 service 和 pod 关联正常，还是无法访问呢？

> 首先可以简单的ping测试，测试pod IP之间是否通信，如果通信的话，在测试cluster.local是否通信，还通信的话就证明pod之前的是正常的，如果不通信的话可以查看proxy生成的iptables规则或者ipvs是否正确。最后可以检查一下pod里面的服务是否正常启动，端口是否正常监听。

8、我新建了一个 pod，发现他一直是 pending 的状态，怎么处理？ 如果不是标签和污点导致，还可能是什么原因，describe 也没有信息

> pod 存储类，是不是用的pv或者pvc，如果用的pv或pvc，那么有没有对应的pv或者pvc资源，没有的话就需要创建好pv或者pvc资源
>
> 亲和性，如果是亲和性的问题也会导致pod处于pending状态
>
> 暴露的端口被占用也会pending，以及资源被占用也会pending，如：显卡资源不足

**pod 亲和性**

10、健康检查有哪几种？监测方式又有哪几种？如果我服务启动特别慢，这个健康检查应该怎么做才合适

> Liveness 未探测出预想结果是重启容器；Readiness 未探测出预想结果则是将容器设置为不可用，不接收 Service 转发的请求
>
> 利用readiness去探测服务端口是否存在，此时就会一直等待服务启动，直至pod READY 为1/1

11、daemonset控制器会和宿主机的哪几个名称空间共享呢

---

## Shell

1、写个脚本，如何批量修改文件中的同名字符串，都有哪几种方式

> ```sh
> #!/bin/bash
>
> for ((j=1;j<=18;j++))
> do
> if [ $j -lt 10 ]
> then
> abc=$((0))$((j))
> else
> abc=$((j))
> fi
> echo $abc
> #将$abc路径下tonc.m文件中的字符串92变换成字符串90
> sed -i "s/92/90/g" `grep 92 -rl $abc/tonc.m`
> done
> ```

2、如何批量删除k8s异常pod

> ```sh
> #!/bin/bash
> # 过虑出异常pod的namespace
> namespaces=`kubectl get pod -A | grep -i "evicted" | awk '{print $1}'`
> for namespace in ${namespaces}
> do
> # 查看异常pod的namespace资源，然后过虑出异常pod的名称，然后将输出的结果“pod 名称”作为xargs 的参数
> kubectl get pod -n ${namespace} |grep -i "evicted"|awk '{print $1}' | xargs kubectl delete pod -n ${namespace}
> done
> ```

3、简述一下标准输入和标准输出以及标准错误输出的区别，怎么使用

4、如何通过脚本看某一个时间段哪个IP访问nginx次数最多呢

5、将一个文本根据第七列进行一个排序，并且去重

---

## mysql

1、mysql 主从复制如何实现的

> - MySQL 从服务器开启 I/O 线程，向主服务器请求数据同步(获取二进制日志)
> - MySQL 主服务器开启 I/O 线程回应从服务器
> - 从服务器得到主的二进制日志写入中继日志
> - 从服务器开启 SQL 线程将日志内容执行，实现数据同步

2、发现数据库查询延迟特别高，这个应该怎么排查

3、查询 test 库的 test 表中，test列包含字符 test 的条目

---

**nginx**

1、承当什么角色，用来做什么？反向代理和正向代理有啥区别？反向代理和负载均衡又有啥区别

> nginx是web服务，用来提供前端访问的。
>
> 反向代理：是，把所有的后端资源归拢到代理机器上，然后客户端对代理端进行访问。
>
> 正向代理：是，客户端访问代理端，代理端去请求最终端的资源，然后再由代理段反馈给客户端
>
> Nginx反向代理服务器接收到的请求数量，就是我们说的负载量，请求数量按照一定的规则进行分发到不同的服务器处理的规则，就是一种均衡规则

2、nginx 的虚拟主机如何实现？ master 进程和 worker 进程有什么区别。16c的机器配置多少 worker 合适

> 默认情况下在conf.d目录下添加一个.conf的文件，然后就是添加虚拟主机。
>
> “master"进程其实是负责管理"worker"进程的，除了管理” worker"进程，master"进程还负责读取配置文件、判断配置文件语法的工作，“master进程"也叫"主进程”
>
> "worker"进程天生就是来"干活"的，**真正负责处理请求的进程就是你看到的"worker"进程**
>
> **"master"进程只能有一个,而"worker"进程可以有多个**，worker"进程的数量可以由管理员自己进行定义，worker的多少取决于CPU数量的多少，16C的那么最多可以设置16个worker

3、如何将 nginx 平滑升级

> 备份旧的nginx可执行文件，然后拷贝新的(升级的)nginx可执行文件，nginx -t 检查  ps -ef 检查旧的nginx进程，然后干掉旧的master进程

---

## redis

1、redis 都有哪几种集群方式，如何配置？用来做什么

> 三种模式：主从、哨兵、cluster。
edis的主从和哨兵两种集群方案，redis从3.0版本开始引入了redis-cluster(集群)。从主从-哨兵-集群可以看到redis的不断完善；主从复制是最简单的节点同步方案无法主从自动故障转移。
哨兵可以同时管理多个主从同步方案同时也可以处理主从自动故障转移，通过配置多个哨兵节点可以解决单点网络故障问题，但是单个节点的性能压力问题无法解决。
集群解决了前面两个方案的所有问题。

2、分片集群有多少槽位？每个槽位存储的数据有大小限制吗？连接到 slave 可以写数据吗

> 16383个，8KB

3、备份如何实现



---

## linux

1、系统DNS怎么配置，其中的搜索域是做什么用的

> /etc/resolv.conf   作用就是，你能访问到哪写网站的域名，就是由搜索域决定的

2、如何做免密登录

> 创建用户A，登录到用户A中，ssh-keygen，一路回车生成一些私钥和公钥，然后ssh-copy-id 跟上 要免密的机器IP，如ssh-copy-id 192.168.100.100
>
> 配置好之后下次登录就不需要密码了

3、发现操作系统内存和cpu占用特别高，这个应该怎么处理

> 可以执行top命令查看那些进程占用资源较多，然后判断改服务是否是自己的服务，如果是可以根据该服务以往的运行特性来判断占用资源过高是不是属于正常范畴，解决方式可以优化该服务的性能。如果不是自己的服务造成的系统负载过高，那么可以直接干掉pid。如果是被挖矿了。那就需要一点点的排查根源了。

4、sudo 提权，su -  和 su，安全加固

> su - 会改变工作目录和PATH变量，不带  -   不会改变，只是拥有了root权限。
>
> 默认情况下，系统只有root用户可以执行sudo命令。需要root用户通过使用visudo命令编辑sudo的配置文件/etc/sudoers，才可以授权其他普通用户执行sudo命令，编辑visudo进行安全加固配置

5、开机自动挂载磁盘怎么做

> /etc/fstab

6、systemd  pid 为什么是 1

> 因为从BIOS进入系统的最后一步就是init初始化，这个进程是系统的第一个进程，PID 为 1，又叫超级进程。也叫根进程

7、如何查看某个进程的启动时间

> ```sh
> $ ps -eo pid,lstart,etime,cmd |grep nginx
> ```

8、如何将 yum rpm 下载到本地

> ```sh
> $ yum install --downloadonly --downloaddir=/
> ```

9、iptables 防火墙封禁端口

> ```sh
> $ iptables -I INPUT -p tcp --dport 80 -j DROP
> ```

10、如何查看系统的已连接数是多少呢

> ```sh
> # 查看端口连接
> $ netstat -lntp
> # 查看用户登录
> $ w 或者 who
> ```

11、系统空间莫名被占用了，这个怎么排查



12、僵尸进程

> 使用Top命令查找，当zombie前的数量不为0时，即系统内存在相应数量的僵尸进程
>
> ```sh
> $ ps -A -ostat,ppid,pid,cmd |grep -e '^[Zz]'
>
> Z     9511  9565 [nginx] <defunct>
>
> $ kill -9 9511
> ```

13、too many open files

14、监控到好多连接等待，如何排查，如何处理

---

## 日常问题

1、前端报 502 是什么原因导致的，怎么排查？ 500、404

> 502是后端服务不可达，需要查看代理和后端之间的路由问题
>
> 500报错是是服务内部问题，比如，cpu内存爆满，或者数据库负载或数据库缺少语句等问题
>
> 404是缺少资源，找不到资源

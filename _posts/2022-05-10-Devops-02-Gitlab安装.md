---
layout: post
title: Devops-02-Gitlab安装
date: 2022-05-10
tags: Devops
---
## 一、gitlab介绍

> GitLab 是一个用于仓库管理系统的开源项目，使用Git作为代码管理工具，并在此基础上搭建起来的web服务。GitLab由乌克兰程序员DmitriyZaporozhets和ValerySizov开发，它由Ruby写成。后来，一些部分用Go语言重写，现今并在国内外大中型互联网公司广泛使用。
>
> git、gitlab、GitHub的简单区别
>
> git 是一种基于命令的版本控制系统，全命令操作，没有可视化界面
>
> gitlab 是一个基于git实现的在线代码仓库软件，提供web可视化管理界面，通常用于企业团队内部协作开发
>
> github 是一个基于git实现的在线代码托管仓库，亦提供可视化管理界面，同时免费账户和提供付费账户，提供开放和私有的仓库，大部分的开源项目都选择
>
> github作为代码托管仓库

## 二、gitlab相关命令

> gitlab-ctl start #启动全部服务
>
> gitlab-ctl restart#重启全部服务
>
> gitlab-ctl stop #停止全部服务
>
> gitlab-ctl restart nginx #重启单个服务，如重启nginx
>
> gitlab-ctl status #查看服务状态
>
> gitlab-ctl reconfigure #使配置文件生效
>
> gitlab-ctl show-config #验证配置文件
>
> gitlab-ctl uninstall #删除gitlab（保留数据）
>
> gitlab-ctl cleanse #删除所有数据，从新开始
>
> gitlab-ctl tail <service name>查看服务的日志
>
> gitlab-ctl tail nginx  #如查看gitlab下nginx日志
>
> gitlab-rails console  #进入控制台

## 三、gitlab常用组件

> nginx：静态Web服务器
>
> gitlab-shell：用于处理Git命令和修改authorized keys列表，gitlab是以Git为底层的，操作实际上最后就是调用gitlab-shell命令进行处理。
>
> gitlab-workhorse:轻量级的反向代理服务器
>
> logrotate：日志文件管理工具
>
> postgresql：数据库
>
> redis：缓存数据库
>
> sidekiq：用于在后台执行队列任务（异步执行）
>
> unicorn：GitLab Rails应用是托管在这个服务器上面的

## 四、k8s安装部署

### 1. 准备storageclass做持久化存储

过程略过，可以参考之前的**nfs-provisioner**，这篇文章有教你如何做持久化存储：https://blog.linuxtian.top/2021/12/Linux-Kubernetes-43-Kubernetes持久化存储实战(一)/

> 注意：准备nfs环境的时候，`/etc/exports`配置文件中，如：`/home/volume_nfs 192.168.1.20(rw,no_root_squash)`地址和后面的()不能有空格，否则pod创建报错`read-only file system`，还有一点就是，我这次的实验环境是3台机器，所以其他两台机去需要挂在nfs存储，以免pod在不同节点运行的时候进行nfs挂载找不到目录而报错
>
> 但是，这是仅限于创建nfs存储类型的pvc这样使用，如果想要手动mount -t nfs ，就必须要有空格，此时可以写两行内容，一行有空格一行没空格

### 2. 准备redis资源清单

首先创建一个namespace

```sh
[root@k8s-master01 ~]# cd /data/devops/gitlab/
[root@k8s-master01 gitlab]#  kubectl create ns dev-op
```
```yaml
[root@k8s-master01 gitlab]# cat > namespace.yaml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: gitlab
EOF
[root@k8s-master01 gitlab]# cat > gitlab-redis-pvc.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-redis-pvc
  namespace: gitlab
  annotations:
    volume.beta.kubernetes.io/storage-class: "nfs-provisioner-storage"
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
EOF
[root@k8s-kubersphere redis]# cat > gitlab-redis.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: gitlab
  labels:
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: sameersbn/redis
        imagePullPolicy: IfNotPresent
        ports:
        - name: redis
          containerPort: 6379
        volumeMounts:
        - mountPath: /var/lib/redis
          name: data
        livenessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 30
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 5
          timeoutSeconds: 1
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: gitlab-redis-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: gitlab
  labels:
    app: redis
spec:
  ports:
    - name: redis
      port: 6379
      targetPort: 6379
  selector:
    app: redis
EOF
```

交付资源

```sh
[root@k8s-kubersphere redis]# kubectl apply -f .
```

### 3. 准备postgres资源清单

```yaml
[root@k8s-kubersphere postgres]# ls
gitlab-postgresql-pvc.yaml  gitlab-postgresql.yaml
[root@k8s-kubersphere postgres]# cat > gitlab-postgresql-pvc.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-postgresql-pvc
  namespace: gitlab
  annotations:
    volume.beta.kubernetes.io/storage-class: "nfs-provisioner-storage"
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
EOF
[root@k8s-kubersphere postgres]# cat > gitlab-postgresql.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql
  namespace: gitlab
  labels:
    app: postgresql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
      - name: postgresql
        image: sameersbn/postgresql
        imagePullPolicy: IfNotPresent
        env:
        - name: DB_USER
          value: gitlab
        - name: DB_PASS
          value: passw0rd
        - name: DB_NAME
          value: gitlab_production
        - name: DB_EXTENSION
          value: pg_trgm
        ports:
        - name: postgresql
          containerPort: 5432
        volumeMounts:
        - mountPath: /var/lib/postgresql
          name: data
        livenessProbe:
          tcpSocket:
            port: 5432
          initialDelaySeconds: 30
          timeoutSeconds: 5
        readinessProbe:
          tcpSocket:
            port: 5432
          initialDelaySeconds: 5
          timeoutSeconds: 1
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: gitlab-postgresql-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgresql
  namespace: gitlab
  labels:
    app: postgresql
spec:
  ports:
    - name: postgresql
      port: 5432
      targetPort: 5432
  selector:
    app: postgresql
EOF
```

交付资源

```sh
[root@k8s-kubersphere postgres]# kubectl apply -f .
```

### 4. 准备gitlab主服务资源清单

```yaml
[root@k8s-kubersphere gitlab]# ls
gitlab-pvc.yaml  gitlab_secrets.yaml  gitlab.yaml
[root@k8s-kubersphere gitlab]# cat > gitlab-pvc.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-pvc
  namespace: gitlab
  annotations:
    volume.beta.kubernetes.io/storage-class: "nfs-provisioner-storage"
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
EOF
# 创建一个secret密钥，用来web登陆的用户名密码
[root@k8s-kubersphere gitlab]# echo -n 'root' | base64
cm9vdA==
[root@k8s-kubersphere gitlab]# echo -n 'root123123' | base64
cm9vdDEyMzEyMw==
# 解码验证一下
[root@k8s-kubersphere gitlab]# echo 'cm9vdDEyMzEyMw==' | base64 --decode
[root@k8s-kubersphere gitlab]# cat > gitlab_secrets.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  namespace: gitlab
  name: git-user-pass
type: Opaque
data:
  username: cm9vdA==
  password: cm9vdDEyMzEyMw==
EOF
[root@k8s-kubersphere gitlab]# cat > gitlab.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitlab
  namespace: gitlab
  labels:
    app: gitlab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitlab
  template:
    metadata:
      labels:
        app: gitlab
    spec:
      containers:
      - name: gitlab
        image: sameersbn/gitlab:12.1.6
        imagePullPolicy: IfNotPresent
        env:
        - name: TZ
          value: Asia/Shanghai
        - name: GITLAB_TIMEZONE
          value: Beijing
        - name: GITLAB_SECRETS_DB_KEY_BASE
          value: long-and-random-alpha-numeric-string
        - name: GITLAB_SECRETS_SECRET_KEY_BASE
          value: long-and-random-alpha-numeric-string
        - name: GITLAB_SECRETS_OTP_KEY_BASE
          value: long-and-random-alpha-numeric-string
        - name: GITLAB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: git-user-pass
              key: password
        - name: GITLAB_ROOT_EMAIL
          value: 2099637909@qq.com
        - name: GITLAB_HOST
          value: gitlab.k8s.com
        - name: GITLAB_PORT
          value: "80"
        - name: GITLAB_SSH_PORT
          value: "30022"
        - name: GITLAB_NOTIFY_ON_BROKEN_BUILDS
          value: "true"
        - name: GITLAB_NOTIFY_PUSHER
          value: "false"
        - name: GITLAB_BACKUP_SCHEDULE
          value: daily
        - name: GITLAB_BACKUP_TIME
          value: 01:00
        - name: DB_TYPE
          value: postgres
        - name: DB_HOST
          value: postgresql
        - name: DB_PORT
          value: "5432"
        - name: DB_USER
          value: gitlab
        - name: DB_PASS
          value: passw0rd
        - name: DB_NAME
          value: gitlab_production
        - name: REDIS_HOST
          value: redis
        - name: REDIS_PORT
          value: "6379"
        ports:
        - name: http
          containerPort: 80
        - name: ssh
          containerPort: 22
        resources:
          limits:
            cpu: 1000m
            memory: 1Gi
          requests:
            cpu: 500m
            memory: 512Mi
        volumeMounts:
        - mountPath: /home/git/data
          name: data
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 180
          timeoutSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          timeoutSeconds: 1
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: gitlab-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: gitlab
  namespace: gitlab
  labels:
    app: gitlab
spec:
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30080
    - name: ssh
      port: 22
      targetPort: 22
      nodePort: 30022
  type: NodePort
  selector:
    app: gitlab
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: gitlab
  namespace: gitlab
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.frontend.rule.type: PathPrefixStrip
spec:
  rules:
  - host: gitlab.k8s.com
    http:
      paths:
      - path: /
        backend:
          serviceName: gitlab
          servicePort: 80
EOF
```

ingress 安装参考：https://blog.linuxtian.top/2021/08/Linux-Kubernetes-31-高版本安装traefik和dashboard/

交付资源

```sh
[root@k8s-kubersphere gitlab]# kubectl apply -f .
```

查看pod运行状态

```sh
[root@k8s-kubersphere gitlab]# kubectl get pods,svc -n gitlab
NAME                              READY   STATUS    RESTARTS   AGE
pod/gitlab-6dd5dc965b-7vg8r       1/1     Running   1          42m
pod/postgresql-5d5d6bc448-x9pbt   1/1     Running   2          44m
pod/redis-dc77f47d8-lsk28         1/1     Running   0          57m

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                     AGE
service/gitlab       NodePort    10.102.142.173   <none>        80:30080/TCP,22:30022/TCP   42m
service/postgresql   ClusterIP   10.107.39.86     <none>        5432/TCP                    46m
service/redis        ClusterIP   10.101.243.65    <none>        6379/TCP                    57m
```

## 五、浏览器登陆

可以使用nodePort端口访问，也可以使用ingress访问，登陆用户名root密码root123123

![](/images/posts/Linux-Kubernetes/k8s_gitlab/1.png)


## 六、docker-compose方式启动

```sh
[root@localhost gitlab]# cat docker-compose.yml
version: '3'
services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    restart: always
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://gitlab.k8s.com:80' #若有域名可以写域名
        gitlab_rails['gitlab_shell_ssh_port'] = 2224
    ports:
      - '80:80'  # 左右必须一样
      - '2224:22'
    # 映射本地解析到容器，后期gitlab要以webhook方式向jenkins推送消息
    extra_hosts:
      - 'jenkins.k8s.com:192.168.20.51'
    volumes:
      #将相关配置映射到当前目录下的config目录
      - './config:/etc/gitlab'
      #将日志映射到当前目录下的logs目录
      - './logs:/var/log/gitlab'
      #将数据映射到当前目录下的data目录
      - './data:/var/opt/gitlab'
```

用户名root，初始密码如下

```sh
[root@localhost gitlab]# ls
config  data  docker-compose.yml  gitlab-ce_latest.tar.gz  logs
[root@localhost gitlab]# cat config/initial_root_password
# WARNING: This value is valid only in the following conditions
#          1. If provided manually (either via `GITLAB_ROOT_PASSWORD` environment variable or via `gitlab_rails['initial_root_password']` setting in `gitlab.rb`, it was provided before database was seeded for the first time (usually, the first reconfigure run).
#          2. Password hasn't been changed manually, either via UI or via command line.
#
#          If the password shown here doesn't work, you must reset the admin password following https://docs.gitlab.com/ee/security/reset_user_password.html#reset-your-root-password.

Password: ztbESjJUD3YvjMyM/eMlEDCCZr4CFN+0nSGIUMWW8ew=

# NOTE: This file will be automatically deleted in the first reconfigure run after 24 hours.
```

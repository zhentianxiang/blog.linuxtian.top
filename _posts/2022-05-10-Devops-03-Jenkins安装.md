---
layout: post
title: Devops-03-Jenkins安装
date: 2022-05-10
tags: Devops
---

## 一、服务部署

### 1. 准备PVC资源清单

```yaml
[root@k8s-master01 jenkins]#  kubectl get sc
NAME                                PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-provisioner-storage (default)   example.com/nfs   Delete          Immediate           false                  2d22h
[root@k8s-master01 jenkins]# cat > namespace.yaml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins
EOF
[root@k8s-master01 jenkins]# cat > jenkins_pvc.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
  namespace: jenkins
  annotations:
    volume.beta.kubernetes.io/storage-class: "nfs-provisioner-storage"
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
EOF
```

### 1.2 准备rbac鉴权

```yaml
[root@k8s-master01 jenkins]# cat > jenkins_rbac.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-sa
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: jenkins-crd
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: jenkins-sa
  namespace: jenkins
EOF
```

### 1.3 准备Jenkins服务资源

```yaml
[root@k8s-master01 jenkins]# cat jenkins.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
  labels:
    app: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      hostAliases:
      - ip: "192.168.20.55"   # 因为我们使用域名看起来比较高大上，所以需要配置一下pod的host解析
        hostnames:
        - "gitlab.k8s.com"
      - ip: "192.168.20.52"   # 因为我们使用域名看起来比较高大上，所以需要配置一下pod的host解析
        hostnames:
        - "sonarqube.k8s.com"
      terminationGracePeriodSeconds: 10
      serviceAccount: jenkins-sa
      containers:
      - name: jenkins
        image: jenkins/jenkins:2.319.1-lts
        imagePullPolicy: IfNotPresent
        env:
        - name: JAVA_OPTS
          value: -XshowSettings:vm -Dhudson.slaves.NodeProvisioner.initialDelay=0 -Dhudson.slaves.NodeProvisioner.MARGIN=50 -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85 -Duser.timezone=Asia/Shanghai
        ports:
        - containerPort: 8080
          name: web
          protocol: TCP
        - containerPort: 50000
          name: agent
          protocol: TCP
        resources:
          limits:
            cpu: 1000m
            memory: 1Gi
          requests:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
          failureThreshold: 12
        readinessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
          failureThreshold: 12
        volumeMounts:
        - name: jenkinshome
          mountPath: /var/jenkins_home
        - name: docker-sock
          mountPath: /run/docker.sock
        - name: docker-command
          mountPath: /usr/bin/docker
      securityContext:
        fsGroup: 1000
      volumes:
      - name: jenkinshome
        persistentVolumeClaim:
          claimName: jenkins-pvc
      - name: docker-sock
        hostPath:
          path: /run/docker.sock
          type: ''
      - name: docker-command
        hostPath:
          path: /usr/bin/docker
          type: ''
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
  labels:
    app: jenkins
spec:
  selector:
    app: jenkins
  type: NodePort
  ports:
  - name: web
    port: 8080
    targetPort: 8080
    nodePort: 30002
  - name: agent
    port: 50000
    targetPort: 50000
---
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: jenkins
    namespace: jenkins
    annotations:
      kubernetes.io/ingress.class: traefik
      traefik.frontend.rule.type: PathPrefixStrip
  spec:
    rules:
    - host: jenkins.k8s.com
      http:
        paths:
        - path: /
          backend:
            serviceName: jenkins
            servicePort: 8080
```

### 1.4 交付资源

```sh
[root@k8s-master01 jenkins]# kubectl apply -f .
persistentvolumeclaim/jenkins-pvc created
serviceaccount/jenkins-sa created
clusterrole.rbac.authorization.k8s.io/jenkins-cr created
clusterrolebinding.rbac.authorization.k8s.io/jenkins-crd created
deployment.apps/jenkins created
service/jenkins created
[root@k8s-master01 jenkins]# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                          STORAGECLASS       REASON   AGE
pvc-3b3cc060-269f-4de5-b500-a816f6e241a7   1Gi        RWX            Delete           Bound    gitlab/gitlab-postgresql-pvc   do-block-storage            13h
pvc-69d916cc-863d-44d6-b621-372f7f794528   1Gi        RWX            Delete           Bound    gitlab/gitlab-redis-pvc        do-block-storage            14h
pvc-7e1828e8-27a5-4df2-a53a-c87c5fba79fd   1Gi        RWX            Delete           Bound    gitlab/gitlab-pvc              do-block-storage            13h
pvc-fb2e10c3-eb14-4a25-9d72-74b258527c20   10Gi       RWX            Delete           Bound    jenkins/jenkins-pvc            do-block-storage            4m9s
[root@k8s-master01 jenkins]# kubectl get pvc -n jenkins
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
jenkins-pvc   Bound    pvc-fb2e10c3-eb14-4a25-9d72-74b258527c20   10Gi       RWX            do-block-storage   4m18s
[root@k8s-master01 jenkins]# kubectl get pods,svc -n jenkins
NAME                           READY   STATUS    RESTARTS   AGE
pod/jenkins-5596c4d587-p2wbs   1/1     Running   0          3m37s

NAME              TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                          AGE
service/jenkins   NodePort   10.104.215.244   <none>        8080:30002/TCP,50000:12405/TCP   3m37s
[root@k8s-master01 jenkins]# kubectl get endpoints -n jenkins jenkins
NAME      ENDPOINTS                              AGE
jenkins   10.100.46.55:8080,10.100.46.55:50000   4m1s
[root@k8s-master01 jenkins]# kubectl get ingress -n jenkins
NAME      CLASS    HOSTS             ADDRESS   PORTS   AGE
jenkins   <none>   jenkins.k8s.com             80      9s
```

### 1.5 配置jdk和maven

准备好jdk8和maven3.6.3

https://www.oracle.com/java/technologies/downloads/#java8

https://maven.apache.org/index.html

```sh
[root@k8s-master01 jenkins]#  ls
apache-maven-3.6.3-bin.tar.gz  jdk-8u231-linux-x64.tar.gz
[root@k8s-master01 jenkins]# tar xvf jdk-8u231-linux-x64.tar.gz
[root@k8s-master01 jenkins]# tar xvf apache-maven-3.6.3-bin.tar.gz
[root@k8s-master01 jenkins]# mv jdk1.8.0_231 jdk
[root@k8s-master01 jenkins]# mv apache-maven-3.6.3 maven
# 配置构建源，146行追加
[root@k8s-master01 jenkins]# vim maven/conf/settings.xml
135   <!-- mirrors
136    | This is a list of mirrors to be used in downloading artifacts from remote repositories.
137    |
138    | It works like this: a POM may declare a repository to use in resolving certain artifacts.
139    | However, this repository may have problems with heavy traffic at times, so people have mirrored
140    | it to several places.
141    |
142    | That repository definition will have a unique id, so we can create a mirror reference for that
143    | repository, to be used as an alternate download site. The mirror site will be the preferred
144    | server for that repository.
145    |-->
146   <mirrors>
147     <mirror>
148       <id>nexus-aliyun</id>
149       <mirrorOf>*</mirrorOf>
150       <name>Nexus aliyun</name>
151       <url>http://maven.aliyun.com/nexus/content/groups/public</url>
152     </mirror>
153     <!-- mirror
154      | Specifies a repository mirror site to use instead of a given repository. The repository that
155      | this mirror serves has an ID that matches the mirrorOf element of this mirror. IDs are used
156      | for inheritance and direct lookup purposes, and must be unique across the set of mirrors.
157      |
158     <mirror>
159       <id>mirrorId</id>
160       <mirrorOf>repositoryId</mirrorOf>
161       <name>Human Readable Name for this Mirror.</name>
162       <url>http://my.repository.com/repo/path</url>
163     </mirror>
164      -->
165   </mirrors>
# 配置jdk和sonar，252行追加
252     -->
253   <profile>
254     <id>jdk8</id>
255     <activation>
256         <activeByDefault>true</activeByDefault>
257         <jdk>1.8</jdk>
258     </activation>
259     <properties>
260         <maven.compiler.source>1.8</maven.compiler.source>
261         <maven.compiler.target>1.8</maven.compiler.target>
262         <maven.compiler.compilerVersion>1.8</maven.compiler.compilerVersion>
263     </properties>
264 </profile>
265
266   <profile>
267       <id>sonar</id>
268       <activation>
269           <activeByDefault>true</activeByDefault>
270       </activation>
271       <properties>
272           <sonar.login>admin</sonar.login>
273           <sonar.password>root123123</sonar.password>
274           <sonar.host.url>http://sonarqube.k8s.com/</sonar.host.url>
275         </properties>
276     </profile>
277   </profiles>
278
279   <!-- activeProfiles
280    | List of profiles that are active for all builds.
281    |
282   <activeProfiles>
283     <activeProfile>alwaysActiveProfile</activeProfile>
284     <activeProfile>anotherAlwaysActiveProfile</activeProfile>
285   </activeProfiles>
286   -->
287   <activeProfiles>
288     <activeProfile>jdk8</activeProfile>
289     <activeProfile>sonar</activeProfile>
290   </activeProfiles>
291
292 </settings>
```

找到pvc的存储目录，然后把文件弄进去，只要pvc不被删除，数据会永远保留

当然，也可以使用dockerfile自己制作一下镜像

```sh
[root@k8s-master01 jenkins]# mv maven /home/volume_nfs/jenkins-jenkins-pvc-pvc-fb2e10c3-eb14-4a25-9d72-74b258527c20/
[root@k8s-master01 jenkins]# mv jdk /home/volume_nfs/jenkins-jenkins-pvc-pvc-fb2e10c3-eb14-4a25-9d72-74b258527c20/
```

### 1.6 配置 sonarqube

https://docs.sonarqube.org/latest/analysis/scan/sonarscanner/

```sh
[root@k8s-master01 ~]# cd /data/nfs_pvc/jenkins-jenkins-pvc-pvc-01eb06ee-11d8-4b91-9e11-bd2b1c342154/
[root@k8s-master01 jenkins-jenkins-pvc-pvc-01eb06ee-11d8-4b91-9e11-bd2b1c342154]# wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.7.0.2747-linux.zip
[root@k8s-master01 jenkins-jenkins-pvc-pvc-01eb06ee-11d8-4b91-9e11-bd2b1c342154]# unzip sonar-scanner-cli-4.7.0.2747-linux.zip
[root@k8s-master01 jenkins-jenkins-pvc-pvc-01eb06ee-11d8-4b91-9e11-bd2b1c342154]# mv sonar-scanner-4.7.0.2747-linux/ sonar-scanner
[root@k8s-master01 jenkins-jenkins-pvc-pvc-01eb06ee-11d8-4b91-9e11-bd2b1c342154]# vim sonar-scanner/conf/sonar-scanner.properties
#Configure here general information about the environment, such as SonarQube server connection details for example
#No information about specific project should appear here

#----- Default SonarQube server
sonar.host.url=http://sonarqube.k8s.com/

#----- Default source code encoding
sonar.sourceEncoding=UTF-8

```


### 1.7 登陆Jenkins

密码就是pvc存储里面的这个
```sh
[root@k8s-master01 jenkins]# cat /home/volume_nfs/jenkins-jenkins-pvc-pvc-fb2e10c3-eb14-4a25-9d72-74b258527c20/secrets/initialAdminPassword
fac9196d916e4703919665620515edce
```

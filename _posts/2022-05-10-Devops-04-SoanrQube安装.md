---
layout: post
title: Devops-04-SoanrQube安装
date: 2022-05-10
tags: Devops
---

```sh
[root@k8s-master01 ~]# cd /data/devops/sonarqube/
[root@k8s-master01 sonarqube]# cat > namespace.yaml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: sonarqube
EOF
[root@k8s-master01 sonarqube]# cat > postgresql-dep.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-sonar
  namespace: sonarqube
  labels:
    app: postgres-sonar
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres-sonar
  template:
    metadata:
      labels:
        app: postgres-sonar
    spec:
      containers:
      - name: postgres-sonar
        image: 192.168.20.54/postgresql/postgresql:11.4
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: "sonarDB"
        - name: POSTGRES_USER
          value: "sonarUser"
        - name: POSTGRES_PASSWORD
          value: "123456"
        resources:
          limits:
            cpu: 1000m
            memory: 2048Mi
          requests:
            cpu: 500m
            memory: 1024Mi
        volumeMounts:
          - name: data
            mountPath: /var/lib/postgresql/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: postgres-sonar-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-sonar
  namespace: sonarqube
  labels:
    app: postgres-sonar
spec:
  clusterIP: None
  ports:
  - port: 5432
    protocol: TCP
    targetPort: 5432
  selector:
    app: postgres-sonar
EOF
```

```sh
[root@k8s-master01 sonarqube]# cat > postgresql-pvc.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-sonar-pvc
  namespace: sonarqube
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

```sh
[root@k8s-master01 sonarqube]# cat > sonarqube-dep.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sonarqube
  namespace: sonarqube
  labels:
    app: sonarqube
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sonarqube
  template:
    metadata:
      labels:
        app: sonarqube
    spec:
      initContainers:
      - name: init-sysctl
        image: 192.168.20.54/library/busybox:latest
        imagePullPolicy: IfNotPresent
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      containers:
      - name: sonarqube
        image: 192.168.20.54/sonarqube/sonarqube:8.9.6-community
        ports:
        - containerPort: 9000
        env:
        - name: SONARQUBE_JDBC_USERNAME
          value: "sonarUser"
        - name: SONARQUBE_JDBC_PASSWORD
          value: "123456"
        - name: SONARQUBE_JDBC_URL
          value: "jdbc:postgresql://postgres-sonar:5432/sonarDB"
        livenessProbe:
          httpGet:
            path: /sessions/new
            port: 9000
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /sessions/new
            port: 9000
          initialDelaySeconds: 60
          periodSeconds: 30
          failureThreshold: 6
        resources:
          limits:
            cpu: 2000m
            memory: 2048Mi
          requests:
            cpu: 1000m
            memory: 1024Mi
        volumeMounts:
        - mountPath: /opt/sonarqube/conf
          name: data
          subPath: conf
        - mountPath: /opt/sonarqube/data
          name: data
          subPath: data
        - mountPath: /opt/sonarqube/extensions
          name: data
          subPath: extensions
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: sonarqube-data-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: sonarqube
  namespace: sonarqube
  labels:
    app: sonarqube
spec:
  type: NodePort
  ports:
    - name: sonarqube
      port: 9000
      targetPort: 9000
      nodePort: 30003
      protocol: TCP
  selector:
    app: sonarqube
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: sonarqube-ingress
  namespace: sonarqube
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.frontend.rule.type: PathPrefixStrip  
spec:
  rules:
  - host: sonarqube.k8s.com
    http:
      paths:
      - path: /
        backend:
          serviceName: sonarqube
          servicePort: 9000
EOF
```

```sh
[root@k8s-master01 sonarqube]# cat sonarqube-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sonarqube-data-pvc
  namespace: sonarqube
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

最后完成所有部署

```sh
[root@k8s-master01 sonarqube]# kubectl get pods -A
NAMESPACE         NAME                                       READY   STATUS    RESTARTS   AGE
default           nfs-provisioner-65595c99fc-zrcmw           1/1     Running   1          46m
gitlab            gitlab-8574d86d4f-dbkfk                    1/1     Running   1          29m
gitlab            postgresql-d958c8495-nwqmd                 1/1     Running   1          36m
gitlab            redis-696f86499f-q6c6l                     1/1     Running   1          36m
ingress-traefik   traefik-ingress-controller-2msjp           1/1     Running   1          27m
ingress-traefik   traefik-ingress-controller-qhj8z           1/1     Running   1          27m
ingress-traefik   traefik-ingress-controller-x4k5x           1/1     Running   1          27m
jenkins           jenkins-57d4fd9d47-w96qx                   1/1     Running   0          16m
kube-system       calico-kube-controllers-54cb9b8bb6-wnwnf   1/1     Running   1          46m
kube-system       calico-node-8x5zt                          1/1     Running   1          46m
kube-system       calico-node-99ws8                          1/1     Running   1          46m
kube-system       calico-node-m8fsw                          1/1     Running   1          46m
kube-system       coredns-546565776c-dpdsf                   1/1     Running   1          48m
kube-system       coredns-546565776c-l8zvl                   1/1     Running   1          48m
kube-system       etcd-k8s-master01                          1/1     Running   1          48m
kube-system       etcd-k8s-master02                          1/1     Running   1          47m
kube-system       etcd-k8s-master03                          1/1     Running   1          46m
kube-system       kube-apiserver-k8s-master01                1/1     Running   1          48m
kube-system       kube-apiserver-k8s-master02                1/1     Running   1          48m
kube-system       kube-apiserver-k8s-master03                1/1     Running   2          46m
kube-system       kube-controller-manager-k8s-master01       1/1     Running   2          48m
kube-system       kube-controller-manager-k8s-master02       1/1     Running   1          48m
kube-system       kube-controller-manager-k8s-master03       1/1     Running   1          45m
kube-system       kube-proxy-6s484                           1/1     Running   1          48m
kube-system       kube-proxy-8p79l                           1/1     Running   1          48m
kube-system       kube-proxy-kw6pb                           1/1     Running   1          47m
kube-system       kube-scheduler-k8s-master01                1/1     Running   2          48m
kube-system       kube-scheduler-k8s-master02                1/1     Running   1          48m
kube-system       kube-scheduler-k8s-master03                1/1     Running   1          45m
sonarqube         postgres-sonar-76f887c66b-t2lpx            1/1     Running   0          112s
sonarqube         sonarqube-7554bbf78b-f2hsx                 1/1     Running   0          112s
```

---
layout: post
title: Devops-09-pipline流水线
date: 2022-05-11
tags: Devops
---

### 1. Jenkinsfile 维护流水线脚本

![](/images/posts/Devops/pipeline流水线/1.png)

![2](/images/posts/Devops/pipeline流水线/2.png)

![3](/images/posts/Devops/pipeline流水线/3.png)

![4](/images/posts/Devops/pipeline流水线/4.png)

### 2. 创建 Jenkinsfile

```sh
pipeline {

	// 指定任务在哪个集群节点上运行
	agent any

	// 声明全局变量，方便后面使用
	environment{
		harborUser = 'admin'
		harborPasswd = 'Harbor12345'
		harborAddres = '192.168.20.54'
		harborRepo = 'repo'
	}

		stages {
		stage('拉取git仓库代码') {
			steps {
				checkout([$class: 'GitSCM', branches: [[name: '${tag}']], extensions: [], userRemoteConfigs: [[url: 'http://gitlab.k8s.com/root/my-test.git']]])

			}
		}
		stage('通过maven构建项目') {
			steps {
				sh '/var/jenkins_home/maven/bin/mvn clean package -DskipTests'

			}
		}
		stage('通过SonarQube做代码质量检测') {
			steps {
				sh '/var/jenkins_home/sonar-scanner/bin/sonar-scanner -Dsonar.source=./ -Dsnoar.projectname=${JOB_NAME} -Dsonar.projectKey=${JOB_NAME} -Dsonar.java.binaries=./target/ -Dsonar.login=64b3bd0cfb51c89d94187ae291a32989c3849d44'

			}
		}
		stage('通过Docker制作自定义镜像') {
			steps {
				sh '''mv ./target/*.jar ./docker/
                docker build -t ${JOB_NAME}:latest ./docker'''

			}
		}
		stage('将自定义镜像推送到Harbor') {
			steps {
				sh '''docker login ${harborAddres} -u ${harborUser} -p ${harborPasswd}
                docker tag ${JOB_NAME}:latest ${harborAddres}/${harborRepo}/${JOB_NAME}:latest
                docker push ${harborAddres}/${harborRepo}/${JOB_NAME}:latest'''

			}
		}
		stage('将pipeline-demo.yml传输到k8s-master机器上') {
			steps {
                sshPublisher(publishers: [sshPublisherDesc(configName: 'k8s-server', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: '', execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: '', sourceFiles: 'pipeline-demo.yml')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
			}
		}
		stage('远程执行kubectl apply命令') {
			steps {
                sh 'ssh root@192.168.20.60 kubectl apply -f /data/pipeline/pipeline.yml'
                sh 'ssh root@192.168.20.60 kubectl rollout restart deployment -n jenkins-pipeline pipeline'

			}
		}
	}

	post {
	    success {
	        dingtalk(
	            robot: 'Jenkins_DingDing',
	            type: 'MARKDOWN',
	            title: "success: ${JOB_NAME}",
	            text: ["- 构建成功: ${JOB_NAME}! \n- 版本: latest \n- 持续时间: ${currentBuild.durationString}"]
	        )
	    }
	    failure {
	        dingtalk(
	            robot: 'Jenkins_DingDing',
	            type: 'MARKDOWN',
	            title: "success: ${JOB_NAME}",
	            text: ["- 构建失败: ${JOB_NAME}! \n- 版本: latest \n- 持续时间: ${currentBuild.durationString}"]
	        )
	    }

	}
}
```

![](/images/posts/Devops/pipeline流水线/5.png)

![6](/images/posts/Devops/pipeline流水线/6.png)

![7](/images/posts/Devops/pipeline流水线/7.png)

![8](/images/posts/Devops/pipeline流水线/8.png)

![9](/images/posts/Devops/pipeline流水线/9.png)

![10](/images/posts/Devops/pipeline流水线/10.png)

![11](/images/posts/Devops/pipeline流水线/11.png)

![12](/images/posts/Devops/pipeline流水线/12.png)

![13](/images/posts/Devops/pipeline流水线/13.png)

![14](/images/posts/Devops/pipeline流水线/14.png)

![15](/images/posts/Devops/pipeline流水线/15.png)

### 3. 创建 pipeline-demo.yml 文件

```yml
apiVersion: v1
kind: Namespace
metadata:
  name: pipeline-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pipeline-demo
  namespace: pipeline-demo
  labels:
    k8s-app: pipeline-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: pipeline-demo
  template:
    metadata:
      labels:
        k8s-app: pipeline-demo
    spec:
      containers:
      - name: pipeline-demo
        image: 192.168.20.54/repo/pipeline-demo:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 80
      restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: pipeline-demo
  namespace: pipeline-demo
  labels:
    k8s-app: pipeline-demo
spec:
  type: NodePort
  selector:
    k8s-app: pipeline-demo
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: pipeline-demo
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.frontend.rule.type: PathPrefixStrip  
spec:
  rules:
  - host: pipeline.k8s.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          serviceName: pipeline-demo
          servicePort: 8080
```

![](/images/posts/Devops/pipeline流水线/16.png)

![17](/images/posts/Devops/pipeline流水线/17.png)

需要注意的是，Jenkins要去远程主机执行 ssh 命令，所以我们要事先配置好Jenkins对目标服务器的免密登录

### 4. 配置Jenkins免密登陆

```sh
[root@k8s-master01 ~]# kubectl exec -it -n jenkins jenkins-57d7d74b66-bkq7s bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.
root@jenkins-57d7d74b66-bkq7s:/#
root@jenkins-57d7d74b66-bkq7s:/# cd
root@jenkins-57d7d74b66-bkq7s:~# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
/root/.ssh/id_rsa already exists.
Overwrite (y/n)? y
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:F5t4hccfXoNbHC6sZrrNpwOzf6RKDHm47TGczqHQmc8 root@jenkins-57d7d74b66-bkq7s
The key's randomart image is:
+---[RSA 3072]----+
|               . |
|           o. + .|
|          o ++.=.|
|        o. *.o+o.|
|       +S.=+ .o  |
|      . X==  .   |
|     . = @+ o    |
|      . O.Bo o   |
|       . Eo==    |
+----[SHA256]-----+
# 因为我k8s是个集群，并且有3个master，所以拷贝了3次
root@jenkins-57d7d74b66-bkq7s:~# scp ~/.ssh/id_rsa.pub 192.168.20.51:~/.ssh/
Warning: Permanently added '192.168.20.51' (ECDSA) to the list of known hosts.
root@192.168.20.51's password:
id_rsa.pub                                                                                                                                                                                                                                  100%  583   773.7KB/s   00:00    
root@jenkins-57d7d74b66-bkq7s:~# scp ~/.ssh/id_rsa.pub 192.168.20.52:~/.ssh/
Warning: Permanently added '192.168.20.52' (ECDSA) to the list of known hosts.
root@192.168.20.52's password:
id_rsa.pub                                                                                                                                                                                                                                  100%  583   567.6KB/s   00:00    
root@jenkins-57d7d74b66-bkq7s:~# scp ~/.ssh/id_rsa.pub 192.168.20.53:~/.ssh/
Warning: Permanently added '192.168.20.53' (ECDSA) to the list of known hosts.
root@192.168.20.53's password:
id_rsa.pub                                                                                                                                                                                                                                  100%  583   778.1KB/s   00:00    
root@jenkins-57d7d74b66-bkq7s:~#
# 测试连接没问题，因为我这个地址VIP地址，所以可以连接VIP
[root@k8s-master01 ~]# cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
root@jenkins-57d7d74b66-bkq7s:/# ssh 192.168.20.60
Warning: Permanently added '192.168.20.60' (ECDSA) to the list of known hosts.
Last login: Wed May 11 11:26:26 2022 from 192.168.20.2
```

### 5. 配置钉钉机器人通知

![](/images/posts/Devops/pipeline流水线/18.png)

钉钉创建一个项目组，然后添加机器人

![](/images/posts/Devops/pipeline流水线/19.png)

![](/images/posts/Devops/pipeline流水线/20.png)

![](/images/posts/Devops/pipeline流水线/21.png)

### 6. 添加新的代码标签

![](/images/posts/Devops/pipeline流水线/22.png)

![](/images/posts/Devops/pipeline流水线/22.png)

![](/images/posts/Devops/pipeline流水线/23.png)

### 7. 构建流水线任务

![](/images/posts/Devops/pipeline流水线/24.png)

![](/images/posts/Devops/pipeline流水线/25.png)

![](/images/posts/Devops/pipeline流水线/26.png)



![](/images/posts/Devops/pipeline流水线/27.png)

### 8. 提交代码自动发布

![](/images/posts/Devops/pipeline流水线/28.png)

![29](/images/posts/Devops/pipeline流水线/29.png)

![30](/images/posts/Devops/pipeline流水线/30.png)

![31](/images/posts/Devops/pipeline流水线/31.png)

![32](/images/posts/Devops/pipeline流水线/32.png)

![33](/images/posts/Devops/pipeline流水线/33.png)

![34](/images/posts/Devops/pipeline流水线/34.png)

![35](/images/posts/Devops/pipeline流水线/35.png)

![36](/images/posts/Devops/pipeline流水线/36.png)

![37](/images/posts/Devops/pipeline流水线/37.png)

![38](/images/posts/Devops/pipeline流水线/38.png)

![39](/images/posts/Devops/pipeline流水线/39.png)

![40](/images/posts/Devops/pipeline流水线/40.png)

![41](/images/posts/Devops/pipeline流水线/41.png)

![42](/images/posts/Devops/pipeline流水线/42.png)

![43](/images/posts/Devops/pipeline流水线/43.png)

![44](/images/posts/Devops/pipeline流水线/44.png)

![44](/images/posts/Devops/pipeline流水线/45.png)

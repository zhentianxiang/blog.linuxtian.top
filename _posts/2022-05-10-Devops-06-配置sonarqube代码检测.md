---
layout: post
title: Devops-06-配置sonarqube代码检测
date: 2022-05-10
tags: Devops
---

### 1. Sonar qube 基本配置

![](/images/posts/Devops/Jenkins使用sonarqube代码检测/1.png)

![2](/images/posts/Devops/Jenkins使用sonarqube代码检测/2.png)

![3](/images/posts/Devops/Jenkins使用sonarqube代码检测/3.png)

![4](/images/posts/Devops/Jenkins使用sonarqube代码检测/4.png)

![5](/images/posts/Devops/Jenkins使用sonarqube代码检测/5.png)

### 2. 本地 maven 配置sonarqube

![5](/images/posts/Devops/Jenkins使用sonarqube代码检测/6.png)

![5](/images/posts/Devops/Jenkins使用sonarqube代码检测/7.png)

```xml
  <profile>
      <id>sonar</id>
      <activation>
          <activeByDefault>true</activeByDefault>
      </activation>
      <properties>
          <sonar.login>admin</sonar.login>
          <sonar.password>root123123</sonar.password>
          <sonar.host.url>http://sonarqube.k8s.com/</sonar.host.url>
        </properties>
    </profile>
  </profiles>
```

最后配自行百度置好path环境变量

![](/images/posts/Devops/Jenkins使用sonarqube代码检测/8.png)

![9](/images/posts/Devops/Jenkins使用sonarqube代码检测/9.png)

![10](/images/posts/Devops/Jenkins使用sonarqube代码检测/10.png)

![11](/images/posts/Devops/Jenkins使用sonarqube代码检测/11.png)

### 3. jenkins 使用 sonarqube

由于之前已经配置好了相关目录以及文件，接下来我们直接测试即可

![](/images/posts/Devops/Jenkins使用sonarqube代码检测/12.png)

![](/images/posts/Devops/Jenkins使用sonarqube代码检测/13.png)

```sh
# 进入到容器内
root@jenkins-57d7d74b66-bkq7s:/var/jenkins_home/workspace/my-test# /var/jenkins_home/sonar-scanner/bin/sonar-scanner -Dsonar.source=./ -Dsonar.projectname=jenkins-test -Dsonar.login=64b3bd0cfb51c89d94187ae291a32989c3849d44 -Dsonar.projectKey=jenkins-test -Dsonar.java.binaries=./target
```

![](/images/posts/Devops/Jenkins使用sonarqube代码检测/14.png)

![](/images/posts/Devops/Jenkins使用sonarqube代码检测/15.png)

```sh
# 跑完任务后删除生成的隐藏目录，不然后面jenkins跑任务会出报错
root@jenkins-57d7d74b66-bkq7s:/var/jenkins_home/workspace# ls -a
.  ..  .scannerwork  my-test
root@jenkins-57d7d74b66-bkq7s:/var/jenkins_home/workspace# rm -rf .scannerwork/
```

### 4. jenkins 整合 sonarqube

![](/images/posts/Devops/Jenkins使用sonarqube代码检测/16.png)

![17](/images/posts/Devops/Jenkins使用sonarqube代码检测/17.png)

![18](/images/posts/Devops/Jenkins使用sonarqube代码检测/18.png)

![19](/images/posts/Devops/Jenkins使用sonarqube代码检测/19.png)

![20](/images/posts/Devops/Jenkins使用sonarqube代码检测/20.png)

![21](/images/posts/Devops/Jenkins使用sonarqube代码检测/21.png)

![22](/images/posts/Devops/Jenkins使用sonarqube代码检测/22.png)

![23](/images/posts/Devops/Jenkins使用sonarqube代码检测/23.png)

![24](/images/posts/Devops/Jenkins使用sonarqube代码检测/24.png)

![25](/images/posts/Devops/Jenkins使用sonarqube代码检测/25.png)

![26](/images/posts/Devops/Jenkins使用sonarqube代码检测/26.png)

![27](/images/posts/Devops/Jenkins使用sonarqube代码检测/27.png)

![28](/images/posts/Devops/Jenkins使用sonarqube代码检测/28.png)

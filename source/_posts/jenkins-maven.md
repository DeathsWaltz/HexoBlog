---
title: maven安装jar到本地仓库后jenkins编译项目引用不到问题
date: 2019-10-19 13:30:31
tags: [jenkins,maven]
categories: [学习,jenkins,error]
---



#### 问题

##### maven安装jar到本地仓库后jenkins编译项目报错

##### 使用将jar安装到本地仓库的方法.

安装jar到本地仓库命令

```shell
 mvn install:install-file -DgroupId=com.taobao.api -DartifactId=taobao-sdk  -Dversion=1.0 -Dpackaging=jar -Dfile=taobao-sdk-java-auto_1565853390086-20190917.jar
```

然后在`pom.xml`文件中引入

```xml
        <dependency>
            <groupId>com.taobao.api</groupId>
            <artifactId>taobao-sdk</artifactId>
            <version>1.0</version>
        </dependency>
```

安装后还是无法引用到jar包.

#### 原因

因maven本地仓库的目录是在运行用户目录的`.m2/reopsitory`下,`settings.xml`文件中有说明.

```xml
 <!-- localRepository
   | The path to the local repository maven will use to store artifacts.
   |
   | Default: ${user.home}/.m2/repository
  <localRepository>/path/to/local/repo</localRepository>
  -->
```

执行jar包安装的命令是在root用户下运行的,安装后jar包被安装到了`/root/.m2/repository`目录下.

jenkins是使用rpm文件进行的安装,安装后jenkins运行在自建的用户下.

所以在jenkins的用户下引用不到安装后的jar包.

#### 解决

修改maven本地仓库的配置`settings.xml`

在`<settings>`标签内新增

```xml
<!-- jenkins 用户的目录 -->
<localRepository>/var/lib/jenkins/.m2/repository</localRepository>
```

再重新执行jar包安装命令后解决.
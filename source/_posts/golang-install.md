---
title: Golang安装
date: 2020-02-19 11:25:24
tags:  [golang,go,goproxy]
categories: [学习,golang]
---

基本信息

centos 7

下载安装包

地址：[https://studygolang.com/dl](https://studygolang.com/dl)

下载对应最新安装包

```shell
#version为对应的版本号 plantfrom为平台
wget https://studygolang.com/dl/golang/go{version}.{plantfrom}-amd64.tar.gz
```



解压安装

解包放在/usr/local/目录下，会生成go文件夹。

```shell
tar zxvf go{version}.{plantfrom}-amd64.tar.gz -C /usr/local
```

配置Go环境变量

```shell
# 编辑profile文件
# vi /etc/profile
# 在文件末尾添加如下内容
#go setting
export GOROOT=/usr/local/go
export GOPATH=/usr/local/gopath
export PATH=$PATH:$GOROOT/bin
```

执行source /etc/profile指令，配置文件的环境变量立刻生效

```shell
source /etc/profile
```

验证生效

```shell
#查看go版本信息
go version
#查看go环境变量配置
go env
```



配置代理

```shell
# 编辑profile文件
# vi /etc/profile
# 在文件末尾添加如下内容
# 启用 Go Modules 功能
export GO111MODULE=on
# 配置 GOPROXY 环境变量
export GOPROXY=https://goproxy.io
```

执行以下命令生效：

```
source /etc/profile
```



参考地址：

[https://yq.aliyun.com/articles/645569](https://yq.aliyun.com/articles/645569)

[https://goproxy.io/zh/](https://goproxy.io/zh/)
---
title: docker以及docker-compose安装
date: 2020-03-30 08:31:06
tags: [docker,docker-compose]
---

#### 基本信息

#### 安装Docker

##### 使用官方脚本自动安装

```shell
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```

##### 手动安装帮助（阿里云ecs可通过内网安装）

centos7

```shell
# step 1: 安装必要的一些系统工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# Step 2: 添加软件源信息
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# Step 3: 更新并安装 Docker-CE
sudo yum makecache fast
sudo yum -y install docker-ce
# Step 4: 开启Docker服务
sudo service docker start

注意：其他注意事项在下面的注释中
# 官方软件源默认启用了最新的软件，您可以通过编辑软件源的方式获取各个版本的软件包。例如官方并没有将测试版本的软件源置为可用，你可以通过以下方式开启。同理可以开启各种测试版本等。
# vim /etc/yum.repos.d/docker-ce.repo
#   将 [docker-ce-test] 下方的 enabled=0 修改为 enabled=1
#
# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# yum list docker-ce.x86_64 --showduplicates | sort -r
#   Loading mirror speeds from cached hostfile
#   Loaded plugins: branch, fastestmirror, langpacks
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            docker-ce-stable
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            @docker-ce-stable
#   docker-ce.x86_64            17.03.0.ce-1.el7.centos            docker-ce-stable
#   Available Packages
# Step2 : 安装指定版本的Docker-CE: (VERSION 例如上面的 17.03.0.ce.1-1.el7.centos)
# sudo yum -y install docker-ce-[VERSION]
# 注意：在某些版本之后，docker-ce安装出现了其他依赖包，如果安装失败的话请关注错误信息。例如 docker-ce 17.03 之后，需要先安装 docker-ce-selinux。
# yum list docker-ce-selinux- --showduplicates | sort -r
# sudo yum -y install docker-ce-selinux-[VERSION]

# 通过经典网络、VPC网络内网安装时，用以下命令替换Step 2中的命令
# 经典网络：
# sudo yum-config-manager --add-repo http://mirrors.aliyuncs.com/docker-ce/linux/centos/docker-ce.repo
# VPC网络：
# sudo yum-config-manager --add-repo http://mirrors.could.aliyuncs.com/docker-ce/linux/centos/docker-ce.repo
```



##### 镜像加速

新建或修改`/etc/docker/daemon.json`，加入：

```json
{
    "registry-mirrors": [
        "https://dockerhub.azk8s.cn",
        "https://reg-mirror.qiniu.com"
    ]
}
```

一定要确保格式没有问题，否则 docker 无法启动，修改完成后执行以下命令：

```shell
#重新加载配置
sudo systemctl daemon-reload
#启动Docker
sudo systemctl restart docker
```

##### 关闭 selinux

1、临时关闭（不用重启机器）

```shell
setenforce 0 #设置SELinux 成为permissive模式
#setenforce 1 设置SELinux 成为enforcing模式
```

2、修改配置文件(重启机器)

修改`/etc/selinux/config` 文件

将`SELINUX=enforcing`改为`SELINUX=disabled`

重启机器即可

#### 安装Docker-compose

```shell
sudo curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

使用github进行安装可能会下载失败，可先下载`docker-compose`安装文件后上传到服务进行安装。上传到服务器后移动到`/usr/local/bin`目录，使用`chmod +x /usr/local/bin/docker-compose`命令修改`docker-compose`为可执行权限。



参考地址：

[https://yq.aliyun.com/articles/110806](https://yq.aliyun.com/articles/110806)

[https://docs.docker.com/compose/install/](https://docs.docker.com/compose/install/)
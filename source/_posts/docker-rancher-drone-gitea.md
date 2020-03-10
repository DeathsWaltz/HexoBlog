---
title: 使用drone实现自动化部署安装配置以及踩坑
date: 2020-02-18 14:55:16
tags: [docker,rancher,drone,gitea]
---

#### 基本信息

centos 7 阿里云

```shell
# 安装 kubelet kubeadm kubectl
sudo yum install -y kubelet kubeadm kubectl
```



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



#### 安装Docker-compose

```shell
sudo pip install docker-compose
```

#### 安装Rancher1.x

```shell
sudo docker run -d -v /var/lib/mysql:/var/lib/mysql --restart=unless-stopped -p 8236:8080 rancher/server
```

运行成功后访问`ip:8236`

![rancher](https://azrael-1252339850.cos.ap-chengdu.myqcloud.com/20200218/index.jpg)

进行配置,先设置`系统管理>访问控制`

![rancher](https://azrael-1252339850.cos.ap-chengdu.myqcloud.com/control.jpg)

`基础框架>添加主机`根据提示添加主机

![rancher](https://azrael-1252339850.cos.ap-chengdu.myqcloud.com/20200218/addhost.jpg)

`API>密钥`生成账号API Keys

![rancher](https://azrael-1252339850.cos.ap-chengdu.myqcloud.com/20200218/addkeys.jpg)

`基础架构>镜像库`添加私有镜像库

![rancher](https://azrael-1252339850.cos.ap-chengdu.myqcloud.com/20200218/addrepo.jpg)

#### 安装Gitea

使用`docker-compose`运行。

```shell
sudo docker-compose -f gitea-docker-compose.yml  up -d
```

`gitea-docker-compose.yml`文件内容：

```yml
version: '2'
services:
  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    ports:
    - "10022:22"
    - "10080:3000"
    volumes:
    - /var/lib/gitea:/data
    restart: always

```

访问`ip:10080`进入安装界面进行配置。

#### 安装Drone

在Gitea新建OAuth Application,参考官方文档。

![rancher](https://azrael-1252339850.cos.ap-chengdu.myqcloud.com/20200218/adddrone.jpg)

将生成的客户端ID和客户端密钥保存。

`docker-compose.yml`文件内容:

```yml
version: '2'

services:
  drone-server:
    image: drone/drone:1.2.1
    container_name: drone-server
    networks:
      - dronenet        # 让drone-server和drone-agent处于一个网络中，方便进行RPC通信
    ports:
      - '8099:80'      # Web管理面板的入口 PROTO=http  时使用该端口
      - '8999:443'     # Web管理面板的入口 PROTO=https 时使用该端口
      - '9000:9000'    # RPC服务端口
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock   # docker.sock [1]
      - /var/lib/drone/:/var/lib/drone             # drone数据存放路径
    environment:
      - DRONE_AGENTS_ENABLED=false                   # 使用Runner
      #- DRONE_GITLAB_SERVER=${DRONE_GITLAB_SERVER}
      #- DRONE_GITLAB_CLIENT_ID=${DRONE_GITLAB_CLIENT_ID}
      #- DRONE_GITLAB_CLIENT_SECRET=${DRONE_GITLAB_CLIENT_SECRET}
      #- DRONE_GITHUB_SERVER=${DRONE_GITHUB_SERVER}
     # - DRONE_GITHUB_CLIENT_ID=${DRONE_GITHUB_CLIENT_ID}
     # - DRONE_GITHUB_CLIENT_SECRET=${DRONE_GITHUB_CLIENT_SECRET}
      - DRONE_GITEA_SERVER=${DRONE_GITEA_SERVER}                   # github的地址
      - DRONE_GITEA_CLIENT_ID=${DRONE_GITEA_CLIENT_ID}          # gitea获得的ClientID
      - DRONE_GITEA_CLIENT_SECRET=${DRONE_GITEA_CLIENT_SECRET}  # gitea获得的ClientSecret
      - DRONE_RPC_SECRET=${DRONE_RPC_SECRET}                      # RPC秘钥     [2]
      - DRONE_SERVER_HOST=${DRONE_SERVER_HOST}                    # RPC域名(在一个实例上可以不用)
      - DRONE_SERVER_PROTO=${DRONE_SERVER_PROTO}                  # git webhook使用的协议(我建议http)
      - DRONE_OPEN=true                                           # 开发drone
      - DRONE_DATABASE_DATASOURCE=/var/lib/drone/drone.sqlite     # 数据库文件
      - DRONE_DATABASE_DRIVER=sqlite3                             # 数据库驱动，我这里选的sqlite
      #- DRONE_DATABASE_DRIVER=mysql
      - DRONE_DEBUG=true                                          # 调试相关，部署的时候建议先打开
      - DRONE_LOGS_DEBUG=true                                     # 调试相关，部署的时候建议先打开
      - DRONE_LOGS_TRACE=true                                     # 调试相关，部署的时候建议先打开
      - DRONE_USER_CREATE=username:***,admin:true           # 初始管理员用户 gitea用户名
      - TZ=Asia/Shanghai                                          # 时区
    restart: always
  drone-agent:
    image: drone/agent:1.2.1
    container_name: drone-agent
    networks:
      - dronenet     # 让drone-server和drone-agent处于一个网络中，方便进行RPC通信
    depends_on:
      - drone-server
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock # docker.sock [1]
    environment:
      - DRONE_RPC_SERVER=http://drone-server  # RPC服务地址
      - DRONE_RPC_SECRET=${DRONE_RPC_SECRET}  # RPC秘钥  
      - DRONE_RPC_PROTO=${DRONE_RPC_PROTO}    # RPC协议(http || https)
      - DRONE_RUNNER_CAPACITY=2               # 最大并发执行的 pipeline 数
      - DRONE_DEBUG=true                      # 调试相关，部署的时候建议先打开
      - DRONE_LOGS_TRACE=true                 # 调试相关，部署的时候建议先打开
      - TZ=Asia/Shanghai
    restart: always
networks:
  dronenet:                                     # 让drone-server和drone-agent处于一个网络中，方便进行RPC通信

```

**[1]** 因为插件本身也是一个容器，要在容器中(docker-server、drone-runnere)中运行容器。将docker.sock挂载到容器中，可以让容器通过docker unix socket API得到管理容器的能力。

**[2]** `openssl rand -hex 16` 这个命令随机生成秘钥

.env

```properties
#DRONE_GITHUB_CLIENT_ID=****
#DRONE_GITHUB_CLIENT_SECRET=****
DRONE_GITEA_SERVER=http://git****com
DRONE_GITEA_CLIENT_ID=****
DRONE_GITEA_CLIENT_SECRET=****
#DRONE_GITLAB_SERVER=http://git.****.com
#DRONE_GITLAB_CLIENT_ID=****
#DRONE_GITLAB_CLIENT_SECRET=****
DRONE_RPC_SECRET=*****
DRONE_SERVER_HOST=drone.****.com
DRONE_SERVER_PROTO=https
DRONE_RPC_SERVER=****:9000
DRONE_RPC_PROTO=http

```

将`docker-compose.yml`和`.env`放在同一目录，然后运行以下命令：

```shell
sudo docker-compose up -d
```





参考地址：

[https://yq.aliyun.com/articles/110806](https://yq.aliyun.com/articles/110806)

https://juejin.im/post/5d97489ee51d457824771d47

https://docs.drone.io/server/provider/gitea/
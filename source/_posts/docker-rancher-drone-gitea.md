---
title: 使用drone实现自动化部署安装配置以及踩坑
date: 2020-02-18 14:55:16
tags: [docker,rancher,drone,gitea]
---

#### 基本信息

centos 7 阿里云

#### 安装docker

参考[docker以及docker-compose安装](/2020/03/30/docker-install-docker-compose-install/)

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

https://juejin.im/post/5d97489ee51d457824771d47

https://docs.drone.io/server/provider/gitea/
---
title: docker修改镜像文件存储位置
date: 2020-04-13 16:32:01
tags: [docker,storage,'存储位置']
categories: [学习,docker]
---

#### 修改docker文件存储位置

#### 问题

因为服务器磁盘存储空间剩余不多，然后挂载了新的存储，需要迁移docker镜像文件。

#### 解决方法

Docker默认的存放位置为：`/var/lib/docker`

以下命令可查看具体位置：

```shell
 docker info | grep "Docker Root Dir"
```

先将docker服务停止:

```shell
service docker stop
```

将数据转移到新挂载的磁盘:

```shell
mv /var/lib/docker /data/docker
```

移动时出现`device or resource busy`问题，原因是因为`docker`运行的某些容器挂载的目录没有被卸载。

使用以下命令查找为卸载的目录(`/var/lib/docker`为`docker`目录):

```shell
cat /proc/mounts | grep "/var/lib/docker" | awk '{print $2}'
```

结果为：

```shell
/var/lib/docker/overlay2/5d89f720aaca052486928e57fb8a86568e8991b0ca327518f4704ba61718beb9/merged
/var/lib/docker/containers/73a873a188f3c7402e45805e5e47ff99346b016a8e9977bd609a4f242a499a25/mounts/shm
/var/lib/docker/overlay2/a27aaef652a6d9edeaa9d8672ae0b79feef3cf58e28ac34c6f56d6d777af8c02/merged
/var/lib/docker/containers/89483898e50f02171486bfa86708e530aa7bd257f717c8a336fba4a79e2e0b2c/mounts/shm
/var/lib/docker/overlay2/94bc54b30dcc128d258a8ec94293071fddd053bc40ebf4692d2fbb719fb596e4/merged

```

然后使用以下命令卸载对应目录：

```shell
umount  var/lib/docker/containers/73a873a188f3c7402e45805e5e47ff99346b016a8e9977bd609a4f242a499a25/mounts/shm
```

为运行方便，编写为`shell`脚本

```shell
#!/bin/bash
# 停止docker服务
service  docker stop
# 查询并卸载处于挂载中的目录
umount $(cat /proc/mounts | grep "/var/lib/docker" | awk '{print $2}')
# 拷贝docker文件到指定目录
cp -rf  /var/lib/docker /data
# 重命名docker原文件，备份防止出错
mv /var/lib/docker /var/lib/dockerbak
# 通过软连接指向原地址
ln -s /data/docker /var/lib/docker
# 重启docker服务
service docker restart
#可不删除，防止出现问题
#rm -rf /var/lib/dockerbak
```



参考地址：

[https://blog.51cto.com/forangela/1949947](https://blog.51cto.com/forangela/1949947)

[https://blog.csdn.net/gulijiang2008/article/details/80591843](https://blog.csdn.net/gulijiang2008/article/details/80591843)
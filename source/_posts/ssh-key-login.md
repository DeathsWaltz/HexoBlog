---
title: ssh免密登录
date: 2020-04-13 15:59:54
tags: [ssh]
categories: [学习,linux]
---

#### 添加hosts

在`/etc/hosts`文件中添加对应主机地址已经名称:

```shell
10.5.16.1 node1 
10.5.16.2 node2
10.5.16.3 node3 
10.5.16.4 node4 
10.5.16.5 node5 
10.5.16.6 node6 
```



#### 生成key

在node1服务器执行:

```shell
ssh-keygen -t rsa
```

提示输入一些信息，默认即可:

```shell
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Created directory '/root/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:xOkgW2P4IcB4epLE2H8UZywgtm/xoLUHaroqVJuoeXY root@sugon51601
The key's randomart image is:
+---[RSA 2048]----+
|o++ ...oo        |
|o+++ .o+..       |
|.+..O.*.+        |
|+ .*.&.*         |
| o* B.+ S        |
| = + .           |
|+.               |
|+.o E            |
|=o .             |
+----[SHA256]-----+

```



然后使用以下命令将公钥拷贝到其他服务器:

```shell
ssh-copy-id node2
```

提示输入`node2`服务器的密码:

```shell
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
The authenticity of host 'node2 (10.5.16.2)' can't be established.
ECDSA key fingerprint is SHA256:b3KWX6G6H7ANx8tNzhobL9su6YnmT55aGSpg3xlO4xY.
ECDSA key fingerprint is MD5:25:03:35:34:e0:e7:70:2b:0b:65:87:71:c6:50:36:60.
Are you sure you want to continue connecting (yes/no)? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@node2's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'node2'"
and check to make sure that only the key(s) you wanted were added
```

#### 问题



如生成密钥时提示权限不足可修改`/etc/ssh/ssh_config`文件的配置(centos系统)：

```shell
PermitRootLogin yes
PubkeyAuthentication yes
PasswordAuthentication  yes
```



参考地址：

[https://yq.aliyun.com/articles/701548](https://yq.aliyun.com/articles/701548)
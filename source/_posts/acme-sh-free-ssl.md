---
title: acme.sh免费https证书
date: 2020-09-12 13:10:43
tags: ['.acme.sh','https','证书','nginx']
categories: [学习,nginx,https]
---

#### 安装acme.sh

```shell
curl https://get.acme.sh | sh
```

或者

```shell
wget -O - https://get.acme.sh | sh
```

最好使用第二个命令，原因：**wget** 命令用来从指定的**URL**下载文件。**wget**非常稳定，它在带宽很窄的情况下和不稳定网络中有很强的适应性，如果是由于网络的原因下载失败，**wget**会不断的尝试，直到整个文件下载完毕。如果是服务器打断下载过程，它会再次联到服务器上从停止的地方继续下载。这对从那些限定了链接时间的服务器上下载大文件非常有用。

添加环境变量：

`/etc/profile`中添加`alias acme.sh=~/.acme.sh/acme.sh`，保存后执行`source /etc/profile`加载配置。



#### 生成证书

**http**

`-d`参数后为需要生成https证书的域名，`--webroot`参数为网站跟目录， 然后自动完成验证.

```shell
acme.sh  --issue  -d mydomain.com -d www.mydomain.com  --webroot  /home/wwwroot/mydomain.com/
```

 **apache**服务器

```shell
acme.sh --issue  -d mydomain.com   --apache
```

**nginx**服务器

```shell
acme.sh --issue  -d mydomain.com   --nginx
```

无**web**服务器,根据域名生成

```shell
acme.sh  --issue -d mydomain.com   --standalone
```

**dns**

无需服务器

```shell
acme.sh  --issue  --dns   -d mydomain.com
```

然后, **acme.sh** 会生成相应的解析记录显示出来, 你只需要在你的域名管理面板中添加这条 txt 记录即可.

等待解析完成之后, 重新生成证书:

```shell
acme.sh  --renew   -d mydomain.com
```

**dns** 方式的真正强大之处在于可以使用域名解析商提供的**api** 自动添加 txt 记录完成验证.

**acme.sh** 目前支持 **cloudflare**, **dnspod**, **cloudxns**, **godaddy** 以及 **ovh** 等数十种解析商的自动集成.

以 **dnspod** 为例, 你需要先登录到 **dnspod** 账号, 生成你的 **api id** 和 **api key**, 都是免费的. 然后:

```shell
export DP_Id="1234"

export DP_Key="sADDsdasdgdsf"

acme.sh   --issue   --dns dns_dp   -d aa.com  -d www.aa.com
```

证书就会自动生成了. 这里给出的 **api id **和 **api key** 会被自动记录下来, 将来你在使用**dnspod api** 的时候, 就不需要再次指定了. 直接生成就好了:

```shell
acme.sh  --issue   -d  mydomain2.com   --dns  dns_dp
```

更多：[https://github.com/acmesh-official/acme.sh/wiki/dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi)

#### copy/安装证书

默认生成的证书都放在安装目录下: `~/.acme.sh/`,请不要直接使用此目录下的文件, 例如: 不要直接让 nginx/apache 的配置文件使用这下面的文件. 这里面的文件都是内部使用, 而且目录结构可能会变化.

正确的使用方法是使用 `--install-cert` 命令,并指定目标位置, 然后证书文件会被copy到相应的位置, 例如:

**Apache example**:

```
acme.sh --install-cert -d mydomain.com \
--cert-file      /path/to/certfile/in/apache/cert.pem  \
--key-file       /path/to/keyfile/in/apache/key.pem  \
--fullchain-file /path/to/fullchain/certfile/apache/fullchain.pem \
--reloadcmd     "service apache2 force-reload"
```

**Nginx example**:

```
acme.sh --install-cert -d example.com \
--key-file       /path/to/keyfile/in/nginx/key.pem  \
--fullchain-file /path/to/fullchain/nginx/cert.pem \
--reloadcmd     "service nginx force-reload"
```

(一个小提醒, 这里用的是 `service nginx force-reload`, 不是 `service nginx reload`, 据测试, `reload` 并不会重新加载证书, 所以用的 `force-reload`)

**Nginx** 的配置 `ssl_certificate` 使用 `/etc/nginx/ssl/fullchain.cer` ，而非 `/etc/nginx/ssl/<domain>.cer` ，否则 [SSL Labs](https://www.ssllabs.com/ssltest/) 的测试会报 `Chain issues Incomplete` 错误。

`--install-cert`命令可以携带很多参数, 来指定目标文件. 并且可以指定 reloadcmd, 当证书更新以后, reloadcmd会被自动调用,让服务器生效.

**Nginx**配置

```properties
server {
        listen 443 ssl;
        server_name www.mydomain.com;
        
        ssl_certificate /path/to/keyfile/in/nginx/cert.pem;
        ssl_certificate_key /path/to/fullchain/nginx/key.pem;
        ssl_session_timeout  5m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;     #指定SSL服务器端支持的协议版本
        ssl_ciphers  HIGH:!aNULL:!MD5;
        #ssl_ciphers  ALL：!ADH：!EXPORT56：RC4+RSA：+HIGH：+MEDIUM：+LOW：+SSLv2：+EXP;    #指定加密算法
        ssl_prefer_server_ciphers   on;    #在使用SSLv3和TLS协议时指定服务器的加密算法要优先于客户端的加密算法
}
```

#### 续期证书

目前证书在 60 天以后会自动更新, 你无需任何操作. 今后有可能会缩短这个时间, 不过都是自动的, 你不用关心.

#### 更新acme.sh

目前由于 **acme** 协议和 **letsencrypt CA** 都在频繁的更新, 因此 **acme.sh** 也经常更新以保持同步.

升级 acme.sh 到最新版 :

```
acme.sh --upgrade
```

如果你不想手动升级, 可以开启自动升级:

```
acme.sh  --upgrade  --auto-upgrade
```

之后, **acme.sh** 就会自动保持更新了.

你也可以随时关闭自动更新:

```
acme.sh --upgrade  --auto-upgrade  0
```

#### 错误

如果出错, 请添加 **debug log**：

```
acme.sh  --issue  .....  --debug 
```

或者：

```
acme.sh  --issue  .....  --debug  2
```

请参考： https://github.com/Neilpang/acme.sh/wiki/How-to-debug-acme.sh

更高级的用法请参看其他 **wiki **页面.

https://github.com/Neilpang/acme.sh/wiki



参考文档:

[https://github.com/acmesh-official/acme.sh](https://github.com/acmesh-official/acme.sh)

[https://github.com/acmesh-official/acme.sh/wiki/说明](https://github.com/acmesh-official/acme.sh/wiki/说明)

[https://segmentfault.com/a/1190000022263697](https://segmentfault.com/a/1190000022263697)

[https://github.com/acmesh-official/acme.sh/wiki/dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi)


---
title: Nacos+Seata+Spring boot+Mybatis plus多数据源事务控制
date: 2019-12-12 09:42:16
tags: [seata,spring boot,mybatis plus,多数据源,事务,nacos]
categories: [学习,java,事务]
---

### 问题

使用Mybatis plus多数据源无法控制事务.

### 解决

使用Nacos为配置中心，Seata分布式事务管理.

#### Nacos配置

单机模式下运行Nacos

> Linux/Unix/Mac
>
> Standalone means it is non-cluster Mode. * sh startup.sh -m standalone

> Windows
>
> cmd startup.cmd 或者双击 startup.cmd 文件

 使用mysql

1.安装数据库，版本要求：5.6.5+

2.初始化mysql数据库，数据库初始化文件：nacos-mysql.sql

3.修改conf/application.properties文件，增加支持mysql数据源配置（目前只支持mysql），添加mysql数据源的url、用户名和密码。

`conf\application.properties`

```properties
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://****:3306/nacos_devtest?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true
db.user=****
db.password=****
```

#### Nacos项目中的配置

`pom.xml`

```xml
        <dependency>
            <groupId>com.alibaba.boot</groupId>
            <artifactId>nacos-config-spring-boot-starter</artifactId>
            <version>${nacos-config-spring-boot.version}</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba.boot</groupId>
            <artifactId>nacos-discovery-spring-boot-starter</artifactId>
            <version>${nacos-config-spring-boot.version}</version>
        </dependency>
```



`application.yml`

```yaml
nacos:
  discovery:
    server-addr: 127.0.0.1:8848 #根据实际情况配置
  config:
    server-addr: 127.0.0.1:8848
```



#### Seata配置

建表

全局事务会话信息由3块内容构成，全局事务-->分支事务-->全局锁，对应表global_table、branch_table、lock_table，

mysql建表脚本存放于module seata-server-->conf-->db_store.sql

修改store.mode

打开seata-server-->conf-->file.conf，修改store.mode="db";也可以在启动时加命令参数-m db指定。

```properties
## transaction log store
store {
  ## store mode: file、db
  mode = "db" #修改为db

  ## file store
  file {
    dir = "sessionStore"
    # branch session size , if exceeded first try compress lockkey, still exceeded throws exceptions
    max-branch-session-size = 16384
    # globe session size , if exceeded throws exceptions
    max-global-session-size = 512
    # file buffer size , if exceeded allocate new buffer
    file-write-buffer-cache-size = 16384
    # when recover batch read size
    session.reload.read_size = 100
    # async, sync
    flush-disk-mode = async
  }

  ## database store
  db {
    ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp) etc.
    datasource = "dbcp"
    ## mysql/oracle/h2/oceanbase etc.
    db-type = "mysql"   #修改为mysql
    driver-class-name = "com.mysql.jdbc.Driver"
    url = "jdbc:mysql://******:3306/seata" #配置数据库地址
    user = "****"							#用户名
    password = "******"						#密码
    min-conn = 1
    max-conn = 3
    global.table = "global_table"
    branch.table = "branch_table"
    lock-table = "lock_table"
    query-limit = 100
  }
}
```



提交配置到nacos

运行`conf/nacos-config.sh`将`nacos-config.txt`的内容发送到nacos中

```properties
transport.type=TCP
transport.server=NIO
transport.heartbeat=true
transport.thread-factory.boss-thread-prefix=NettyBoss
transport.thread-factory.worker-thread-prefix=NettyServerNIOWorker
transport.thread-factory.server-executor-thread-prefix=NettyServerBizHandler
transport.thread-factory.share-boss-worker=false
transport.thread-factory.client-selector-thread-prefix=NettyClientSelector
transport.thread-factory.client-selector-thread-size=1
transport.thread-factory.client-worker-thread-prefix=NettyClientWorkerThread
transport.thread-factory.boss-thread-size=1
transport.thread-factory.worker-thread-size=8
transport.shutdown.wait=3
service.vgroup_mapping.my_test_tx_groyp=default  #分组
service.enableDegrade=false
service.disable=false
service.max.commit.retry.timeout=-1
service.max.rollback.retry.timeout=-1
client.async.commit.buffer.limit=10000
client.lock.retry.internal=10
client.lock.retry.times=30
store.mode=db
store.file.dir=file_store/data
store.file.max-branch-session-size=16384
store.file.max-global-session-size=512
store.file.file-write-buffer-cache-size=16384
store.file.flush-disk-mode=async
store.file.session.reload.read_size=100
store.db.driver-class-name=com.mysql.jdbc.Driver
store.db.datasource=dbcp
store.db.db-type=mysql
store.db.url=jdbc:mysql://*********:3306/seata?useUnicode=true   #数据库地址
store.db.user=********         #用户名
store.db.password=********     #密码
store.db.min-conn=1
store.db.max-conn=3
store.db.global.table=global_table
store.db.branch.table=branch_table
store.db.query-limit=100
store.db.lock-table=lock_table
recovery.committing-retry-period=1000
recovery.asyn-committing-retry-period=1000
recovery.rollbacking-retry-period=1000
recovery.timeout-retry-period=1000
transaction.undo.data.validation=true
transaction.undo.log.serialization=jackson
transaction.undo.log.save.days=7
transaction.undo.log.delete.period=86400000
transaction.undo.log.table=undo_log
transport.serialization=seata
transport.compressor=none
metrics.enabled=false
metrics.registry-type=compact
metrics.exporter-list=prometheus
metrics.exporter-prometheus-port=9898
```



启动

- 源码启动: 执行Server.java的main方法
- 命令启动: [seata-server.sh](http://seata-server.sh/) -h 127.0.0.1 -p 8091 -m db -n 1 -DSEATA_ENV=test

```sh
    -h: 注册到注册中心的ip
    -p: Server rpc 监听端口
    -m: 全局事务会话信息存储模式，file、db，优先读取启动参数
    -n: Server node，多个Server时，需区分各自节点，用于生成不同的transactionId范围，以免冲突
    SEATA_ENV: 多环境配置参考 https://github.com/seata/seata/wiki/Multi-configuration-Isolation
```



#### Seata项目中的配置

`pom.xml`

```xml
		<dependency>
            <groupId>io.seata</groupId>
            <artifactId>seata-spring</artifactId>
            <version>${seata.version}</version>
        </dependency>
        <dependency>
            <groupId>io.seata</groupId>
            <artifactId>seata-all</artifactId>
            <version>${seata.version}</version>
        </dependency>
        <!-- seata 依赖jar包  -->
		<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>${druid.version}</version>
        </dependency>
        <dependency>
            <groupId>com.google.protobuf</groupId>
            <artifactId>protobuf-java</artifactId>
            <version>${protobuf.version}</version>
        </dependency>
        <dependency>
            <groupId>io.netty</groupId>
            <artifactId>netty-all</artifactId>
            <version>${netty4.version}</version>
        </dependency>
         <dependencyManagement>
             <dependencies>
                 <dependency>
                     <groupId>com.alibaba.cloud</groupId>
                     <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                     <version>2.1.0.RELEASE</version>
                     <scope>import</scope>
                     <type>pom</type>
                 </dependency>
             </dependencies>
           </dependencyManagement>
```



`resource/registry.conf`对应nacos的IP端口

```
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos"
  nacos {
    serverAddr = "localhost"
    namespace = "public"
    cluster = "default"
  }
}
config {
  # file、nacos 、apollo、zk
  type = "nacos"
  nacos {
    serverAddr = "localhost"
    namespace = "public"
    cluster = "default"
  }
}
```

`application.yml`

```yml
  spring:
      cloud:
        alibaba:
          seata:
            tx-service-group: my_test_tx_group
```



##### 参考地址：

[https://nacos.io/zh-cn/docs/deployment.html](https://nacos.io/zh-cn/docs/deployment.html)

[https://seata.io/zh-cn/](https://seata.io/zh-cn/)

[https://seata.io/zh-cn/docs/user/configurations.html](https://seata.io/zh-cn/docs/user/configurations.html)

[https://github.com/seata/seata-samples/tree/master/springcloud-nacos-seata](https://github.com/seata/seata-samples/tree/master/springcloud-nacos-seata)

[https://github.com/seata/seata-samples/tree/master/multiple-datasource-mybatis-plus](https://github.com/seata/seata-samples/tree/master/multiple-datasource-mybatis-plus)

[http://seata.io/zh-cn/blog/seata-nacos-analysis.html](http://seata.io/zh-cn/blog/seata-nacos-analysis.html)

[http://seata.io/zh-cn/docs/user/configurations.html](http://seata.io/zh-cn/docs/user/configurations.html)
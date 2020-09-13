---

title: 使用canal进行mysql数据实时同步
date: 2020-03-07 14:00:42
tags: [canal,mysql,canal-server,canal-adapter]
categories: [学习,data]
---

## 场景

对数据库进行实时增量备份。

## 安装配置

#### 准备

`mysql`数据库需要先开启 `Binlog `写入功能，配置` binlog-format` 为 `ROW `模式，`my.cnf `中配置如下

```properties
[mysqld]
log-bin=mysql-bin # 开启 binlog
binlog-format=ROW # 选择 ROW 模式
server_id=1 # 配置 MySQL replaction 需要定义，不要和 canal 的 slaveId 重复
```

#### 拉取Docker镜像

使用`docker`启动`canal-server`

1. 访问`docker hub`获取最新的版本 访问：https://hub.docker.com/r/canal/canal-server/tags/
2. 下载对应的版本

```shell
docker pull canal/canal-server:lastet
```

#### 启动（单机模式）

使用`canal`提供的`run.sh`脚本:

 https://github.com/alibaba/canal/blob/master/docker/run.sh

```shell
# 下载脚本
wget https://raw.githubusercontent.com/alibaba/canal/master/docker/run.sh 
# 构建一个destination name为sys_info的队列，命令中mysql的链接地址为需要备份的数据库
sh run.sh -e canal.auto.scan=false \
		  -e canal.destinations=sys_info \
		  -e canal.instance.master.address=***.mysql.***.rds.aliyuncs.com:3306  \
		  -e canal.instance.dbUsername=****  \
		  -e canal.instance.dbPassword=****  \
		  -e canal.instance.connectionCharset=UTF-8 \
		  -e canal.instance.tsdb.enable=true \
		  -e canal.instance.gtidon=false  
```

`docker`模式下，单`docker`实例只能运行一个`instance`，主要为配置问题。

#### 运行效果

![](https://azrael-1252339850.cos.ap-chengdu.myqcloud.com/20200307141300.png)

successful代表`canal-server`启动成功。

### 启动canal-adapter

#### 配置

`canal-adapter`下载地址：https://github.com/alibaba/canal/releases

获取对应`canal-server(canal.deployer)`版本的`canal-adapter`

![](https://azrael-1252339850.cos.ap-chengdu.myqcloud.com/20200307142325.png)

最好使用稳定版本

```shell
#下载canal-adapter 
#${version} 为对应版本 如 1.1.4
wget https://github.com/alibaba/canal/releases/download/canal-${version}/canal.adapter-${version}.tar.gz 
#下载后解压到指定目录
tar -zxvf canal.adapter-${version}.tar.gz -C canal-adapter
```

修改配置文件` conf/application.yml`为：

```yaml
server:
  port: 8081
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8
    default-property-inclusion: non_null

canal.conf:
  mode: tcp # kafka rocketMQ
  canalServerHost: 127.0.0.1:11111
  batchSize: 500
  syncBatchSize: 1000
  retries: 0
  timeout:
  canalAdapters:
  - instance: sys_info # canal instance Name or mq topic name  
    groups:
    - groupId: g1
      outerAdapters:
      #- name: logger
      - name: rdb  # 指定为rdb类型同步 # 接收数据的新数据库
        key: mysql1 # 指定adapter的唯一key, 与表映射配置中outerAdapterKey对应
        properties:
          jdbc.driverClassName: com.mysql.jdbc.Driver # jdbc驱动名, 部分jdbc的jar包需要自行放致lib目录下
          jdbc.url: jdbc:mysql://***.***.rds.aliyuncs.com:3306/sys_info?useUnicode=true # jdbc url
          jdbc.username: **** # jdbc username
          jdbc.password: **** # jdbc password
          threads: 5     # 并行执行的线程数, 默认为1

```

1. 其中 `outAdapter` 的配置: `name`统一为`rdb`,` key`为对应的数据源的唯一标识需和下面的表映射文件中的`outerAdapterKey`对应,` properties`为目标库`jdb`的相关参数
2. adapter将会自动加载` conf/rdb `下的所有`.yml`结尾的表映射配置文件

#### 适配器列表

```yaml
#最简单的处理, 将受到的变更事件通过日志打印的方式进行输出, 如配置所示, 只需要定义name: logger即可
...
      outerAdapters:                        
      - name: logger 
```

#### RDB表映射文件

修改 `conf/rdb/mytest_user.yml`文件:

```yml
dataSourceKey: defaultDS        # 源数据源的key, 对应上面配置的srcDataSources中的值
destination: sys_info            # cannal的instance或者MQ的topic
groupId:                        # 对应MQ模式下的groupId, 只会同步对应groupId的数据
outerAdapterKey: mysql1        # adapter key, 对应上面配置outAdapters中的key
concurrent: true                # 是否按主键hash并行同步, 并行同步的表必须保证主键不会更改及主键不能为其他同步表的外键!!
dbMapping:
  database: sys_info              # 源数据源的database/shcema
  table: user                   # 源数据源表名
  targetTable: sys_info.tb_user   # 目标数据源的库名.表名
  targetPk:                     # 主键映射
    id: id                      # 如果是复合主键可以换行映射多个
#  mapAll: true                 # 是否整表映射, 要求源表和目标表字段名一模一样 (如果targetColumns也配置了映射,则以targetColumns配置为准)
  targetColumns:                # 字段映射, 格式: 目标表字段: 源表字段, 如果字段名一样源表字段名可不填
    id:
    name:
    role_id:
```

如果两个库之间表、字段都相同可直接进行镜像备份，`Mysql` 库间镜像`schema DDL DML`同步

```yml
## Mirror schema synchronize config
dataSourceKey: defaultDS
destination: sys_info
groupId: g1
outerAdapterKey: mysql1
concurrent: true
dbMapping:
  mirrorDb: true
  database: sys_info
```

其中`dbMapping.database`的值代表源库和目标库的`schema`名称，即两库的`schema`要一模一样

#### 启动

将目标库的`jdbc jar`包放入`lib`文件夹 (其他数据库放入对应的驱动)

启动`canal-adapter`启动器

```bash
bin/startup.sh
```

验证 修改`mysql sys_info.user`表的数据, 将会自动同步到新`mysql`数据库的`sys_info.user`表下面, 并会打出`DML`的`log`

#### adapter管理REST接口

- 查询所有订阅同步的`canal instance`或`MQ topic`

```shell
curl http://127.0.0.1:8081/destinations
```

- 数据同步开关

```shell
curl http://127.0.0.1:8081/syncSwitch/sys_info/off -X PUT
```

针对 `sys_info`这个`canal instance/MQ topic` 进行开关操作. `off`代表关闭,` instance/topic`下的同步将阻塞或者断开连接不再接收数据, on代表开启

注: 如果在配置文件中配置了 `zookeeperHosts `项, 则会使用分布式锁来控制`HA`中的数据同步开关, 如果是单机模式则使用本地锁来控制开关

- 数据同步开关状态

```shell
curl http://127.0.0.1:8081/syncSwitch/sys_info
```

查看指定 canal instance/MQ topic 的数据同步开关状态

## 增加Prometheus监控

安装并部署对应平台的`prometheus`，参见[这里](prometheus-grafana)

配置`prometheus.yml`，添加`canal`的`job`，示例：

```yml
 - job_name: 'canal'
    static_configs:
    - targets: ['localhost:11112'] //端口配置即为canal.properties中的canal.metrics.pull.port
```

导入模板([Canal_instances_tmpl.json](https://raw.githubusercontent.com/alibaba/canal/master/deployer/src/main/resources/metrics/Canal_instances_tmpl.json))，参考[这里](http://docs.grafana.org/reference/export_import/#importing-a-dashboard)

![](https://azrael-1252339850.cos.ap-chengdu.myqcloud.com/20200307150301.png)

导入后效果：

![](https://azrael-1252339850.cos.ap-chengdu.myqcloud.com/20200307145939.png)

参考地址：

https://github.com/alibaba/canal/wiki/QuickStart

https://github.com/alibaba/canal/wiki/Docker-QuickStart

https://github.com/alibaba/canal/wiki/ClientAdapter

https://github.com/alibaba/canal/wiki/Sync-RDB

https://github.com/alibaba/canal/wiki/Prometheus-QuickStart
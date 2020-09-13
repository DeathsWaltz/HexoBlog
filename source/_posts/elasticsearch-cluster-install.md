---
title: docker安装elasticsearch集群
date: 2020-07-20 13:58:19
tags: [elasticsearch,cluster,elasticsearch集群安装,elasticsearch集群安装配置,docker]
categories: [学习,docker,elasticsearch]
---

#### 准备

准备三个`elasticsearch`节点的配置，放在`/etc/elasticsearch/`目录下。

```yaml
#elasticsearch1.yml
# 集群名，三份配置相同
cluster.name: es-cluster-cen
# 节点名，同一个集群的节点名字不能相同
node.name: node-1
# 该节点是否有资格竞选成为主节点
node.master: true
# 该节点是否作为数据节点
node.data: true
# 一台服务器能运行的节点数，根据需要调整
node.max_local_storage_nodes: 3
# 索引数据存储的位置
path.data: /usr/share/elasticsearch/data
# 日志路径
path.logs: /usr/share/elasticsearch/log
# 网关地址
network.host: 0.0.0.0
# 和其他节点通信的地址,如果不设置的话会自动获取
network.publish_host: x.x.x.x
# 定义http传输监听的端口
http.port: 9200
# 设置节点之间通信的端口
transport.tcp.port: 9300
# 7.x新配置，写入候选主节点的设备地址，在开启服务后可以被选为主节点
discovery.seed_hosts: ["x.x.x.x:9300","y.y.y.y:9301","z.z.z.z:9302"]
# 与上一个配置类似，这里用节点名
cluster.initial_master_nodes: ["node-1","node-2","node-3"]
# 集群启动后，复活2个及以上的节点，这个服务才能被使用
gateway.recover_after_nodes: 2
# 允许跨域，可选
http.cors.enabled: true
http.cors.allow-origin: "*"
```

```yaml
# elasticsearch2.yml
cluster.name: es-cluster-cen
node.name: node-2
node.master: true
node.data: true
node.max_local_storage_nodes: 3
path.data: /usr/share/elasticsearch/data
path.logs: /usr/share/elasticsearch/log
network.host: 0.0.0.0
network.publish_host: y.y.y.y
http.port: 9201
transport.tcp.port: 9301
discovery.seed_hosts: ["x.x.x.x:9300","y.y.y.y:9301","z.z.z.z:9302"]
cluster.initial_master_nodes: ["node-1","node-2","node-3"]
gateway.recover_after_nodes: 2
http.cors.enabled: true
http.cors.allow-origin: "*"
```

```yaml
#elasticsearch3.yml
cluster.name: es-cluster-cen
node.name: node-3
node.master: true
node.data: true
node.max_local_storage_nodes: 3
path.data: /usr/share/elasticsearch/data
path.logs: /usr/share/elasticsearch/log
network.host: 0.0.0.0
network.publish_host: z.z.z.z
http.port: 9202
transport.tcp.port: 9302
discovery.seed_hosts: ["x.x.x.x:9300","y.y.y.y:9301","z.z.z.z:9302"]
cluster.initial_master_nodes: ["node-1","node-2","node-3"]
gateway.recover_after_nodes: 2
http.cors.enabled: true
http.cors.allow-origin: "*"
```

拉取镜像

```shell
docker pull elasticsearch:7.7.0
```

#### 启动

```shell
docker run \
-p 9200:9200 \
-p 9300:9300 \
-e ES_JAVA_OPTS="-Xms256m -Xmx256m" \
-v /etc/elasticsearch/elasticsearch1.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
docker.elastic.co/elasticsearch/elasticsearch:7.7.0 
```

```shell
docker run \
-p 9201:9201 \
-p 9301:9301 \
-e ES_JAVA_OPTS="-Xms256m -Xmx256m" \
-v /etc/elasticsearch/elasticsearch2.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
docker.elastic.co/elasticsearch/elasticsearch:7.7.0 
```

```shell
docker run \
-p 9202:9202 \
-p 9302:9302 \
-e ES_JAVA_OPTS="-Xms256m -Xmx256m" \
-v /etc/elasticsearch/elasticsearch2.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
docker.elastic.co/elasticsearch/elasticsearch:7.7.0 
```

启动完成后，任意访问一个节点查看集群状态:

```shell
curl http://127.0.0.1:9200/_cluster/health
```

返回

```json
{
    "cluster_name":"es-cluster-cen",
    "status":"green",
    "timed_out":false,
    "number_of_nodes":3,
    "number_of_data_nodes":3,
    "active_primary_shards":62,
    "active_shards":93,
    "relocating_shards":0,
    "initializing_shards":0,
    "unassigned_shards":31,
    "delayed_unassigned_shards":0,
    "number_of_pending_tasks":0,
    "number_of_in_flight_fetch":0,
    "task_max_waiting_in_queue_millis":0,
    "active_shards_percent_as_number":75
}
```

返回结果为集群的基本信息。



参考文档:

[https://blog.andycen.com/2020/05/24/Elasticsearch%E5%8E%9F%E7%90%86%E4%BB%8B%E7%BB%8D%E4%B8%8E%E9%9B%86%E7%BE%A4%E6%90%AD%E5%BB%BA/](https://blog.andycen.com/2020/05/24/Elasticsearch%E5%8E%9F%E7%90%86%E4%BB%8B%E7%BB%8D%E4%B8%8E%E9%9B%86%E7%BE%A4%E6%90%AD%E5%BB%BA/)


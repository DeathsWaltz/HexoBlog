---
title: elasticsearch异常 Data too large
date: 2020-08-13 13:35:44
tags: [elasticsearch,error,异常,Data too large]
categories: [学习,docker,elasticsearch,error]
---

#### 问题

使用`canal`向`elasticsearch`同步数据时，出现：

```
 Data too large, data for [<transport_request>] would be [256669646/244.7mb], which is larger than the limit of [255013683/243.1mb], real usage: [256666520/244.7mb], new bytes reserved: [3126/3kb], usages [request=0/0b, fielddata=4812/4.6kb, in_flight_requests=3126/3kb, accounting=6615462/6.3mb]
```

#### 原因

堆内存不够当前查询加载数据所以会报错。

#### 解决

提高堆栈内存

修改`elasticsearch`的`config`目录下的`jvm.options`:

```properties
################################################################
## IMPORTANT: JVM heap size
################################################################
##
## You should always set the min and max JVM heap
## size to the same value. For example, to set
## the heap to 4 GB, set:
##
## -Xms4g
## -Xmx4g
##
## See https://www.elastic.co/guide/en/elasticsearch/reference/current/heap-size.html
## for more information
##
################################################################

# Xms represents the initial size of total heap space
# Xmx represents the maximum size of total heap space

-Xms4g
-Xmx6g
```

然后重启`elasticsearch`。



增加堆内存使用率，默认70%，修改为90%：

```shell
curl -X PUT "http://127.0.0.1:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
 {
     "transient" : {
         "indices.breaker.total.limit" : "90%"
     }
 }
```



参考文档:

[https://github.com/docker-library/elasticsearch/issues/98](https://github.com/docker-library/elasticsearch/issues/98)

[https://www.cnblogs.com/zhanchenjin/archive/2019/10/14/11672900.html](https://www.cnblogs.com/zhanchenjin/archive/2019/10/14/11672900.html)
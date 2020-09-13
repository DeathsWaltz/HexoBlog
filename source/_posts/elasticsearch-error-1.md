---
title: elasticsearch异常maximum allowed to be analyzed for highlighting解决
date: 2020-09-13 13:25:29
tags: [elasticsearch,error,异常,maximum allowed to be analyzed for highlighting]
categories: [学习,docker,elasticsearch,error]
---

#### 问题

根据指定字段进行模糊查询，进行查询后报错：

```json
{"type":"illegal_argument_exception","reason":"The length of [message] field of [SY_yKc9yQj6HAx8MKMt1pQ] doc of [syslog] index has exceeded [1000000] - maximum allowed to be analyzed for highlighting. This maximum can be set by changing the [index.highlight.max_analyzed_offset] index level setting. For large texts, indexing with offsets or term vectors is recommended!”}}
```

#### 原因

索引偏移量默认是`100000`，当前索引偏移量超过默认值。

#### 解决

修改索引偏移量

```shell
curl -XPUT "http://127.0.0.1:9200/_settings" -H 'Content-Type: application/json' -d' {
    "index" : {
        "highlight.max_analyzed_offset" : 100000000
    }
}
```

参考文档：

[https://www.cnblogs.com/zhanchenjin/archive/2019/10/14/11672900.html](https://www.cnblogs.com/zhanchenjin/archive/2019/10/14/11672900.html)




---
title: mysql保存日期mybatis查询后多8小时问题
date: 2019-10-19 11:11:10
tags: [mysql,mybaits,java]
---



#### 问题

发现查询出的日期数据与mysql数据库中的日期不符.

mysql查询出的数据是 2019-10-15 13:31:10 mybatis查询出的数据是 2019-10-15 05:31:10.

  #### 原因

因mysql设置时区与 Tomcat java 使用的时区不同.

#### 解决

在数据库链接配置设置时区.

增加`serverTimezone=Asia/Shanghai`

```
jdbc:mysql://localhost:3306/db?serverTimezone=Asia/Shanghai
```





参考地址：

 [https://blog.csdn.net/babybabyup/article/details/97802707](https://blog.csdn.net/babybabyup/article/details/97802707 )

 [https://blog.csdn.net/saroll57/article/details/51837551](https://blog.csdn.net/saroll57/article/details/51837551)

 [https://blog.csdn.net/a549654065/article/details/88664077 ](https://blog.csdn.net/a549654065/article/details/88664077)


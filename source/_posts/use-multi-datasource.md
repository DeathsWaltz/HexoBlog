---
title: Spring boot使用多数据源
date: 2019-09-11 15:56:39
tags: [springboot,多数据源]
---

#### 依赖： pom.xml

```xml
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
        <version>${baomidou.version}</version>
    </dependency>
```

#### 数据源配置： application.yml

```yaml
spring:
  datasource:
    dynamic:
      datasource:
        master:
          driver-class-name: com.mysql.cj.jdbc.Driver
          url: jdbc:mysql://129.xx.xx.xx:3306/jiahua?useUnicode=true&characterEncoding=UTF-8&useSSL=false
          username: xxxx
          password: xxxx
        slave:
          driver-class-name: com.microsoft.sqlserver.jdbc.SQLServerDriver
          url: jdbc:sqlserver://192.168.xx.xx:1433; DatabaseName=xxx1
          username: xxxx
          password: xxxx
        slave1:
          driver-class-name: com.microsoft.sqlserver.jdbc.SQLServerDriver
          url: jdbc:sqlserver://192.168.xx.xx:1433; DatabaseName=xxx2
          username: xxxx
          password: xxxx
      mp-enabled: true
```



#### ServiceImpl

@DS: 注解在类上或方法上来切换数据源，方法上的@DS优先级大于类上的@DS
默认走从库: @DS(value = "slave")在类上，默认走从库，除非在方法在添加@DS(value = "master")才走主库

```java
@Service
@DS(value = "slave")
public class ItemServiceImpl extends ServiceImpl<ItemMapper, Item> implements ItemService {
    /**
     * 类上 {@code @DS("slave")} 代表默认从库，在方法上写 {@code @DS("master")} 代表默认主库
     *
     * @param user 用户
     */
    @DS("master")
    @Override
    public void addUser(User user) {
        baseMapper.insert(user);
    }
}
```


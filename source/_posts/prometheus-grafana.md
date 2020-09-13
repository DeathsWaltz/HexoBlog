---
title: Prometheus、Grafana安装和配置监控Java应用
date: 2020-03-07 10:32:11
tags: [Prometheus,Grafana,Java,spring boot]
categories: [学习,docker,Grafana]
---

## Prometheus安装

使用`docker`安装

`docker`已经安装好之后，正式安装`prometheus`。

```shell
docker run -d  -p 9090:9090 -v /etc/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml   -v /etc/localtime:/etc/localtime prom/prometheus

```

如果启动成功，访问`http://服务器地址:9090`

## node export 安装

```shell
docker run -d  -p 9100:9100 quay.io/prometheus/node-exporter
```

安装完之后，需要修改`prometheus.yml`配置文件，增加要监听的job，需要指定job的名称，以及暴露的`metrics`的访问路径

```yml
  - job_name: "node"
    # metrics_path defaults to '/metrics'
    #  scheme defaults to 'http'.
    static_configs:
      - targets: ["localhost:9001"]  #使用宿主机内网ip
```

重启`prometheus`容器生效

## Grafana安装

可以使用`grafana`展示监控视图。

```shell
docker run -d --name=grafana -p 3001:3000 -v /var/lib/grafana:/var/lib/grafana -v /etc/grafana/grafana.ini:/etc/grafana/grafana.ini   -v /etc/localtime:/etc/localtime grafana/grafana 
```

访问`http://ip地址：指定的端口`，`grafana`安装成功，第一次访问需要修改密码，初始密码是`admin/admin`,修改密码之后，需要按照新密码登录。

### grafana配置

##### 添加数据源

![添加数据源](https://azrael-1252339850.cos.ap-chengdu.myqcloud.com/20200307104004.png)

![](https://azrael-1252339850.cos.ap-chengdu.myqcloud.com/20200307104106.png)

![](https://azrael-1252339850.cos.ap-chengdu.myqcloud.com/20200307113107.png)

`Name`为数据源名，`URL`为`prometheus`地址

##### 导入模板

![导入模板](https://azrael-1252339850.cos.ap-chengdu.myqcloud.com/20200307113316.png)

![](https://azrael-1252339850.cos.ap-chengdu.myqcloud.com/20200307113537.png)

填入`8919`导入模板

![](https://azrael-1252339850.cos.ap-chengdu.myqcloud.com/20200307113623.png)

`Prometheus Data Source`为`grafana`添加的数据源

## 监控jvm

web项目中增加依赖

```xml
<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-actuator</artifactId>
	</dependency>
	<dependency>
		<groupId>io.micrometer</groupId>
		<artifactId>micrometer-registry-prometheus</artifactId>
		<version>1.1.4</version>
	</dependency>
```

在`application.yml`中添加配置

```yaml
management:
  endpoints:
    web:
      exposure:
        include: '*'  #开启 Actuator 服务
  metrics:			#将该工程应用名称添加到计量器注册表的 tag 中
    tags:
      application: ${spring.application.name}  #应用名称比如：app

```

在工程启动主类中增加`监控 JVM 性能指标注释下的`内容：

```java
@SpringBootApplication
public class WebsiteApplication {

    public static void main(String[] args) {
        SpringApplication.run(WebsiteApplication.class, args);
    }
    //监控 JVM 性能指标
    @Bean
    MeterRegistryCustomizer<MeterRegistry> configurer(@Value("${spring.application.name}") String applicationName){
        return registry -> registry.config().commonTags("application", applicationName);
    }
}
```

启动动服务，浏览器访问 `http://127.0.0.1:8088/actuator/prometheus` 就可以看到应用的 一系列不同类型 `metrics` 信息

#### Prometheus配置新增

在`prometheus.yml`文件新增如下配置：

```yaml
- job_name: 'application'
    scrape_interval: 5s
    metrics_path: '/actuator/prometheus'
    file_sd_configs:
      - files: ['/etc/prometheus/*.json']
```

新建`json`文件`app.json`,内容如下：

```json
[
    {
        "targets": [
            "ip:port"	//项目访问地址
        ],
        "labels": {
            "instance": "app",	//可使用应用名称
            "service": "app-service"
        }
    }
    //可增加多个项目的配置
]
```

重启 `Prometheus` 服务，查看 `Prometheu `界面 `Target` 中确认是否添加成功。

![](https://azrael-1252339850.cos.ap-chengdu.myqcloud.com/20200307135140.png)

![](https://azrael-1252339850.cos.ap-chengdu.myqcloud.com/20200307135341.png)

#### 配置 Grafana JVM Dashboard 监控项

参考grafana配置中导入模板的操作，导入`4701`模板

效果如下：

![](https://azrael-1252339850.cos.ap-chengdu.myqcloud.com/20200307134443.png)





参考地址：

https://blog.csdn.net/aixiaoyang168/article/details/100866159

https://cloud.tencent.com/developer/article/1442143

https://www.jianshu.com/p/12df755f2c66
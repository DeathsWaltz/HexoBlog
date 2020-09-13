---
title: Spring boot分库分表方案
date: 2019-09-11 16:51:59
tags: [mycat,database,sharding-jdbc,分库分表,sqlserver]
categories: [学习]
---




### 问题

> 因业务原因需要实现查询多库操作，目前多个库中表结构相同。

> 同时使用mysql数据库和sqlserver数据库。



### Sharding-jdbc方案

#### 项目配置

依赖

```xml
    <dependency>
        <groupId>io.shardingsphere</groupId>
        <artifactId>sharding-jdbc-core</artifactId>
        <version>${shardingsphere.version}</version>
    </dependency>
```

##### 配置sharding-jdbc参数


```java
package com.jiahua.ddxdataserver.config;

import com.zaxxer.hikari.HikariDataSource;
import io.shardingsphere.api.config.rule.ShardingRuleConfiguration;
import io.shardingsphere.api.config.rule.TableRuleConfiguration;
import io.shardingsphere.api.config.strategy.NoneShardingStrategyConfiguration;
import io.shardingsphere.shardingjdbc.api.ShardingDataSourceFactory;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Primary;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;

import javax.sql.DataSource;
import java.sql.SQLException;
import java.util.HashMap;
import java.util.Map;
import java.util.Properties;
import java.util.concurrent.ConcurrentHashMap;

@Configuration
public class DataSourceShardingConfig {

    /**
     * 需要手动配置事务管理器
     */
    @Bean
    public DataSourceTransactionManager transactionManager(@Qualifier("dataSource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

    @Bean(name = "dataSource")
    @Primary
    public DataSource dataSource() throws SQLException {
        ShardingRuleConfiguration shardingRuleConfig = new ShardingRuleConfiguration();
        // 设置分表策略
        shardingRuleConfig.getTableRuleConfigs().add(orderTableRule());
        shardingRuleConfig.setDefaultDataSourceName("ds0");
        shardingRuleConfig.setDefaultTableShardingStrategyConfig(new NoneShardingStrategyConfiguration());

        Properties properties = new Properties();
        properties.setProperty("sql.show", "true");

        return ShardingDataSourceFactory.createDataSource(dataSourceMap(), shardingRuleConfig, new ConcurrentHashMap<>(16), properties);
    }

    private TableRuleConfiguration orderTableRule() {
        TableRuleConfiguration tableRule = new TableRuleConfiguration();
        // 设置逻辑表名
        tableRule.setLogicTable("top_XXX");
        // ds${0..1}.t_order_${0..2} 也可以写成 ds$->{0..1}.t_order_$->{0..1}
        tableRule.setActualDataNodes("ds${0..2}.top_order");
        return tableRule;
    }


    private Map<String, DataSource> dataSourceMap() {
        Map<String, DataSource> dataSourceMap = new HashMap<>(16);

        // 配置第一个数据源
        HikariDataSource ds0 = new HikariDataSource();
        ds0.setDriverClassName("com.microsoft.sqlserver.jdbc.SQLServerDriver");
        ds0.setJdbcUrl("jdbc:sqlserver://192.168.xx.xx:1433; DatabaseName=XXXX1");
        ds0.setUsername("XXXX");
        ds0.setPassword("XXXX");

        // 配置第二个数据源
        HikariDataSource ds1 = new HikariDataSource();
        ds1.setDriverClassName("com.microsoft.sqlserver.jdbc.SQLServerDriver");
        ds1.setJdbcUrl("jdbc:sqlserver://192.168.xx.xx:1433; DatabaseName=XXXX2");
        ds1.setUsername("XXXX");
        ds1.setPassword("XXXX");

        // 配置第三个数据源
        HikariDataSource ds2 = new HikariDataSource();
        ds2.setDriverClassName("com.microsoft.sqlserver.jdbc.SQLServerDriver");
        ds2.setJdbcUrl("jdbc:sqlserver://192.168.XX.XX:1433; DatabaseName=XXXX3");
        ds2.setUsername("XXXX");
        ds2.setPassword("XXXX");

        dataSourceMap.put("ds0", ds0);
        dataSourceMap.put("ds1", ds1);
        dataSourceMap.put("ds2", ds2);
        return dataSourceMap;
    }



}

```

##### 存在问题

> 使用后发现，sqlserver数据库中nvarchar类型无法对应java中String类型，报错如下：

```java
org.springframework.dao.InvalidDataAccessApiUsageException: Error attempting to get column 'top_sku_properties_name' from result set.  Cause: java.sql.SQLFeatureNotSupportedException: getNString
; getNString; nested exception is java.sql.SQLFeatureNotSupportedException: getNString
```

> 暂未找到解决办法，改用mycat.

---



### Mycat方案

mycat官网 [http://mycat.io/](http://mycat.io/)

#### mycat配置

##### server.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- - - Licensed under the Apache License, Version 2.0 (the "License"); 
	- you may not use this file except in compliance with the License. - You 
	may obtain a copy of the License at - - http://www.apache.org/licenses/LICENSE-2.0 
	- - Unless required by applicable law or agreed to in writing, software - 
	distributed under the License is distributed on an "AS IS" BASIS, - WITHOUT 
	WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. - See the 
	License for the specific language governing permissions and - limitations 
	under the License. -->
<!DOCTYPE mycat:server SYSTEM "server.dtd">
<mycat:server xmlns:mycat="http://io.mycat/">
	<system>
	<property name="nonePasswordLogin">0</property> <!-- 0为需要密码登陆、1为不需要密码登陆 ,默认为0，设置为1则需要指定默认账户-->
	<property name="useHandshakeV10">1</property>
	<property name="useSqlStat">0</property>  <!-- 1为开启实时统计、0为关闭 -->
	<property name="useGlobleTableCheck">0</property>  <!-- 1为开启全加班一致性检测、0为关闭 -->
		<property name="sqlExecuteTimeout">300</property>  <!-- SQL 执行超时 单位:秒-->
		<property name="sequnceHandlerType">2</property>
		<!--<property name="sequnceHandlerPattern">(?:(\s*next\s+value\s+for\s*MYCATSEQ_(\w+))(,|\)|\s)*)+</property>-->
		<!--必须带有MYCATSEQ_或者 mycatseq_进入序列匹配流程 注意MYCATSEQ_有空格的情况-->
		<property name="sequnceHandlerPattern">(?:(\s*next\s+value\s+for\s*MYCATSEQ_(\w+))(,|\)|\s)*)+</property>
	<property name="subqueryRelationshipCheck">false</property> <!-- 子查询中存在关联查询的情况下,检查关联字段中是否有分片字段 .默认 false -->
      <!--  <property name="useCompression">1</property>--> <!--1为开启mysql压缩协议-->
        <!--  <property name="fakeMySQLVersion">5.6.20</property>--> <!--设置模拟的MySQL版本号-->
	<!-- <property name="processorBufferChunk">40960</property> -->
	<!-- 
	<property name="processors">1</property> 
	<property name="processorExecutor">32</property> 
	 -->
        <!--默认为type 0: DirectByteBufferPool | type 1 ByteBufferArena | type 2 NettyBufferPool -->
		<property name="processorBufferPoolType">0</property>
		<!--默认是65535 64K 用于sql解析时最大文本长度 -->
		<!--<property name="maxStringLiteralLength">65535</property>-->
		<!--<property name="sequnceHandlerType">0</property>-->
		<!--<property name="backSocketNoDelay">1</property>-->
		<!--<property name="frontSocketNoDelay">1</property>-->
		<!--<property name="processorExecutor">16</property>-->
		<!--
			<property name="serverPort">8066</property> <property name="managerPort">9066</property> 
			<property name="idleTimeout">300000</property> <property name="bindIp">0.0.0.0</property>
			<property name="dataNodeIdleCheckPeriod">300000</property> 5 * 60 * 1000L; //连接空闲检查
			<property name="frontWriteQueueSize">4096</property> <property name="processors">32</property> -->
		<!--分布式事务开关，0为不过滤分布式事务，1为过滤分布式事务（如果分布式事务内只涉及全局表，则不过滤），2为不过滤分布式事务,但是记录分布式事务日志-->
		<property name="handleDistributedTransactions">0</property>
		
			<!--
			off heap for merge/order/group/limit      1开启   0关闭
		-->
		<property name="useOffHeapForMerge">1</property>

		<!--
			单位为m
		-->
        <property name="memoryPageSize">64k</property>

		<!--
			单位为k
		-->
		<property name="spillsFileBufferSize">1k</property>

		<property name="useStreamOutput">0</property>

		<!--
			单位为m
		-->
		<property name="systemReserveMemorySize">384m</property>


		<!--是否采用zookeeper协调切换  -->
		<property name="useZKSwitch">false</property>

		<!-- XA Recovery Log日志路径 -->
		<!--<property name="XARecoveryLogBaseDir">./</property>-->

		<!-- XA Recovery Log日志名称 -->
		<!--<property name="XARecoveryLogBaseName">tmlog</property>-->
		<!--如果为 true的话 严格遵守隔离级别,不会在仅仅只有select语句的时候在事务中切换连接-->
		<property name="strictTxIsolation">false</property>
		
		<property name="useZKSwitch">true</property>
		
	</system>

<user name="xxxx">
		<property name="password">xxxx</property>
		<property name="schemas">box</property>
	</user>


</mycat:server>

```



##### schema.xml

```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">

	<schema name="box" checkSQLschema="false" sqlMaxLimit="100">
		<!-- 需要查询的表 -->
		<table name="top_xxx" dataNode="ds0,ds1,ds3" needAddLimit="false" />
		<table name="top_xxxx" dataNode="ds0,ds1,ds3" needAddLimit="false" />
	</schema>

    <!-- 数据库节点 -->
	<dataNode name="ds0" dataHost="sqlserver1" database="xxx1" />
	<dataNode name="ds1" dataHost="sqlserver1" database="xxx2" />
	<dataNode name="ds3" dataHost="sqlserver1" database="xxx3" />
    
	<!-- sqlserver数据库配置 以jdbc方式链接 -->
	<dataHost name="sqlserver1" maxCon="1000" minCon="10" balance="0"
			  writeType="0" dbType="sqlserver" dbDriver="jdbc" >
		<heartbeat></heartbeat>
		<connectionInitSql></connectionInitSql>
		<writeHost host="hostM1" url="jdbc:sqlserver://192.168.xx.xx:1433" user="xxxx"
				   password="xxxx" >
				</writeHost>

	</dataHost>
	
</mycat:schema>
```

> `server.xml`中`user`标签对应`schema.xml`中`writeHost`标签的`user`.



##### rule.xml

不使用规则可不配置rule.xml (待研究.

#### 启动mycat

```shell
mycat start
```

> mycat启动后默认端口为8066，用户名、密码、数据库为`server.xml`中`user`的配置.
>
> 例：

```xml
<user name="xxxx">
	<property name="password">xxxx</property>
	<property name="schemas">box</property>
</user>
```


> mycat启动后，项目中配置数据源后即可使用.

#### 使用多数据源

同时链接mysql和mycat。

参考  [多数据源配置](/2019/09/11/use-multi-datasource/)



##### 参考：

[https://github.com/xkcoding/spring-boot-demo](https://github.com/xkcoding/spring-boot-demo)

[http://www.mycat.io/document/mycat-definitive-guide.pdf](http://www.mycat.io/document/mycat-definitive-guide.pdf)
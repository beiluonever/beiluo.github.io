---
title: MySQL8驱动逆向工程或Flowable代码生成或Mybatis代码生成问题解决
date: 2019-12-15 18:51:41
tags:
- MySQL
- Flowable
categories:
- Java
---
# MySQL 8版本中代码生成时出现无法生成或错误对象信息的问题解决
## Flowable代码生成时出现无法创建表，报错表不存在
### 情况说明：
1. 数据库中有多个库，并且另一个库中已经有了flowable或activiti相关的表
2. 配置中已经添加了自动生成表的配置database-schema-update: true
### 现象：
无法创建表，一直报表不存在
```
Caused by: java.sql.SQLSyntaxErrorException: Table 'workflow.act_ge_property' doesn't exist
```
### 解决方法：
1. 在数据库链接后添加配置：nullCatalogMeansCurrent = true
2. 添加SpringBoot配置-根据自己使用的数据库池:
```
spring.datasource.hikari.data-source-properties.nullCatalogMeansCurrent=true
```
3. Java代码配置-根据自己使用的数据库池:
```
HikariConfig config = new HikariConfig();
...
config.addDataSourceProperty("nullCatalogMeansCurrent", true);
```
### 导致的原因分析
在MySQL官方更新日志中发现配置修改nullCatalogMeansCurrent从默认为True改成了默认为False，如果你使用DatabaseMetaData.getTables获取所有的表信息，8.0版本驱动将返回所有库的表。
[MySQL Changes in Connection Properties](https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-properties-changed.html)
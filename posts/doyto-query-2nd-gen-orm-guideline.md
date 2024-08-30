---
title: 'DoytoQuery——第二代ORM框架的Java实现（提纲版）'
date: 2021-05-24 20:53:12
tags: []
published: true
hideInList: true
feature: 
isTop: false
---
# 背景摘要
背景：当前的ORM框架都是将数据库表映射为实体(Entity)对象来对数据库进行访问操作。
问题：动态查询语句和复杂的连接查询需要大量编码，且有结构型重复。
方案：既然通过SQL语句进行数据库操作，那就将SQL语句作为对象映射的目标。
结果：动态查询和复杂查询的开发大大简化，并由此搭建了一整套闭合的Web应用分层体系。

# 第二代ORM框架概述

## 一条核心理念
**将SQL语句映射为对象进行数据库访问操作**

## 两大理论基础
#### 一、一条SQL语句最多映射为三个对象
1. WHERE语句映射为Query对象
    - 每条查询语句都包含了隐式分页和排序
    - 查询字段可以通过以下三种方式映射为条件语句
        - 字段后缀映射
        - 嵌套查询注解
        - 自定义查询注解
2. SELECT语句映射为Entity对象
3. 水平分表的动态表名由IdWrapper对象确定

#### 二、 SQL访问语句可以归为三类
1. 单表增删查改
    - 基于上述三个对象映射单表的增删查改语句
2. 中间表操作
    - 表结构类似，变量部分有3个：表名，左表外键列名，右表外键列名
    - 只有增删查操作，没有修改操作
    - 根据上述变量可以生成对应的增删查语句
3. 聚合/连接查询
    - 只有查询操作
    - 复用Query对象生成查询语句
    - Entity对象的字段的注解用于拼接SELECT语句
    - 在Entity对象上使用注解拼接JOIN和GROUP BY语句

# 数据库访问操作的两大步骤及对应的代表框架

1. SQL生成
    - JPA/Hibernate
    - MyBatis Generator
    - MyBatis-Plus
    - tkMapper
    - **DoytoQuery**(core)
  
2. SQL执行
    - Hibernate
    - Mybatis
    - Spring JDBC
    - DbUtils

## DoytoQuery的对应实现

| SQL分类 |        接口        |            实现            |         SQL构建          |   SQL执行   |
| :-----: | :----------------: | :------------------------: | :----------------------: | :---------: |
|  单表   |     DataAccess     |       JdbcDataAccess       | QueryBuilder/CrudBuilder | Spring-Jdbc |
| 中间表  | AssociativeService | TemplateAssociativeService |  AssociativeSqlBuilder   | Spring-Jdbc |
| 连接表  |    QueryService    |      JoinQueryService      |     JoinQueryBuilder     | Spring-Jdbc |

# 基于DataAccess接口的分层体系搭建
## DataAccess层
- 单表增删查改操作
- 数据库方言扩展
- DataAccess接口的一个Mock实现

## Service层
- 增删查改接口
- 二级缓存
- UserId注入
- EntityAspect扩展

## Controller层
- RESTFul API 接口
- 错误码
- 异常断言
- 异常处理
- 响应封装
- Request/Entity/Response转换
- 分组校验

# 相关资源
- 源码
    [https://github.com/f0rb/doyto-query]()
    [https://github.com/f0rb/doyto-query-web]()
    [https://github.com/doytowin/doyto-query-dialect]()
- 相关项目
    [Demo演示](https://github.com/f0rb/doyto-query-demo)
    [Idea插件]([https://github.com/doytowin/doyto-query-intellij-plugin) todo
    [多语言管理服务](https://github.com/f0rb/doyto-service-i18n)
    [代码生成服务](https://gitee.com/doyto/doyto-service-generator)
- 文档
    [https://query.doyto.win/]()
- Sonar
    [https://sonarcloud.io/dashboard?id=win.doyto%3Adoyto-query]()
- Maven
    [https://search.maven.org/search?q=g:win.doyto]()    

# 扩展

话题

使用其他语言实现
前缀分包vs后缀分包
安全问题和性能问题的识别和改进
重构理念在DoytoQuery演进过程中的驱动作用
更优雅的缓存清理方案
多高的测试覆盖率才能保证没有bug

功能

旧系统的迁移改造方案
patch(Query)和delete(Query)接口的非空检查
IDE的插件，包括查询字段的后缀提示和单表组件的代码生成等功能
相关文档和教程
流量控制功能
SQL记录功能
代码热部署

一点牢骚：
面向对象开发没问题，面向的对象错了才有问题。


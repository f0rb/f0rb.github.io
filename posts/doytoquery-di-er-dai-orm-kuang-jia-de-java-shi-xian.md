---
title: 'DoytoQuery——第二代ORM框架的Java实现'
date: 2021-05-24 20:53:12
tags: []
published: false
hideInList: false
feature: 
isTop: false
---
# 背景介绍

ORM的概念由来已久，全称是Object-Relational Mapping，即对象关系映射，也就是在开发中，通过将数据库表映射为实体(Entity)对象来对数据库进行增删查改等操作，

诞生了一系列优秀的框架，如Hibernate/JPA，MyBatis，
局限于实体和表的对象映射关系，
但仍然遗留了很多问题，
业务开发中的动态查询就很难写，尤其涉及到复杂的连表查询以及查询调优时，可能还需要手写SQL，这样完全没法发挥面向对象的优势，
比如使用JpaSpecification进行查询时，


但是如果换个思路，将SQL语句作为对象映射的目标时，原来基于表映射的ORM理念所衍生出的很多问题就迎刃而解了。

# 第二代ORM框架核心理念和两大基础

## 动态查询和Query对象

- 后缀映射
- 嵌套查询
- 自定义查询

## Entity对象
Entity对象的用于映射SELECT语句里的列字段。

## 水平分表和IdWrapper对象
当单表的数据量很大并且不断增长时，访问性能会不断下降，一种常见的解决方案就是进行水平分表，

user_${hash}这样的表名，还需要一个参数来确定hash的值，

- get(ID)
- delete(ID)

## 小结：SQL语句映射的三类对象

# 中间表操作
多对多关系通常通过中间表来进行关联，

相似的结构有着相似的行为，


# 聚合查询和连接查询
- 表别名

# 小结：SQL访问语句的三大种类
单表的增删查改
中间表的增删改
聚合查询和连接查询的查询SQL

核心理念就是针对SQL语句做对象映射

## 数据库访问操作的两个步骤
SQL生成             
SQL执行
              接口                        SQL构建                             SQL执行 
    单表        DataAccess              CrudBuilder                    spring-jdbc
    中间表     AssociativeService     AssociativeSqlBuilder
    连接表     QueryService       JoinQueryBuilder

# DataAccess
### MemoryDataAccess——DataAccess的mock实现


# 基于DataAccess接口的分层体系搭建

Controller
Service
DataAccess
MemoryDataAccess JdbcDataAccess

## 的实现类

## Service层
-   增删查改接口
-   二级缓存
-   UserId注入
-   EntityAspect扩展
基于开闭原则，避免在单表操作后相关联的其他操作导致Service类引入其他依赖

## Controller层
- RestApi
- 错误码
- 异常断言
- 异常处理
- 响应封装
- 请求对象的分组校验：在Request对象字段的校验注解中，关联对应的Group

# 好处
学习成本大幅降低
开发效率大幅提高
组织结构更加规范

# Demo


RoleQuery RoleEntity RoleController

# 扩展

话题

使用其他语言来实现
安全问题和性能问题的识别和改进
重构理念在DoytoQuery演进过程中的驱动作用
更优雅的缓存清理方案

周边

功能

旧系统的迁移改造方案
patch(Query)和delete(Query)接口的非空检查
IDE的插件，包括查询字段的后缀提示和单表组件的代码生成等功能
相关文档和教程
流量控制功能
SQL记录功能
代码热部署


参考资料：
[Object–relational mapping](https://en.wikipedia.org/wiki/Object%E2%80%93relational_mapping)

---
title: '在GoooQo中怎么表达select * from user where id = ? or (name = ? and age = ?)'
date: 2024-09-02 18:58:26
tags: [GoooQo,Go,OQM,cn,CRUD,f0rb说]
published: true
hideInList: false
feature: /post-images/goooqo-or-clause.webp
isTop: false
---

## 前言

GoooQo是一个可以仅根据对象自动构建SQL语句并执行的OQM框架。

OQM是一项仅通过对象来构建数据库查询语句的技术，专注于研究面向对象编程语言和数据库查询语言之间的映射关系。

本文就标题中的问题来探讨一下SQL语句中逻辑运算符和对象结构之间的关系。

## 查询子句的层级结构

对于SQL语言中的WHERE子句，OQM技术提出一个新的观点：查询子句并不是扁平的，而是有层级的。

其中，使用AND连接的查询条件位于一个层级，使用OR连接的查询条件位于一个层级，每一层级的查询条件使用同一种逻辑运算符进行连接。

查询子句之所以有层级划分，是因为逻辑运算符的优先级不同，AND的优先级高于OR。

一般情况下，多个查询条件是用AND连接的，我们默认AND作为第一层级，那么标题中的查询语句里，OR连接的两个条件就在第二层级，OR语句的第二个条件里用AND连接的两个条件就在第三层级。

使用JSON表示这样的层级结构如下:

```json
{
  "userOr": {
    "id": 5,
    "userAnd": {
      "name": "John",
      "age": 30
    }
  }
}
```

在这条数据中，我们通过键名称的后缀Or和And来确定子查询条件的逻辑运算符。

这条数据转换为WHERE子句的构建过程如下：

* 第一层级只有一个键userOr，可以省掉AND连接；
* userOr下的多个查询条件使用OR进行连接；
* 第一个键值对`"id": 5`对应的查询条件为`id = ?`；
* 第二个键userAnd的后缀为And，使用AND连接子查询条件，得到`name = ? ADN age = ?`；
* 再把这个条件加上括号和查询条件`id = ?`用OR连起来。

这样，我们就得到了标题中的查询子句：`id = ? OR (name = ? AND age = ?)`，以及对应的三个参数：\[5, name, 30]。

由于AND的优先级高于OR，这里的括号也可以省略。

![对比图](https://blog.doyto.win/post-images/1725275009209.webp)


通过这张对比图，我们可以更清晰的看清楚查询子句和JSON数据库之间的对应关系。


## 数据构造和映射关系

然后我们尝试使用Go语言来构造这条JSON数据。

首先，我们定义对应的结构体`UserQuery`，可以使用如下嵌套的方式来定义：
```go
type UserQuery struct {
    Id      *int       `json:"id,omitempty"`
    Name    *string    `json:"name,omitempty"`
    Age     *int       `json:"age,omitempty"`
    UserOr  *UserQuery `json:"userOr,omitempty"`
    UserAnd *UserQuery `json:"userAnd,omitempty"`
}
```

然后，根据JSON数据中的赋值构造对应的变量：
```go
func P[T any](t T) *T { return &t }

func main() {
    userQuery := UserQuery{UserOr: &UserQuery{Id: P(1), UserAnd: &UserQuery{Name: P("John"), Age: P(30)}}}
    bytes, _ := json.Marshal(query)
    print(string(bytes))
}
```  

运行后输出如下，没有赋值的字段都被忽略了：
```json
{"userOr":{"id":1,"userAnd":{"name":"John","age":30}}}
```

这与上面的JSON数据完全一致。也就是说，我们可以根据这里的`userQuery`变量构造出同样的查询子句。

## GoooQo接口调用

GoooQo就是利用对象结构和查询子句的层级之间的联系来构造包含AND和OR的查询子句的。

首先为user表定义实体对象和查询对象：
```go
type UserEntity struct {
    goooqo.IntId
    Name     *string `json:"name,omitempty"`
    Age      *int    `json:"age,omitempty"`\
}

func (u UserEntity) GetTableName() string { return "user" }

type UserQuery struct {
    goooqo.PageQuery
    Id      *int       `json:"id,omitempty"`
    Name    *string    `json:"name,omitempty"`
    Age     *int       `json:"age,omitempty"`
    UserOr  *UserQuery `json:"userOr,omitempty"`
    UserAnd *UserQuery `json:"userAnd,omitempty"`
}
```

再根据数据库连接为user表创建数据访问接口：
```go
db := rdb.Connect("local.properties")
defer rdb.Disconnect(db)
tm := rdb.NewTransactionManager(db)

userDataAccess := rdb.NewTxDataAccess[UserEntity](tm)
```

最后，根据需要执行的SQL语句，使用创建的查询对象调用`userDataAccess`的Query方法即可:
```go
userQuery := UserQuery{UserOr: &UserQuery{Id: P(1), UserAnd: &UserQuery{Name: P("John"), Age: P(30)}}}
userEntities, err := userDataAccess.Query(context.Background(), userQuery)
```

对应的日志信息如下，与标题中的SQL语句基本一致：
```
time="2024-09-02T17:11:27+08:00" level=info msg=Executing SQL="SELECT id, name, age FROM user WHERE (id = ? OR name = ? AND age = ?)" args="[5 John 30]"
```

这样，当我们需要执行不同的SQL语句的时候，只需要根据查询条件给对应的字段赋值即可。对于前后端交互的项目，使用这类查询对象接收前端传入的参数，可以极大地减少后端的编码量。
## 改进

这里还有个小问题，当Or后缀的字段的类型是个数组呢，例如`UserOr *[]UserQuery`？

GoooQo里的处理如下：
- 数组中每个结构体内部字段映射得到的查询条件采用AND连接；
- 多个结构体对应的每组查询条件再用OR连接。

新的结构体定义如下：
```go
type UserQuery struct {
    goooqo.PageQuery
    Id      *int         `json:"id,omitempty"`
    Name    *string      `json:"name,omitempty"`
    Age     *int         `json:"age,omitempty"`
    UserOr  *[]UserQuery `json:"userOr,omitempty"`
    UserAnd *UserQuery   `json:"userAnd,omitempty"`
}
```
然后我们使用新的结构体构造同样的查询语句如下：
```
userQuery := UserQuery{UserOr: &[]UserQuery{ {Id: P(1)}, {Name: P("John"), Age: P(30)}}}
//json: {"userOr":[{"id":1},{"name":"John","age":30}]}
```

此时，键userOr变为了一个包含两个元素的数组，元素之间的运算符为OR，元素内部的运算符为AND。

于是，查询语句的每个查询条件和对象字段的对应关系表示如下：
![对比图](https://blog.doyto.win/post-images/1725615897117.webp)

另外，我们还可以发现，当数组userOr中仅包含一个元素时，它实际上与字段`UserAnd`映射得到的条件相同，于是，我们可以进一步优化掉`UserAnd`字段。

## 结语

本文介绍了查询子句的层级关系以及与对象结构之间的联系，并详细介绍了OQM技术中如何利用对象结构来构造包含AND和OR运算符的查询子句。

传统的ORM技术专注于对象模型和关系模型的映射，并没有指出对象语言和查询语言的这种联系，因而ORM框架对于含OR的查询子句的构造没有统一的实现方式。

而且进一步研究我们会发现，这样的查询条件既然能用JSON数据表示出来，那就意味着只要是支持JSON的编程语言都可以构造出这样的查询条件。下一步，我们可以尝试在不同的编程语言中实现本文所述方案。

---
© 2024 Yuan Zhen. All rights reserved.

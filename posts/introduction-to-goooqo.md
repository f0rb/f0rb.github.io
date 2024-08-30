---
title: '什么?！Golang里两行代码就能搞定增删查改接口了？'
date: 2024-08-26 11:24:31
tags: [OQM,GoooQo,Go,f0rb说,CRUD,cn]
published: true
hideInList: false
feature: 
isTop: false
---


这都得益于OQM（Object Query-Language Mapping）技术的加持，我们开发了一个Go版本的实现，[GoooQo](https://github.com/doytowin/goooqo) 。

<!-- more -->

### OQM介绍

OQM技术与传统的ORM（Object Relational Mapping）技术的最大区别是，OQM技术提出直接基于对象构造各种增删查改语句。

其中，最核心的功能就是**通过查询对象构建查询子句**，这也是OQM名字中Q的由来。

OQM技术中另一个重要的发现就是**查询对象的字段名称和查询子句的查询条件可以相互转换**。

例如SQL中的查询条件`age > ?`，我们只需要将`>`改写为字符串别名`Gt`（Greater-Than的缩写），再将Gt作为后缀附在列名age后得到`ageGt`，即可用作字段的名称。

然后我们使用OQM技术对这样的对象进行解析，在遇到名称为`ageGt`的字段被赋值时，只需要逆向进行上述过程就能得到对应的查询条件。

这样，我们只需要创建一个实体对象和一个查询对象，通过实体对象来确定表名和列名，通过在查询对象中定义字段和赋值来控制查询子句的构造。再将各种样板代码进行封装后，为单张表构建增删查改接口的代码就只剩下两行了。

### Demo介绍

我们单独开发了一个[demo](https://github.com/doytowin/goooqo-demo)来演示[GoooQo](https://github.com/doytowin/goooqo)的这项功能。

其中，[test.db](https://github.com/doytowin/goooqo-demo/blob/main/test.db) 是一个 sqlite 数据库，包含一个有4行数据的表`t_user`。

| id | username | password | email        | mobile      | nickname | memo | valid |
|----|----------|----------|--------------|-------------|----------|------|-------|
| 1  | f0rb     | 123456   | f0rb@163.com | 18888888881 | test1    |      | 1     |
| 2  | user2    | 123456   | test2@qq.com | 18888888882 | test2    |      | 0     |
| 3  | user3    | 123456   | test3@qq.com | 18888888883 | test3    | memo | 1     |
| 4  | user4    | 123456   | test4@qq.com | 18888888884 | test4    |      | 1     |

接着我们在 [user.go](https://github.com/doytowin/goooqo-demo/blob/main/user.go) 中为user表定义两个结构体 `UserEntity` 和 `UserQuery`。

`UserEntity`就是一个传统的实体对象，字段对应上述表中的列名。

`UserQuery`中的字段按照列名加后缀的格式定义，以便生成对应的查询语句（字段后缀请参考文末的后缀映射表）：
```go
type UserQuery struct {
    goooqo.PageQuery
    IdGt         *int     // id > ?
    IdIn         *[]int   // id IN (?,?,?)
    EmailContain *string  // email LIKE "%value%"
    MemoNull     *bool    // memo is [NOT] NULL
}
```

此外，嵌入的`goooqo.PageQuery`中定义了`PageNumber`、`PageSize`、`Sort`3个字段，用于构造分页子句和排序子句。

接下来，在main方法里建立好数据库连接后，我们即可使用下面两行代码来为user表构建一个增删查改接口：

```go
userDataAccess := rdb.NewTxDataAccess[UserEntity](tm)
goooqo.BuildRestService[UserEntity, UserQuery]("/user/", userDataAccess)
```

其中，第一行代码为user表创建一个数据库访问模块，第二行代码使用创建的数据库访问模块为user表创建一个web访问模块。

这个web访问模块负责处理 `http://localhost:9090/user/` 上的访问请求。

### Demo运行

把[demo](https://github.com/doytowin/goooqo-demo)仓库的代码克隆到本地，运行[demo.go](https://github.com/doytowin/goooqo-demo/blob/main/demo.go)文件里的main方法启动程序，然后访问以下地址即可查看请求结果:
  - http://localhost:9090/user/
  - http://localhost:9090/user/3
  - http://localhost:9090/user/?pageNumber=2&pageSize=2
  - http://localhost:9090/user/?sort=id,desc
  - http://localhost:9090/user/?sort=memo,desc%3Bemail,desc
  - http://localhost:9090/user/?idIn=1,3
  - http://localhost:9090/user/?emailContain=qq
  - http://localhost:9090/user/?emailContain=qq&IdGt=2
  - http://localhost:9090/user/?emailContain=qq&memoNull=true
  - http://localhost:9090/user/?emailContain=qq&memoNull=false

对于查询参数`?emailContain=qq&memoNull=false`，将会生成SQL语句`SELECT * FROM t_user WHERE email LIKE '%qq' AND memo IS NOT NULL`，对应的返回结果为：

```json
{
  "data": {
    "list": [
      {
        "id": 3,
        "username": "user3",
        "email": "test3@qq.com",
        "mobile": "18888888883",
        "nickname": "test3",
        "memo": "memo",
        "valid": true
      }
    ],
    "total": 1
  },
  "success": true
}
```

我们还可以使用[user.http](https://github.com/doytowin/goooqo-demo/blob/main/user.http)文件里提供的HTTP请求来测试 `POST/PUT/PATCH/DELETE` 等接口.
例如：

```http
### 添加新的用户
POST http://localhost:9090/user/
Content-Type: application/json

[{
  "username": "Ada Wong",
  "email": "AdaW@gmail.com",
  "mobile": "01066666",
  "nickname": "ada",
  "memo": "An agent.",
  "valid": true
}, {
  "username": "Leon Kennedy",
  "email": "LeonKennedy@gmail.com",
  "mobile": "01077777",
  "nickname": "leon",
  "memo": "The hero.",
  "valid": true
}]


### 为id为3的用户更新指定字段

PATCH http://localhost:9090/user/3
Content-Type: application/json

{
  "username": "Leon Kennedy",
  "email": "LeonKennedy@gmail.com"
}

### 删除memo字段为NULL的用户
DELETE http://localhost:9090/user/?memoNull=false
```

### 后记

[GoooQo](https://github.com/doytowin/goooqo)是OQM技术在继Java版框架[DoytoQuery](http://github.com/doytowin/doyto-query)之后的Go语言版本，现在还只是一个得到初步验证的MVP，后续还需多轮迭代才能实现OQM技术提出的所有特性。请大家保持关注，多多支持！

### 附录：后缀映射表

当前版本支持的后缀以及对应的查询条件可以参考这张后缀映射表：

| 后缀名称       | 字段名称           | 字段赋值    | SQL查询条件               | MongoDB查询条件                            |
|------------|----------------|---------|-----------------------|----------------------------------------|
| (EMPTY)    | id             | 5       | id = 5                | {"id":5}                               |
| Eq         | idEq           | 5       | id = 5                | {"idEq":5}                             |
| Not        | idNot          | 5       | id != 5               | {"idNot":{"$ne":5}}                    |
| Ne         | idNe           | 5       | id <> 5               | {"idNe":{"$ne":5}}                     |
| Gt         | idGt           | 5       | id > 5                | {"idGt":{"$gt":5}}                     |
| Ge         | idGe           | 5       | id >= 5               | {"idGe":{"$gte":5}}                    |
| Lt         | idLt           | 5       | id < 5                | {"idLt":{"$lt":5}}                     |
| Le         | idLe           | 5       | id <= 5               | {"idLe":{"$lte":5}}                    |
| NotIn      | idNotIn        | [1,2,3] | id NOT IN (1,2,3)     | {"id":{"$nin":[1, 2, 3]}}              |
| In         | idIn           | [1,2,3] | id IN (1,2,3)         | {"id":{"$in":[1, 2, 3]}}               |
| Null       | memoNull       | false   | memo IS NOT NULL      | {"memo":{"\$not":{"$type", 10}}}       |
| Null       | memoNull       | true    | memo IS NULL          | {"memo":{"$type", 10}}                 |
| NotLike    | nameNotLike    | "arg"   | name NOT LIKE '%arg%' | {"name":{"\$not":{"$regex":"arg"}}}    |
| Like       | nameLike       | "arg"   | name LIKE '%arg%'     | {"name":{"$regex":"arg"}}              |
| NotStart   | nameNotStart   | "arg"   | name NOT LIKE 'arg%'  | {"name":{"\$not":{"$regex":"^arg"}}}   |
| Start      | nameStart      | "arg"   | name LIKE 'arg%'      | {"name":{"$regex":"^arg"}}             |
| NotEnd     | nameNotEnd     | "arg"   | name NOT LIKE '%arg'  | {"name":{"\$not":{"\$regex":"arg\$"}}} |
| End        | nameEnd        | "arg"   | name LIKE '%arg'      | {"name":{"\$regex":"arg\$"}}           |
| NotContain | nameNotContain | "arg"   | name NOT LIKE '%arg%’ | {"name":{"\$not":{"$regex":"arg"}}}    |
| Contain    | nameContain    | "arg"   | name LIKE '%arg%’     | {"name":{"$regex":"arg"}}              |
| Rx         | nameRx         | "arg\d" | name REGEXP 'arg\d’   | {"name":{"$regex":"arg\d"}}            |

---
© 2024 Yuan Zhen. All rights reserved. 

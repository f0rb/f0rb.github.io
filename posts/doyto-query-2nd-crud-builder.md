---
title: 'DoytoQuery系列二：增删查改语句生成'
date: 2020-04-23 04:10:05
tags: [f0rb说,DoytoQuery,Java,CRUD]
published: true
hideInList: true
feature: 
isTop: false
---
## Entity对象的映射
通过Query对象生成WHERE语句后，已经具备了对单表的查询能力，接下来我们完善一下对单表的增删查改操作，考虑引入Entity对象。


## 增删改语句生成[CrudBuilder](https://github.com/f0rb/doyto-query/blob/master/src/main/java/win/doyto/query/core/CrudBuilder.java)


#### 查询语句
对于如下Entity对象
```
@Table(name = "t_user")
public class UserEntity {
    private Integer id;
    private String username;
    private String password;
    private String mobile;
    private String email;
    private Date createTime;
}
```
- 把UserEntity的字段全部列出来，得到表列
```
id, username, password, mobile, email, createTime
```
- 从@Table注解里得到表名
```
t_user
```
- 代入到SELECT [columns] FROM [table]，得到
```
SELECT id, username, password, mobile, email, createTime FROM t_user
```
- 再与UserQuery生成的WHERE语句拼接，即可得到对应的查询语句，如：
```
SELECT id, username, password, mobile, email, createTime FROM t_user 
WHERE username LIKE ？LIMIT 10 OFFSET 0
```

#### 插入语句
将表名和列名代入INSERT INTO [table] [columns] VALUES (?[, ?])得到
```
INSERT INTO t_user (id, username, password, mobile, email, createTime) 
VALUES (?, ?, ?, ?, ?, ?)
```

#### 更新语句
将表名和列名代入UPDATE [table] SET $column = ?[, $column = ?] WHERE id = ?，得到
```
UPDATE t_user SET username = ?, password = ?, mobile = ?, email = ?, createTime = ? 
WHERE id = ?
```
_更新语句可以更新实体的所有字段，或者只更新非空字段_

#### 删除语句
将表名代入DELETE FROM [table] WHERE id = ?，得到
```
DELETE FROM t_user WHERE id = ?
```

#### 批量插入语句
```
INSERT INTO t_user (id, username, password, mobile, email, createTime) 
VALUES (?, ?, ?, ?, ?, ?),  (?, ?, ?, ?, ?, ?)
```

#### WHERE拼接
然后UPDATE和DELETE语句也可以和WHERE语句拼接，如
```
UPDATE t_user SET valid = ? WHERE username = ? LIMIT 10
DELETE FROM t_user WHERE username LIKE ? LIMIT 5
```

### SQL语句执行
根据Entity类和Query类生成SQL语句和参数列表后，就可以交给JDBC执行了
```
    public List<E> query(Q query) {
        SqlAndArgs sqlAndArgs = crudBuilder.buildSelectColumnsAndArgs(q, "*");
        return jdbcOperations.query(sqlAndArgs.sql, sqlAndArgs.args);
    }

    public final int update(E e) {
        return doUpdate(crudBuilder.buildUpdateAndArgs(e));
    }

    public final int patch(E e) {
        return doUpdate(crudBuilder.buildPatchAndArgsWithId(e));
    }

    public final int patch(E e, Q q) {
        return doUpdate(crudBuilder.buildPatchAndArgsWithQuery(e, q));
    }

    private int doUpdate(SqlAndArgs sqlAndArgs) {
        return jdbcOperations.update(sqlAndArgs.sql, sqlAndArgs.args);
    }
```

## 小结
可以看到，通过解析Query对象和Entity对象，我们已经可以生成针对单表访问的增删查改等基本SQL语句并执行数据库操作，然后我们就可以在这个基础上抽象出一个通用的数据访问接口——[DataAccess](https://github.com/f0rb/doyto-query/blob/master/src/main/java/win/doyto/query/core/DataAccess.java)。



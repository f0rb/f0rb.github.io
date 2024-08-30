---
title: 'DoytoQuery中的查询映射方案'
date: 2022-12-04 20:44:29
tags: [DoytoQuery,Java,CRUD,f0rb说]
published: false
hideInList: true
feature: /post-images/the-query-mapping-solution-in-doyto-query-zh.png
isTop: false
---
![DoytoQuery中的4种字段映射方式](https://blog.doyto.win/post-images/1670157991801.png)

## 引言

DoytoQuery作为一个ORM框架，在研发过程中积累了很多全新的思路和解决方案，其中一个核心方案就是将查询对象的字段映射为SQL语句中的WHERE条件进行动态查询，目前支持四种字段映射方式。

让我们看看如何为以下`UserEntity`编写查询对象：

```java
@Getter
@Setter
@Entity
public class UserEntity extends AbstractPersistable<Integer> {
    private String username;
    private String email;
    private String mobile;
    private String password;
    private String nickname;
    private Boolean valid;
}
```

##  四种字段映射方式详解

### 1. 后缀映射

先定义一个类`UserQuery`, 对于一些常见的查询条件，比如`id = ?`, `username = ?` and `valid = ?`等，我们只需要在`UserQuery`类中定义如下字段即可。

```java
@Getter
@Setter
@SuperBuilder
@NoArgsConstructor
@AllArgsConstructor
public class UserQuery extends PageQuery {
    private Integer id;
    private String username;
    private Boolean valid;
}
```

像其他一些常用的比较符号. `[NOT] LIKE`, `>`, `<`, `!=`, `>=`, `<=`, `IS [NOT] NULL`, `[NOT] IN`, 我们需要在列名后加一些特定的后缀作为字段名称来标识对应的条件符号。比如，将`idIn`映射为`id IN (?,?,?)`, 将`idGt`映射为`id > ?`, 将`usernameLike`映射为`username LIKE ?`, 还需要定义好对应的类型。

```java
public class UserQuery extends PageQuery {
    private Integer id;
    private String username;
    private Boolean valid;
    private List<Integer> idIn;
    private Integer idGt;
    private String usernameLike;
}
```

只有值不为NULL的字段才会被映射，并且多个条件由`AND`连接。

```java
UserQuery userQuery = UserQuery.builder().usernameLike("test").valid(true).build();
```  
这个`userQuery`会被映射为如下查询语句以及对应的参数。
```sql
SELECT id, username, email, mobile, password, nickname, valid FROM user WHERE username LIKE ? and valid = ?
-- With parameters: ["%test%", true] 
```
所有支持的后缀详见附录I。

### 2. OR映射

`OR`语句也是SQL里一种常用的查询方式，DoytoQuery支持的方式有两种。

2.1 将含`Or`的字段映射为`OR`语句

像`usernameOrEmailOrMobile`这样多个字段由`Or`连接的字段会被映射为`username = ? OR email = ? OR mobile = ?`。

2.2 将实现了`Or`的对象映射为`OR`语句

```java
public interface Or {
}

public class AccountOr implements Or {
    private String username;
    private String email;
    private String mobile;
}

public class UserQuery {
    private AccountOr account;
}
```

遍历`AccountOr`中的字段可以得到`[username, email, mobile]`，然后使用关键字`OR`相连而得`username = ? OR email = ? OR mobile = ?`.

### 3. 嵌套查询映射

嵌套查询是SQL的另一个常用特性。在管理菜单树的`menu`表的常见定义中，我们通常定义一个外键`parentId`来引向父菜单实体的`id`。

要查询拥有子菜单的父菜单，我们可以执行以下SQL：
```sql
SELECT * FROM menu WHERE id IN (SELECT parentId FROM menu WHERE ...)
```

要查询某些菜单的子菜单，我们可以执行以下SQL：
```sql
SELECT * FROM menu WHERE parentId IN (SELECT id FROM menu WHERE ...)
```

这条语句用于查询指定用户拥有的菜单：
```sql
SELECT * FROM menu WHERE id IN (
    SELECT menu_id FROM a_perm_and_menu WHERE perm_id IN (
        SELECT perm_id FROM a_role_and_perm WHERE role_id IN (
            SELECT role_id FROM a_user_and_role WHERE user_id IN (
                SELECT id FROM t_user WHERE id = ?
))))
```

这些都是ERM模型里常见的一对多/多对一/多对多关系. DoytoQuery定义了一个新的`@DomainPath`注解来映射这种嵌套查询。

```java
@Target(FIELD)
@Retention(RUNTIME)
public @interface DomainPath {
    // To describe how to route from the host domain to the target domain.
    String[] value();
    String localField() default "id";
    String foreignField() default "id";
}
```

这里有一个`@DomainPath`注解用法介绍。

```java
public class MenuQuery extends PageQuery {
    // 多对一: 根据父菜单的条件查询他们的子菜单
    @DomainPath(localField = "parentId", foreignField = "id", value = "menu")
    MenuQuery parent;      // parentId   IN (SELECT      id   FROM     menu  WHERE ... )
    // 一对多: 根据子菜单的条件查询父菜单
    @DomainPath(localField = "id", foreignField = "parentId", value = "menu")
    MenuQuery children;    // id   IN (SELECT      parentId   FROM     menu  WHERE ... )

    // 多对多: 查询满足条件的用户被授予的菜单
    @DomainPath({"menu", "~", "perm",  "~", "role", "~", "user"})
    UserQuery user;
}
```

### 4. 直接映射

当上述方法都不使用时，还有最后一种将字段映射到条件的方法，就是使用注解`@QueryField`定义查询条件后直接映射。这是DoytoQuery创建时的第一种映射方式，但是是最后一种推荐的方式，用法如下：
```java
public class MenuQuery extends PageQuery {
    @QueryField(and = "id = (SELECT parent_id FROM menu WHERE id = ?)")
    private Integer childId;
}
```
嵌套查询的这种映射方式在`@DomainPath`发明后就已弃用。

## 总结

在本文中，我们介绍了DoytoQuery中的四种字段映射方式。前三种方式不需要编写任何SQL语句，并且可以涵盖到关系数据库开发中大部分涉及单表查询的场景。


## 附录I: 后缀支持列表

| 后缀名称                 | 比较符号     | 占位符                                                        | 类型        | 值处理           |
|-------------------------|-------------|------------------------------------------------------------|------------|------------------|
| (No matching<br>suffix) | =           | ？                                                          |            |                  |
| Not                     | !=          | ?                                                          |            |                  |
| NotLike                 | NOT LIKE    | ?                                                          | String     | %value%          |
| Like                    | LIKE        | ?                                                          | String     | %value%          |
| Start                   | LIKE        | ?                                                          | String     | %value           |
| End                     | LIKE        | ?                                                          | String     | value%           |
| NotIn                   | NOT IN      | <p>非空集合: (?[, ?]);<br>空集合则忽略</p> | Collection |                  |
| In                      | IN          | <p>非空集合(?[, ?]);<br>空集合: (null)</p> | Collection |                  |
| NotNull                 | IS NOT NULL | -                                                          | boolean    |                  |
| Null                    | IS NULL     | -                                                          | boolean    |                  |
| Gt                      | >           | ?                                                          |            |                  |
| Ge                      | >=          | ?                                                          |            |                  |
| Lt                      | <           | ?                                                          |            |                  |
| Le                      | <=          | ?                                                          |            |                  |
| Eq                      | =           | ?                                                          |            |                  |

 
 
 
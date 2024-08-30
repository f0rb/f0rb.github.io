---
title: 'The Query Mapping Solution in DoytoQuery'
date: 2022-12-02 22:07:03
tags: [f0rb说,DoytoQuery,Java,CRUD]
published: false
hideInList: true
feature: /post-images/the-query-mapping-solution-in-doyto-query.png
isTop: false
---
![](https://blog.doyto.win/post-images/1669993942608.png)

## Introduction

As an ORM framework, there are many new ideas and solutions in DoytoQuery. The core idea of DoytoQuery is to map the query object to WHERE clause in a SQL statement for dynamic query, and there are four ways to map the fields in DoytoQuery.

Let's see how to write a query object for the following `UserEntity`:

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

##  The four fields mapping ways

### 1. Suffix mapping

We define a class `UserQuery` first, and for common conditions like `id = ?`, `username = ?` and `valid = ?`, we just define the fields in `UserQuery` class as follows.

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

For other comparison operators e.g. `[NOT] LIKE`, `>`, `<`, `!=`, `>=`, `<=`, `IS [NOT] NULL`, `[NOT] IN`, we can add a suffix after the column name to specify the comparator. For examples, map `idIn` to `id IN (?,?,?)`, `idGt` to `id > ?`, `usernameLike` to `username LIKE ?`, and also with the right type as following.

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

Only non-null fields will be mapped and mutliple condiitions will be connected by `AND`. 

```java
UserQuery userQuery = UserQuery.builder().usernameLike("test").valid(true).build();
```  
The above `userQuery` object will be mapped to the following SQL with corresponding parameters.
```sql
SELECT id, username, email, mobile, password, nickname, valid FROM user WHERE username LIKE ? and valid = ?
-- With parameters: ["%test%", true] 
```

Check Appendix I for all supported suffixes.
   

### 2. OR Mapping 

We also need to connect the conditions by `OR` in some scenarios and there are two ways in DoytoQuery.

2.1 Map `OR` clause by field name containing `Or`

Connect multi column names with `Or` like `usernameOrEmailOrMobile`, and map it to `username = ? OR email = ? OR mobile = ?`.

2.2 Map `OR` clause for objects marked by `Or` interface

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

Traverse the fields in `AccountOr` to get `[username, email, mobile]` and then connect them with `OR` and get `username = ? OR email = ? OR mobile = ?`.

### 3. Nested Query Mapping

The nested query is another frequently used feature of SQL. In a common definititon of a `menu` table to manage the menu tree, we usually define a foreign key `parentId` to refer to the `id` of the parent menu entity. 

To query the menus which have children menus, we execute the following SQL:
```sql
SELECT * FROM menu WHERE id IN (SELECT parentId FROM menu)
```

To query the children menus of the specified menu, we execute the following SQL:
```sql
SELECT * FROM menu WHERE parentId IN (SELECT id FROM menu)
```

And query the menus onwed by certain users:
```sql
SELECT * FROM menu WHERE id IN (
  SELECT menu_id FROM a_perm_and_menu WHERE perm_id IN (
    SELECT perm_id FROM a_role_and_perm WHERE role_id IN (
      SELECT role_id FROM a_user_and_role WHERE user_id IN (
        SELECT id FROM t_user WHERE id = ?
))))
```

These are the typical one-to-many/many-to-one/many-to-many relationships in ERM. DoytoQuery defines a new annotation `@DomainPath` to map the nested query.

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

Here are the full examples about the usages of `@DomainPath`.

```java
public class MenuQuery extends PageQuery {
    // many-to-one: query children menus by their parents' conditions
    @DomainPath(localField = "parentId", foreignField = "id", value = "menu")
    MenuQuery parent;      // parentId   IN (SELECT      id   FROM     menu  WHERE ... )
    // one-to-many: query parent menus by their children's conditions
    @DomainPath(localField = "id", foreignField = "parentId", value = "menu")
    MenuQuery children;    // id   IN (SELECT      parentId   FROM     menu  WHERE ... )
    
    // Many-to-many: query menus owned by the users meeting the query conditions
    @DomainPath({"menu", "~", "perm",  "~", "role", "~", "user"})
    UserQuery user;
}
```

### 4. Mapping explicitly

The last way to map the field to the condition when no ways matching above is to use the annotation `@QueryField`, it will map the clause defined by the annotation directly. It is the first mapping way when DoytoQuery is created, but the last one to use now. Here is an example.
```java
public class MenuQuery extends PageQuery {
    @QueryField(and = "id = (SELECT parent_id FROM menu WHERE id = ?)")
    private Integer childId;
}
```
This way is deprecated for nested query after the `@DomainPath` is created.
 
## Conclution

In this article, we explored four ways of mapping fields of a query object to the SQL conditions with DoytoQuery. You don't need write any SQL with the first three ways and they can cover most of the single-table query scenarios when working with relational databases.





## Appendix I: Supported Mapping Suffix


|      Suffix             | Operator         | PlaceHolder                                 | Type Restriction    | Value Conversion     |
| ----------------------- | ----------- | -------------------------------------- | ---------- | ------- |
| (No matching<br>suffix) | =           | ？                                      |       -     |     -    |
| Not     | !=          | ?                                      |            |         |
| NotLike | NOT LIKE    | ?                                      | String     | %value% |
| Like    | LIKE        | ?                                      | String     | %value% |
| Start   | LIKE        | ?                                      | String     | %value  |
| End     | LIKE        | ?                                      | String     | value%  |
| NotIn   | NOT IN      | <p>(?[, ?]) for nonempty set;<br>Bypass for emply set.</p>     | Collection |         |
| In      | IN          | <p>(?[, ?]) for nonempty set;<br>(null) for emply set.</p> | Collection |         |
| NotNull | IS NOT NULL | -                                      | boolean    |         |
| Null    | IS NULL     | -                                      | boolean    |         |
| Gt      | >           | ?                                      |            |         |
| Ge      | >=          | ?                                      |            |         |
| Lt      | <           | ?                                      |            |         |
| Le      | <=          | ?                                      |            |         |
| Eq      | =           | ?                                      |            |         |

   
 
 
 
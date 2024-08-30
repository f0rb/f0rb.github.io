---
title: '关于查询对象字段映射的一篇小结'
date: 2022-03-04 20:55:04
tags: [f0rb说,DoytoQuery,Java,CRUD]
published: false
hideInList: true
feature: 
isTop: true
---
第二代ORM理论第一条：

**我们可以定义一个查询对象用于映射查询语句**，

具体分为以下五种情形：

1. 通用注解

   [《SpringDataJPA之从入门到弃用》](https://blog.doyto.win/post/spring-data-jpa-from-start-to-deprecated/)

2. 后缀映射

   [《基于查询对象字段的后缀推导》](https://blog.doyto.win/post/suffix-deduction/)

3. 子查询

   [《子查询的几种映射方法》](https://blog.doyto.win/post/nested-query/)
   
4. OR查询

   [《OR语句的三种映射方法》](https://blog.doyto.win/post/or-query/)
   
5. 分页和排序

   [《数据库查询中的分页和排序方案》](https://blog.doyto.win/post/paging-and-sorting/)
   
## 示例

定义一个`UserQuery`类用于`t_user`表的数据查询：

```java
public class AccountOr implements Or {
    private String username;
    private String email;
    private String mobile;
}

public class UserQuery extends PageQuery {

    private AccountOr accountOr;
    private Boolean valid;
    private boolean mobileNotNull;
   
    @NestedQueries({
            @NestedQuery(select = "user_id", from = "t_user_and_role"),
            @NestedQuery(select = "role_id", from = "t_role_and_perm", where = "perm_id"),
            @NestedQuery(select = "id", from = "t_perm")
    })
    private String permNameLike;

    public void setAccount(String account) {
        this.accountOr = AccountOr.builder().username(account).email(account).mobile(account).build();
    }
}
```

前端请求的参数为：
```
http://localhost:8080/user/?account=test&valid=true&permNameLike=list&mobileNotNull=true&sort=id,desc;create_time,asc&pageSize=10&pageNumber=5
```

基于`UserQuery`字段推导的SQL语句为：
```sql
SELECT * FROM t_user
WHERE (username = ? OR email = ? OR mobile = ?)
  AND valid = ?
  AND mobile IS NOT NULL
  AND id IN (
      SELECT user_id FROM t_user_and_role WHERE role_id IN (
          SELECT role_id FROM t_role_and_perm WHERE perm_id IN (
              SELECT id FROM t_perm WHERE perm_name LIKE ?))
  )
ORDER BY id desc, create_time asc
LIMIT 10 OFFSET 40
```
还有参数：
```
["test", "test", "test", true, "%list%"]
```
> [测试用例](https://github.com/doytowin/doyto-query-demo/blob/main/src/test/java/win/doyto/query/demo/module/user/UserMvcTest.java#L38)

于是，我们在数据访问层定义以下接口用于单表数据的动态查询：
```java
public interface DataAccess<E extends Persistable<I>, I extends Serializable, Q> {
    List<E> query(Q query);
    long count(Q query);
}
```
> 详见：[github](https://github.com/doytowin/doyto-query/blob/main/doyto-query-api/src/main/java/win/doyto/query/core/DataAccess.java)


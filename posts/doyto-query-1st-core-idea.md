---
title: 'DoytoQuery系列一：核心理念之WHERE语句生成'
date: 2020-04-20 23:16:12
tags: [f0rb说,DoytoQuery,Java,CRUD]
published: true
hideInList: true
feature: 
isTop: false
---
### 先说结论
**SQL查询的WHERE语句是可以通过对象生成的**

---
### 接着论证
- 先定义这样一个对象, 假设作如下赋值
```
@Getter
@Setter
@Table(name = "t_user")
public class UserQuery {
     private Integer pageNumber;                // 3
     private Integer pageSize;                  // 10
     private String sort;                       // id,desc;create_time,asc

     private String usernameOrEmailOrMobile;    // admin
     private String nicknameLike;               // 管理员
     private Boolean valid;                     // true
 }
```
- 对于usernameOrEmailOrMobile，按Or拆开后分别接上" = ?"，再用" OR "拼接，得到
```
username = ? OR email = ? OR mobile = ?
```
- 对于nicknameLike，去掉后缀Like，拼接" LIKE ?"，得到
```
nickame LIKE ?
```
- 对于valid，字段名加上"= ?"得到
```
valid = ?
```
- 将上述3条语句用" AND "拼接，含OR的语句加括号，再在前面加上"WHERE"，得到
```
WHERE (username = ? OR email = ? OR mobile = ?) AND nickname LIKE ? AND valid = ?
```
- 对于sort字段，逗号替换为空格，分号替换为逗号，前面加上"ORDER BY "，得到
```
ORDER BY id DESC, createTime ASC
```
- 将pageNumber和pageSize的值代入LIMIT ${pageSize} OFFSET ${(pageNumber - 1) * pageSize}，得到
```
LIMIT 10 OFFSET 20
```
- 将上述SELECT, WHERE, ORDER BY, LIMIT语句合并，得到
```
WHERE (username = ? OR email = ? OR mobile = ?) AND nickname LIKE ? AND valid = ?
ORDER BY id DESC, createTime ASC
LIMIT 10 OFFSET 20
```
- 同时得到对应的参数列表
```
["admin", "admin", "admin", "%管理员%", true]
```
_其中WHERE语句的过滤条件根据对应字段是否赋值来决定是否拼接_
- 再从@Table注解里得到表名，代入SELECT * FROM [TABLE]，得到
```
SELECT * FROM t_user
```
- 再同上述WHERE语句拼接，即可得到一条完整的查询语句
```
SELECT * FROM t_user
WHERE (username = ? OR email = ? OR mobile = ?) AND nickname LIKE ? AND valid = ?
ORDER BY id DESC, createTime ASC
LIMIT 10 OFFSET 20
```
证毕。

---
### 扩展
每条SELECT语句都需要支持分页和排序，所以可以将pageNumber，pageSize，sort提取为父类
```
public class PageQuery {
    private Integer pageNumber;
    private Integer pageSize;
    private String sort;
}

@Table(name = "t_user")
public class UserQuery extends PageQuery {
    private String usernameOrEmailOrMobile;
    private String nicknameLike;
    private Boolean valid;
}
```

---
### 查询构造器[QueryBuilder](https://github.com/f0rb/doyto-query/blob/master/src/main/java/win/doyto/query/core/QueryBuilder.java)

基于上述分析，我们就可以编写出测试用例，通过测试用例驱动出一个自动化的查询语句构造器


---
### [单元测试用例](https://github.com/f0rb/doyto-query/blob/master/src/test/java/win/doyto/query/core/QueryBuilderTest.java)
```
    @Test
    public void buildSelect() {
        TestQuery testQuery = TestQuery.builder().build();
        assertEquals("SELECT * FROM user", testQueryBuilder.buildSelectAndArgs(testQuery, argList));
    }
```
```
    @Test
    public void buildSelectWithWhere() {
        TestQuery testQuery = TestQuery.builder().username("test").build();
        assertEquals("SELECT * FROM user WHERE username = ?", testQueryBuilder.buildSelectAndArgs(testQuery, argList));
    }
```
```
    @Test
    public void buildSelectWithWhereAndPage() {
        TestQuery testQuery = TestQuery.builder().username("test").build();
        testQuery.setPageNumber(3).setPageSize(10);
        assertEquals("SELECT * FROM user WHERE username = ? LIMIT 10 OFFSET 30",
                     testQueryBuilder.buildSelectAndArgs(testQuery, argList));
    }
```
```
    @Test
    public void buildSelectWithArgs() {
        argList = new ArrayList<>();
        TestQuery testQuery = TestQuery.builder().username("test").build();
        assertEquals("SELECT * FROM user WHERE username = ?",
                     testQueryBuilder.buildSelectAndArgs(testQuery, argList));
        assertEquals(1, argList.size());
        assertEquals("test", argList.get(0));
    }
```

---
### Github地址
[doyto-query](https://github.com/f0rb/doyto-query/)




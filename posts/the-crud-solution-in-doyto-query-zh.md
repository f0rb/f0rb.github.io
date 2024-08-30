---
title: 'DoytoQuery中关于增删改查接口的增强方法'
date: 2023-04-14 21:46:41
tags: [DoytoQuery,Java,CRUD,f0rb说]
published: true
hideInList: true
feature: 
isTop: false
---
## 背景

目前，大部分成熟的ORM框架都为数据库单表操作封装了增删查改接口，在接口调用时会自动生成对应的SQL语句并执行数据库操作。

DoytoQuery框架在实现这些功能的同时，对于需要对数据进行动态筛选的更新和删除的接口，基于《DoytoQuery中的查询映射方案》进行了增强，并大幅简化了其参数的构造工作。

## ORM框架中的增删查改接口及典型实现

我们依然使用以下实体对象作为示范：

```java
@Getter
@Setter
@Entity
@Table(name = "user")
public class UserEntity extends AbstractPersistable<Integer> {
    private String username;
    private String password;
    private String mobile;
    private String email;
    private String nickname;
    private Boolean valid;
}
```

以SpringDataJPA中的`JpaRepository`为例，基础的增删查改接口定义如下：

```java
@NoRepositoryBean
public interface JpaRepository<T, ID> extends ListCrudRepository<T, ID>, ListPagingAndSortingRepository<T, ID>, QueryByExampleExecutor<T> {

    Optional<T> findById(ID id);
  
    <S extends T> S save(S entity);

    void deleteById(ID id);
    
    //...
}
```

这类接口的每个方法在被调用后，围绕SQL语句大致都会执行以下两大步骤：
1. **生成SQL语句**
2. **执行SQL语句**

当使用SpringDataJPA时，我们会为UserEntity定义以下Repository：
```java
public interface UserRepository extends JpaRepository<UserEntity, Integer> {
}
```

当调用`UserRepository#findById`时，会根据`UserEntity`类中定义的字段和注释的表名生成一下SQL语句然后执行：
```sql
SELECT id, username, password, mobile, email, nickname, valid FROM user WHERE id = ?
```

对于`save`方法，当保存的`User`为新实体时，会执行一条`Insert`的语句：
```sql
INSERT INTO t_user (id, username, password, mobile, email, nickname, valid) VALUES (?, ?, ?, ?, ?, ?)
```

而当实体已存在，也就是当`User`对象中赋有对应的`id`时，会执行以下更新语句：
```sql
UPDATE t_user SET username = ?, password = ?, mobile = ?, email = ?, nickname = ?, valid = ? WHERE id = ?
```

而`deleteById`方法则会对应一条删除语句：
```sql
DELETE FROM t_user WHERE id = ?;
```
其他的ORM框架如Hibernate、MyBatis/Mybatis-Generator、MyBatis-Plus、jOOQ等，也都提供了类似的接口。

## DoytoQuery中的数据操作接口

作为一款持久层框架，DoytoQuery中的单表操作接口被命名为`DataAccess`，定义如下：

```java
public interface DataAccess<E extends Persistable<I>, I extends Serializable, Q> {
    E get(I id);
    void create(E e);
    int delete(I id);
    int update(E e);
    int patch(E e);
    
    List<E> query(Q query);
    long count(Q query);
    int patch(E e, Q q);
    int delete(Q query);
}
```
其中`get(I)/create(E)/delete(I)/update(E)/patch(E)`等基本的增删查改接口跟其他框架提供的功能基本一致，除了更新方法`update`和`patch`方法略有区别，`update`方法会更新对象的所有字段，而`patch`方法仅更新非`null`的字段。

## DoytoQuery中的增强接口

在进行动态条件查询时，大部分ORM框架都需要借助一个中间对象来生成条件语句，比如Hibernate中的`Criteria`，SpringDataJPA中的`Example`，`Specification`，MybatisPlus中的`Wrapper`等，这就要求框架的使用者先根据已有参数构造出这个中间对象，再作为参数调用对应的方法，从而增加的代码的开发量和程序的复杂度，不利于代码的复用和优化。

而DoytoQuery基于其独创的查询对象字段映射方案，可以直接从用户定义的Query对象推导出`WHERE`语句，不需要借助任何中间对象，从而大大简化了参数的构造，以及接口的定义和调用。

以下接口支持在查询、更新和删除时，通过传入一个Query对象来进行数据的筛选：
- List\<E\> query(Q query);
- long count(Q query);
- int patch(E e, Q q);
- int delete(Q query);
  
首先，我们定义一个UserQuery类如下：
```java
@Getter
@Setter
@SuperBuilder
@NoArgsConstructor
@AllArgsConstructor
public class UserQuery extends PageQuery {
    private String emailEnd;
    private Boolean valid;
}
```
  
然后创建一个新的对象并进行赋值：
```java
UserQuery userQuery = UserQuery.builder().emailEnd("@163.com").valid(true).build();
```

当调用方法`query(Q)`时，对应执行的SQL以及参数为：
```sql
SELECT id, username, password, mobile, email, nickname, valid FROM t_user WHERE email LIKE ? AND valid = ?
- 参数: "%@163.com", true
```  

`count(Q)`则是用于配合`query(Q)`执行分页查询的，当被调用时，执行的SQL为：
```sql
SELECT count(*) FROM t_user WHERE email LIKE ? AND valid = ?
```

当调用方法`patch(E, Q)`时，假设只将`UserEntity`对象的`valid`字段设为了`false`，会执行：
```sql
UPDATE t_user SET valid = ? WHERE id IN (SELECT id FROM t_user t WHERE email LIKE ? AND valid = ?)
```
> 这里采用子查询的原因是由于某些数据库不支持在`UPDATE`和`DELETE`语句上执行分页操作。


最后在调用方法`delete(Q)`时执行的是：
```sql
DELETE FROM t_user WHERE id IN (SELECT id FROM t_user t WHERE email LIKE ? AND valid = ?)
```

另外，当给`UserQuery`传入分页参数后，对应会为括号中的`SELECT`语句加上分页语句：
```sql
DELETE FROM t_user WHERE id IN (SELECT id FROM t_user t WHERE email LIKE ? AND valid = ? LIMIT 10 OFFSET 10)
```

可以看到，`patch(E, Q)`和`delete(Q)`分别将`UPDATE`语句和`DELETE`语句与复杂的`WHERE`语句结合起来，充分利用了SQL语句的条件过滤特性，以封装简洁的接口，提供 了更加通用的更新和查询功能。

## 结论

以上介绍了DoytoQuery中单表增删查改的解决方案和接口设计，在提供基本的增删查改方法的同时，还提供了更为便利的与动态查询相关的删改查方法，进一步满足了复杂业务场景的需要。

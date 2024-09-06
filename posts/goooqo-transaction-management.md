---
title: 'GoooQo中的事务传播管理'
date: 2024-08-30 21:21:13
tags: [GoooQo,Go,OQM,cn,f0rb说]
published: true
hideInList: false
feature: 
isTop: false
---

## 基本机制

`GoooQo`中事务管理通过[TransactionManager](../core/core.go)和[TransactionContext](../core/core.go)（简称TC）配合完成。

`TransactionManager`中的方法`StartTransaction(ctx context.Context) (TransactionContext, error)`负责开启事务并返回TC；
TC组合了`driver.Tx`，负责事务的提交和回滚。

[TxDataAccess](../core/core.go)接口组合了`DataAccess`和`TransactionManager`，可以在实现数据库访问接口的同时，更方便的进行事务管理。


## 事务使用示例

使用`TransactionManager#StartTransaction`开启事务，手动提交或者回滚事务：
```go
tc, err := userDataAccess.StartTransaction(ctx)
userQuery := UserQuery{ScoreLt: PInt(80)}
cnt, err := userDataAccess.DeleteByQuery(tc, userQuery)
if err != nil {
	err = tc.Rollback()
	return 0
}
err = tc.Commit()
return cnt
```

或者使用`TransactionManager#SubmitTransaction`通过回调的方式提交事务：
```go
err := tm.SubmitTransaction(ctx, func(tc TransactionContext) (err error) {
    // transaction body
    return
})
```

## 事务的传播管理

在Spring的事务传播机制中定义了以下7个级别，`GoooQo`中的对应处理方式如下：

- REQUIRED

  使用任意context调用`TxDataAccess`的`StartTransaction`：

  如果context为TC，则将context强制转化为TC后返回；

  如果context不是TC，则调用`db#BeginTx`开启事务获取`sql.Tx`，再通过context和`sql.Tx`创建TC后返回。

- SUPPORTS

  使用任意`Context`调用`TxDataAccess`的数据库访问方法。

- REQUIRES_NEW

  当`ctx`为TC时，使用`ctx.(TC).Context`开启事务；
  当`ctx`不为TC时，使用`ctx`开启事务；

- NOT_SUPPORTED

  当`ctx`为TC时，使用`ctx.(TC).Context`调用`TxDataAccess`的数据库访问方法；
  当`ctx`不为TC时，使用`ctx`调用`TxDataAccess`的数据库访问方法；

- MANDATORY:

  对传入的`ctx`不是TC的情况进行处理。

- NEVER

  对传入的`ctx`是TC的情况进行处理。 

- NESTED

  使用TC的`SavePoint/RollbackTo`方法。

这里整理了一个表格对前4个传播级别进行了对比：

| context参数\数据库操作 | 开启事务            | 调用数据库           |
|:------------------:|--------------|---------------|
|        任意Context         | REQUIRED          | SUPPORTS          |
| ctx.(TC).Context \| ctx  | REQUIRES_NEW | NOT_SUPPORTED |


## 总结

由于Golang语法限制，无法实现SpringTx提供的基于注解的管理事务传播级别的方法，
但是GoooQo利用Golang的Context机制，使得开发人员能够使用最少的代码来对事务的传播级别进行对应的处理。

---
© 2024 Yuan Zhen. All rights reserved. 

---
title: 'DoytoQuery中的关联查询方案'
date: 2022-12-09 11:27:04
tags: [DoytoQuery,Java,f0rb说,CRUD]
published: true
hideInList: false
feature: /post-images/the-related-query-solution-in-doyto-query-zh.png
isTop: false
---
本文介绍了DoytoQuery对N+1查询问题的解决方案，以及实体间关系的查询方案。
<!-- more -->
![](https://blog.doyto.win/post-images/1670558022938.png)

## 1. 背景
_Java Persistence with Hibernate_ 在12.2.1小节使用如下例子描述 _n+1查询问题_:

```java
List<Item> items = em.createQuery("select i from Item i").getResultList();
// select * from ITEM
for (Item item : items) {
    assertTrue(item.getBids().size() > 0);
    // select * from BID where ITEM_ID = ?
}
```
在这个例子中，每个bids集合的加载都需要执行一条额外的查询语句，当item有N条记录，一共就会执行N+1条查询语句：

```sql
SELECT * FROM item;
SELECT * FROM bid WHERE item_id = ?;
SELECT * FROM bid WHERE item_id = ?;
SELECT * FROM bid WHERE item_id = ?;
SELECT * FROM bid WHERE item_id = ?;
```

# 2. SQL层的解决方案

在本方案中，首先通过两个步骤对SQL语句加以改造，从SQL层面上解决这个问题。

1. 使用关键字`UNION ALL`将N条查询语句合为一条语句，便将N+1次查询转化为了1+1次查询。

2. 由于第二次查询的所有记录被一次性返回，而我们需要将`Bid`实体关联到相关的`Item`实体上，因此我们需要添加一个额外的`item_id`列以便进行实体关联。

以下是改造后的两条查询语句。

```sql
SELECT * FROM item;
SELECT ? AS item_id, b.* FROM bid b WHERE item_id = ? UNION ALL
SELECT ? AS item_id, b.* FROM bid b WHERE item_id = ? UNION ALL
SELECT ? AS item_id, b.* FROM bid b WHERE item_id = ? UNION ALL
SELECT ? AS item_id, b.* FROM bid b WHERE item_id = ?;
```

When we want to query a bid list and every bid entity to carry its item, we can execute two query statements as follows:

```sql
SELECT * FROM bid;
SELECT ? AS bid_id, i.* FROM item i WHERE id IN (SELECT item_id FROM bid WHERE id = ?) UNION ALL
SELECT ? AS bid_id, i.* FROM item i WHERE id IN (SELECT item_id FROM bid WHERE id = ?) UNION ALL
SELECT ? AS bid_id, i.* FROM item i WHERE id IN (SELECT item_id FROM bid WHERE id = ?) UNION ALL
SELECT ? AS bid_id, i.* FROM item i WHERE id IN (SELECT item_id FROM bid WHERE id = ?);
```

`Item`和`Bid`之间的关系是典型的一对多/多对一关系。以上这一解决方案也可用于多对多关系。

## 3. ORM应用层面的解决方案

对于ORM层，我们需要想办法从表结构的信息中映射到第二条查询语句，Java中常用注解的方式。

上面的SQL语句中只有四个要素，两个表名`item`和`bid`，表`bid`中的外键列`item_id`和表`item`中的引用列`id`。 其中，查询实体的表名是已知的，于是便只剩下三个要素。 DoytoQuery定义了一个注解`@DomainPath`来配置这三个要素，用以映射第二条查询语句。

```java
@Target(FIELD)
@Retention(RUNTIME)
public @interface DomainPath {
    String[] value();
    String localField() default "id";
    String foreignField() default "id";
}
```

由于第二条查询语句中附加的`id`列仅用于实体赋值，因此我们将附加列的别名统一命名为 `MAIN_ENTITY_ID`。`Item`和`Bid`的类定义如下：

```java
@Getter
@Setter
public class ItemView extends AbstractPersistable<Integer> {
    // other fields in Item

    // one-to-many
    // SELECT ? AS MAIN_ENTITY_ID, b.*
    //              FROM bid b           WHERE item_id = ? [UNION ALL ...]
    @DomainPath(value = "bid", foreignField = "item_id")
    private List<BidView> bids;
}

@Getter
@Setter
public class BidView extends AbstractPersistable<Integer> {
    // other fields in Bid

    // many-to-one
    // SELECT ? AS MAIN_ENTITY_ID, i.*
    //              FROM item i           WHERE id      IN (SELECT item_id FROM bid WHERE id = ?)  [UNION ALL ...]
    @DomainPath(value = "item", foreignField = "id", localField = "item_id")
    private ItemView item;
}  
```

假设`Item`和`Category`之间的多对多关系存放于中间表`CATEGORY_ITEM`中，我们可以使用`@DomainPath`来定义如下实体，用以映射第二条查询语句：

```java
@Getter
@Setter
public class ItemView extends AbstractPersistable<Integer> {
    // other fields in Item

    // many-to-many
    @DomainPath({"item", "~", "category"})
    private List<CategoryView> categories;
}

@Getter
@Setter
public class CategoryView extends AbstractPersistable<Integer> {
    // other fields in Category

    // many-to-many
    @DomainPath({"category", "item"})
    private List<ItemView> items;
}
```

## 4. 小结

在本文中，我们介绍了`DoytoQuery`中的一种可以避免n+1查询问题的关联查询方案。并且，我们只需要通过一个注解`@DomainPath`便可管理ERM中定义的四种实体关系。



<div id="busuanzi_container_page_pv">
  本文总阅读量<span id="busuanzi_value_page_pv"></span>次
</div>

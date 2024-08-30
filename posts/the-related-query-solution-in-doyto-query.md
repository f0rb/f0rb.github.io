---
title: 'The Related Query Solution in DoytoQuery (for n+1 selects problem)'
date: 2022-12-05 12:17:25
tags: [DoytoQuery,Java,CRUD,f0rb说]
published: true
hideInList: false
feature: /post-images/the-related-query-solution-in-doyto-query.png
isTop: false
---
In this article, we introduced a solution in DoytoQuery to do the related query avoiding the n+1 problem. 
<!-- more -->


![](https://blog.doyto.win/post-images/1670557996771.png)

## 1. Background
_Java Persistence with Hibernate_ describes the _n+1 selects problem_ in section 12.2.1 with the following example:

```java
List<Item> items = em.createQuery("select i from Item i").getResultList();
// select * from ITEM
for (Item item : items) {
    assertTrue(item.getBids().size() > 0);
    // select * from BID where ITEM_ID = ?
}
```
In this case, each bids collection has to be loaded with an additional SELECT, when there are N records in 'Item', there will be N+1 queries to execute:

```sql
SELECT * FROM item;
SELECT * FROM bid WHERE item_id = ?;
SELECT * FROM bid WHERE item_id = ?;
SELECT * FROM bid WHERE item_id = ?;
SELECT * FROM bid WHERE item_id = ?;
```

## 2. The SQL Layer Solution

In this solution, there are two steps to solve this problem in the SQL layer.

1. Use SQL keyword `UNION ALL` to combine the N query statements into one statement, so the N+1 queries are converted to 1+1 queries.

2. All the records of the second query are returned at one time and we need to assign the `Bid` entities to their related `Item` entity, so we add an extra `item_id` column to help with such assignments.

Here are the two query statements after optimization.

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

The relationship between `Item` and `Bid` is a typical one-to-many/many-to-one relationship. The above solution should also work for the many-to-many relationship.

## 3. The ORM Layer Solution

For the ORM layer, we need to find a way to map to the second query statement from the information of the table structure, using annotation is a common way to do such stuff in Java.

There are only four factors in the above SQL statements, the two table names `item` and `bid`, the foreign key column `item_id` in table `bid` and the referenced column `id` in table `item`. We should always know the table name of the querying entity so that only three factors remain. DoytoQuery defines an annotation `@DomainPath` to hold these three factors.

```java
@Target(FIELD)
@Retention(RUNTIME)
public @interface DomainPath {
    String[] value();
    String localField() default "id";
    String foreignField() default "id";
}
```

Since the additional `id` column is only used for entity assignments, we rename the alias of the additional column to `MAIN_ENTITY_ID`. And the class for the item and bid would be defined as follows:

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

And for a many-to-many relationship between `Item` and `Category` with an associative table named `CATEGORY_ITEM`, we can define the following entities with `@DomainPath` to map to the second query statement:

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

## 4. Conclusion

In this article, we introduced a solution in DoytoQuery to do the related query avoiding the n+1 problem. And we can manage all four relationships in ERM with only one annotation `@DomainPath`.

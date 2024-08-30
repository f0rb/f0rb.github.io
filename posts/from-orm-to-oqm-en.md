---
title: 'From ORM to OQM: An Object-Only SQL Construction Solution'
date: 2024-08-20 21:02:19
tags: [ORM,OQM,f0rb说,en]
published: true
hideInList: false
feature: 
isTop: false
---
Object/Relational Mapping (ORM) may lead the development of object-oriented applications based on relational databases in the wrong direction.

<!-- more -->

There are actually only two steps to perform relational database access: SQL construction and SQL execution. ORM frameworks support SQL execution well, for example, Spring JDBC is a typical SQL execution tool, however, we still need to handwrite SQL or SQL construction code wrapped in any format for complex query statements. What we need is a completely automated SQL construction solution.

Nowadays, most ORM frameworks can resolve the table name and column names through an entity object, and then build basic CRUD statements for a single table, while constructing the query clause requires users to provide several corresponding query parameters. According to the principle of high cohesion, we can define all query parameters of each table in one object, called a query object. Since each field in the query object can be built as a query condition, we can build a complete query clause through such a query object. Among them, these query parameters also include sorting parameters and paging parameters. In this way, the entity object and the query object constitute a pair of orthogonal objects for building CRUD statements for a single table.

In addition to single-table CRUD statements, SQL construction also includes complex query statements. However, entity objects cannot be used to build complex query statements, because the fields in an entity object correspond to the columns in a single table, and the columns in a complex query statement may come from multiple tables or contain expressions, so we need to design another object to build static parts such as aggregate expressions, grouping clauses, and join relationships, which I call it a view object. At the same time, we can reuse query objects to build query clauses in complex query statements. Theoretically, these three types of objects can implement the automatic construction of SQL statements. The following figure shows an example of the Java implementation. The SQL statement on the right is the third query statement of the TPC-H benchmark test.

![A Java Example of OQM](https://blog.doyto.win/post-images/1724155485489.jpg)

This article shares the idea of ​​automatically building SQL statements from objects only. I call this technology Object/Query-Language Mapping(OQM). In my next article, I will introduce the query object mapping method in detail.

---
© 2024 Yuan Zhen. All rights reserved. 
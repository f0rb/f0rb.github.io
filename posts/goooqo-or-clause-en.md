---
title: 'How to express `select * from user where id = ? or name = ? and age = ?` in GoooQo'
date: 2024-09-06 17:40:01
tags: [GoooQo,Go,en,OQM,Forb,ORM]
published: true
hideInList: false
feature: /post-images/goooqo-or-clause-en.webp
isTop: false
---

## Preface

GoooQo is a CRUD framework in Golang based on OQM technique.

OQM is a technique that builds database query statements only through objects, focusing on the mapping relationship between object-oriented programming languages and database query languages.

This article discusses the relationship between logical operators and object structures in SQL statements based on the question in the title.

## Hierarchical structure of query clauses

OQM technique proposes a new viewpoint: **Query clauses in SQL are not flat, but hierarchical**.

Among them, query conditions connected by AND are located at one level, and query conditions connected by OR are located at another level. The query conditions at each level are connected by the same logical operator.

The reason why query clauses are hierarchical is that **the priority of logical operators is different**, and `AND` has a higher priority than `OR`.

In most cases, we use the AND operator to connect query conditions, so we assume AND for the first level. In the query statement of the title, the two conditions connected by OR are thus at the second level, and the two conditions connected by AND in the second condition of the OR statement are at the third level.

Let's use a JSON data to represent such a hierarchical structure:

```json
{
  "userOr": {
    "id": 5,
    "userAnd": {
      "name": "John",
      "age": 30
    }
  }
}
```

In this data, we use the key name suffixes Or and And to specify the logical operator of the nested conditions.

There is only one key userOr at the first level, and the AND connection can be omitted;, so we skip the first AND level and start to build the WHERE clause from the second level:


* The query condition corresponding to the first key-value pair `"id": 5` is `id = ?`;

* The suffix of the second key userAnd is And, and the subquery conditions are connected using AND to obtain `name = ? ADN age = ?`;

* The above two conditions are under key `userOr`, so we connect them using OR.

In this way, we get the query clause in the title: `id = ? OR name = ? AND age = ?`, and the corresponding three parameters: \[5, name, 30].

![Comparison chart](https://blog.doyto.win/post-images/1725275009209.webp)

Through this comparison chart, we can more clearly see the correspondence between the query clauses and JSON data.


## Data construction and mapping relationship

Then we try to use Go struct to construct this JSON data.

First, we define the corresponding struct `UserQuery` in the following nested way:

```go
type UserQuery struct {
    Id      *int       `json:"id,omitempty"`
    Name    *string    `json:"name,omitempty"`
    Age     *int       `json:"age,omitempty"`
    UserOr  *UserQuery `json:"userOr,omitempty"`
    UserAnd *UserQuery `json:"userAnd,omitempty"`
}
```

Then, construct the corresponding variable `userQuery` according to the assignment in the JSON data:

```go
func P[T any](t T) *T { return &t }

func main() {
    userQuery := UserQuery{UserOr: &UserQuery{Id: P(1), UserAnd: &UserQuery{Name: P("John"), Age: P(30)}}}
    bytes, _ := json.Marshal(query)
    print(string(bytes))
}
```  

This code outputs the following, fields without values are ignored:

```json
{"userOr":{"id":1,"userAnd":{"name":"John","age":30}}}
```

This is exactly the same as the JSON data above. In other words, we can construct the same query clause based on the `userQuery` variable here.

## GoooQo interface call

GoooQo uses the relationship between the object structure and the hierarchy of the query clause to construct query clauses containing AND and OR.

First, define the entity object and query object for the user table:

```go
type UserEntity struct {
    goooqo.IntId
    Name     *string `json:"name,omitempty"`
    Age      *int    `json:"age,omitempty"`
}

func (u UserEntity) GetTableName() string { return "user" }

type UserQuery struct {
    goooqo.PageQuery
    Id      *int       `json:"id,omitempty"`
    Name    *string    `json:"name,omitempty"`
    Age     *int       `json:"age,omitempty"`
    UserOr  *UserQuery `json:"userOr,omitempty"`
    UserAnd *UserQuery `json:"userAnd,omitempty"`
}
```
Then create a transaction manager and a data access interface for the user table based on the database connection:

```go
db := rdb.Connect("local.properties")
defer rdb.Disconnect(db)
tm := rdb.NewTransactionManager(db)

userDataAccess := rdb.NewTxDataAccess[UserEntity](tm)
```
Finally, call the `Query` method of `userDataAccess` using the created query object according to the SQL statement that needs to be executed:

```go
userQuery := UserQuery{UserOr: &UserQuery{Id: P(1), UserAnd: &UserQuery{Name: P("John"), Age: P(30)}}}
userEntities, err := userDataAccess.Query(context.Background(), userQuery)
```

The corresponding log information is basically the same as the SQL statement in the title:
```
time="2024-09-02T17:11:27+08:00" level=info msg=Executing SQL="SELECT id, name, age FROM user WHERE (id = ? OR name = ? AND age = ?)" args="[5 John 30]"
```

In this way, when we need to execute different SQL statements, we only need to assign values to the corresponding fields according to the query conditions. For projects with frontend and backend interactions, using this type of query object to receive parameters passed in by the front-end can greatly reduce the amount of backend coding.


## Improvement

There is still a small problem here. What if the type of the field with the Or suffix is an array, such as `UserOr *[]UserQuery`?

In GoooQo, the query conditions corresponding to the fields inside each struct are connected by AND; each set of query conditions are connected by OR.

Then we construct the follwing struct to build the same query statement:
```go
userQuery := &UserQuery{UserOr: &[]UserQuery{ {Id: P(1)}, {Name: P("John"), Age: P(30)}}}
//json: {"userOr":[{"id":1},{"name":"John","age":30}]}
```

The key `userOr` becomes an array containing two elements, the operator between elements is OR, and the operator within an element is AND.

Thus, the corresponding relationship between each query condition and the object field of the query statement is expressed as follows:
![Comparison chart](https://blog.doyto.win/post-images/1725615897117.webp)

In addition, we can also find that when the array `userOr` contains only one element, it is actually the same as the condition mapped by the field `userAnd`, so we can further optimize the `userAnd` field.

## Conclusion

This article introduces the hierarchical relationship of query clauses and their relationship with the object structure, and it also explains in detail how to use object structures to construct query clauses containing AND and OR operators in OQM technique.

Traditional ORM technique focuses on the mapping between object model and relational model, and does not point out the connection between object-oriented programming languages and query languages. Therefore, ORM frameworks don't have a unified implementation for constructing query clauses containing OR.

Furthermore, we will find that since such query conditions can be expressed in JSON data, it means that any programming language that supports JSON can construct such query conditions. Next, we can try to implement the solution described in this article in different programming languages.

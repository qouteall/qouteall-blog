---
date: 2026-01-21
tags:
  - Programming
unlisted: true
---
# Notes about SQL

<!-- truncate -->

## Why move compute to database

There are two paradigms:

- The database just simply stores data. It doesn't support doing custom compute on its data. KV databases.
- The database allows executing custom compute within database. Then you need a query language or scripting to specify compute done in database.

The second paradigm of moving compute into database is more complex. The application need to have a distinction of what compute is done in database and what is done in app's own code. This practice makes **compute closer to data**, which then can create benefits:

- By making compute done in database, less networking IO is required between app and database.
- Allows easily doing special queries (e.g. some statistics for business insight) just by interacting with database.
- Expose more compute details to database so that database can optimize. It can optimize actual data structure and maintain acceleration structure (index).  

## Why separate app and database


## Data modelling

- Relational (tables)
- Tree. [Hierarchical database model](https://en.wikipedia.org/wiki/Hierarchical_database_model)
- Graph.


## CRUD biolerplate

The data model of SQL schema differs to the in-memory data model in programming language. This leads to a lot of CRUD biolerplates that convert data between two models.

## Cumbersome lists

If one user can have multiple labels, then you need to add a new table, then CRUD also need to handle the new table. Changing the labels of one user needs deleting then inserting (or use `merge` statement, but it's unintuitive)

Fortunately modern databases provide things like array and json that alleviate this problem.

## N+1 problem and networking latency

N+1 problem is a natural consequence of writing many modular, reusable and single-responsibility SQL. To solve N+1 you have to write large, complex and deeply coupled SQL.

## ORM problems

- ORM aim to solve the conversion issue. However, ORM creates new issues:
  - ORM may query all the correlations for convenience. When the correlation data is not needed, it will degrade performance.
  - ORM can easily trigger N+1 problem.
  - ORM that track object identity can cause many issues. You cannot treat object as just data. You cannot create two entity objects with the same primary key. You need to carefully track existing objects.

## Conditional filtering, sorting and paging

A common pattern is to have an offset and limit number for paging. But it has DOS attack risk. 

To avoid the DOS risk, get rid of paging and replace with "infinite" scrolling, query by previous sorting key instead of offset. Sorting key need to be in index.

If there is a complex filtering condition, expressing it in SQL may be hard. But paging in SQL requires expressing filter condition in SQL 

## SQL injection


## Lack built-in cache, maintaing external cache coherency is hard

If use redis as cache, then you need to deal with cache invalidation issue.

Under high concurrency, there are all kinds of anomaly

```pseudocode
query from cache
if cache doesn't have it:
    query from DB
    (A GC pause happens here)
    put result into cache
```

```pseudocode
write to DB and commit transaction
delete relevant data from cache
```

that has risk of making stale cache exist for long time

## Lack built-in event queue

If you need a transaction across SQL DB and Kafka, it will be hard. 

However most just don't care about transaction and consistency.


## Repeatable read level trap

Serializable isolation level is easy-to-understand: it prevents possible concurrency issues at the cost of performance. It's actually not slow if all transactions are short and usually touch different data.

Read-commited level is also easy-to-understand: it involves some concurrency issues.

But repeatable-read level is complex and hard-to-understand:

- It usually means snapshot isolation. But locking can break it.
- It doesn't avoid write skew in PostgreSQL


## Drawbacks of Non-SQL solutions

Although SQL has many problems, non-SQL solutions also have problems:

In MongoDB, because there is no schema constraint, the actual data format tend to be chaotic and hard-to-maintain.

What if don't use DB? Manually reading and writing files involve other issues:

- Simply reading and writing the whole file is easy. But if the data is large, it's inefficient. Reading or writing parts of the file is much more complex.
- When querying data, read all related files is slow. You need special in-disk data structures to accelerate.  
- Backing up data.
- No transaction support. If app quits or crashes during writing, may leave corrupted partially-written file.







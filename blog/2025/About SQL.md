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





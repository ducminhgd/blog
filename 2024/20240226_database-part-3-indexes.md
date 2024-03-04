# Data - Part 3: Indexes

> This post mostly focuses in PostgreSQL

Indexes allow the database server to find and retrieve specific rows much faster than it could do without them.

## Types of Indexes in PostgreSQL:

- B-Tree Index: The default index type in PostgreSQL. It’s suitable for most use cases.
- Hash Index: Useful for equality-based queries.
- GIN (Generalized Inverted Index): Ideal for complex data types like arrays and full-text search.
- GiST (Generalized Search Tree): Useful for geometric and text search queries.
- SP-GiST (Space-Partitioned Generalized Search Tree): Designed for spatial data.
- BRIN (Block Range INdex): Efficient for large tables with sorted data.
- Partial Index: Allows indexing a subset of rows based on a condition.
- Unique Index: Ensures uniqueness of values in a column.
- Expression Index: Based on expressions computed from column values.
- Multicolumn Index: Combines multiple columns for more efficient queries

## Manage index

- Monitor Index Usage: Use the `pg_stat_user_indexes` system view to monitor the usage of your indexes and determine which indexes are being used and which are not.
- Rebuild Indexes: Use the `REINDEX` command to rebuild an index if it becomes fragmented or if its performance decreases.
- Vacuum Regularly: Use the `VACUUM` command to reclaim space and improve the performance of your indexes.
- Drop Unused Indexes: Use the `DROP INDEX` command to drop indexes that are not being used.

## Secondary indexes

Every index in PostgreSQL is a secondary index - a data structure stored separately from the heap table structure, with some type of pointer into the heap table. In PostgreSQL, all tables are heap tables, so no order is maintained by the relational engine for the table. There isn’t the ability to define a clustered index to maintain the order of the data in the table physically by a key. Because the data in the table is unordered, it can make certain types of scan operations less efficient. However, PostgreSQL does have the concept of the CLUSTER command for a table. When the CLUSTER command is executed against a table, the data in the table is physically ordered based on the keys of the secondary index (more on those in a bit) you provide

## Compare B-Tree Index and Hash Index

- Hash Index only supports single column-key.
- Hash indexes, do not store the actual key value, only the 4-byte signed hashed value of the key

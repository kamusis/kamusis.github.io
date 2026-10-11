# The best practices for using partition table in PostgreSQL

## What is partition table?

Partition table is a table that is divided into smaller parts. Each part is called a partition.

Why use partition table?

1. **Performance**: Partition table can improve the performance of the table. Because each partition can be stored in a separate file, the query can be executed faster.
2. **Maintenance**: Partition table can make the maintenance of the table easier. Because each partition can be maintained separately, the maintenance of the table can be done faster.

## How to create a partition table?

1. **Create a partition table**:

```sql
CREATE TABLE measurement (
    city_id         int not null,
    logdate         date not null,
    peaktemp        int,
    unitsales       int
) PARTITION BY RANGE (logdate);
```

2. **Create a partition table for each partition**:

```sql
CREATE TABLE measurement_y2006m02 PARTITION OF measurement
    FOR VALUES FROM ('2006-02-01') TO ('2006-03-01');
...
CREATE TABLE measurement_y2007m11 PARTITION OF measurement
    FOR VALUES FROM ('2007-11-01') TO ('2007-12-01');
```

## Partition table types

There are 3 basic types of partition table in PostgreSQL:

1. **List Partition**: The table is partitioned by explicitly listing which key value(s) appear in each partition.
2. **Range Partition**: The table is partitioned into “ranges” defined by a key column or set of columns, with no overlap between the ranges of values assigned to different partitions.
3. **Hash Partition**: The table is partitioned by specifying a modulus and a remainder for each partition.

Combine these 3 basic types of partition table, we can create more complex partition table. This is called **Composite Partitioning**.

There are 4 types of composite partitioning:

1. **List-Range Partition**: The partition is divided into several ranges.
The step is: create table A partition by list, then create table A1 partition of A partition by range, then create table A1-1 partition of A1
2. **Range-List Partition**: The partition is divided into several ranges.
The step is: create table A partition by range, then create table B of table A but partition by list.
3. **Range-Hash Partition**: The partition is divided into several ranges.
4. **Hash-Range Partition**: The partition is divided into several ranges.

## Data dictionary about partition table

1. **pg_partitioned_table**: This table stores the information about the partitioned table.
2. **pg_partition_rule**: This table stores the information about the partition rule.
3. **pg_partition_column**: This table stores the information about the partition column.
4. **pg_partition_index**: This table stores the information about the partition index.
5. **pg_partition_constraint**: This table stores the information about the partition constraint.
6. **pg_partition_constraint_column**: This table stores the information about the partition constraint column.
7. **pg_partition_constraint_index**: This table stores the information about the partition constraint index.










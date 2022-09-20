## `CREATE TABLE`

```postgresql
CREATE TABLE sample (id bigint COMMENT 'unique id', data string);
```



Iceberg will convert the column type in Spark to the corresponding Iceberg type. Please check the section of [type compatibility on creating table](https://iceberg.apache.org/spark-writes/#spark-type-to-iceberg-type) for details.

Table create commands, including CTAS and RTAS, support the full range of Spark create clauses, including:

- `PARTITION BY (partition-expressions)` to configure partitioning
- `LOCATION '(fully-qualified-uri)'` to set the table location
- `COMMENT 'table documentation'` to set a table description
- `TBLPROPERTIES ('key'='value', ...)` to set [table configuration](https://iceberg.apache.org/configuration/)

<br/>

### `PARTITIONED BY`

To create a partitioned table, use `PARTITIONED BY`:

```postgresql
CREATE TABLE sample ( id bigint, data string, category string) PARTITIONED BY (category)
```



The `PARTITIONED BY` clause supports transform expressions to create [hidden partitions](https://iceberg.apache.org/partitioning/).

```postgresql
CREATE TABLE sample (
    id bigint,
    data string,
    category string,
    ts timestamp)
PARTITIONED BY (bucket(16, id), days(ts), category)
```



Supported transformations are:

- `years(ts)`: partition by year
- `months(ts):` partition by month
- `days(ts)` or `date(ts)`: equivalent to dateint partitioning
- `hours(ts)` or `date_hour(ts)`: equivalent to dateint and hour partitioning
- `bucket(N, col)`: partition by hashed value mod N buckets
- `truncate(L, col)`: partition by value truncated to L
  - Strings are truncated to the given length
  - Integers and longs truncate to bins: truncate(10, i) produces partitions 0, 10, 20, 30, …

<br/>

***



<br/>

## `CREATE TABLE ... AS SELECT`

```postgresql
CREATE TABLE sample
AS SELECT ...
```



<br/>

***



<br/>

## `REPLACE TABLE ... AS SELECT`

Atomic table replacement creates a new snapshot with the results of the `SELECT` query, but keeps table history.

```postgresql
REPLACE TABLE sample
AS SELECT ...
```



```postgresql
CREATE OR REPLACE TABLE sample
AS SELECT ...
```



> 🚧 
> 
> The schema and partition spec will be replaced if changed. To avoid modifying the table’s schema and partitioning, use INSERT OVERWRITE instead of REPLACE TABLE. The new table properties in the REPLACE TABLE command will be merged with any existing table properties. The existing table properties will be updated if changed else they are preserved.

<br/>

***



<br/>

## `DROP TABLE`

To delete a table, run:

```sql
DROP TABLE sample
```



<br/>

***



<br/>

## `ALTER TABLE`

Full `ALTER TABLE` support:

- Renaming a table
- Setting or removing table properties
- Adding, deleting, and renaming columns
- Adding, deleting, and renaming nested fields
- Reordering top-level columns and nested struct fields
- Widening the type of `int`, `float`, and `decimal` fields
- Making required columns optional

### `ALTER TABLE ... RENAME TO`

```sql
ALTER TABLE sample RENAME TO new_name
```



### `ALTER TABLE ... SET TBLPROPERTIES`

```sql
ALTER TABLE sample SET TBLPROPERTIES (
    'read.split.target-size'='268435456'
)
```



Iceberg uses table properties to control table behavior. For a list of available properties, see [Table configuration](https://iceberg.apache.org/configuration/).

UNSET is used to remove properties:

```sql
ALTER TABLE sample UNSET TBLPROPERTIES ('read.split.target-size')
```



### `ALTER TABLE ... ADD COLUMN`

To add a column use the `ADD COLUMNS` clause with `ALTER TABLE`:

```sql
ALTER TABLE sample
ADD COLUMNS (
    new_column string comment 'new_column docs'
 )
```



Multiple columns can be added at the same time, separated by commas.

Nested columns should be identified using the full column name:

```sql
-- create a struct column
ALTER TABLE sample
ADD COLUMN point struct<x: double, y: double>;

-- add a field to the struct
ALTER TABLE sample
ADD COLUMN point.z double
```



You can add columns in any position by adding `FIRST` or `AFTER` clauses:

```sql
ALTER TABLE sample
ADD COLUMN new_column bigint AFTER other_column
```



```sql
ALTER TABLE sample
ADD COLUMN nested.new_column bigint FIRST
```



### `ALTER TABLE ... RENAME COLUMN`

To rename a field, use `RENAME COLUMN`:

```sql
ALTER TABLE sample RENAME COLUMN data TO payload
ALTER TABLE sample RENAME COLUMN location.lat TO latitude
```



Note that nested rename commands only rename the leaf field. The above command renames `location.lat` to `location.latitude`

### `ALTER TABLE ... ALTER COLUMN`

Alter column is used to widen types, make a field optional, set comments, and reorder fields.

Iceberg allows updating column types if the update is safe. Safe updates are:

- `int` to `bigint`
- `float` to `double`
- `decimal(P,S)` to `decimal(P2,S)` when P2 > P (scale cannot change)

```sql
ALTER TABLE sample ALTER COLUMN measurement TYPE double
```



To add or remove columns from a struct, use `ADD COLUMN` or `DROP COLUMN` with a nested column name.

Column comments can also be updated using `ALTER COLUMN`:

```sql
ALTER TABLE sample ALTER COLUMN measurement TYPE double COMMENT 'unit is bytes per second'
ALTER TABLE sample ALTER COLUMN measurement COMMENT 'unit is kilobytes per second'
```



Iceberg allows reordering top-level columns or columns in a struct using `FIRST` and `AFTER` clauses:

```sql
ALTER TABLE sample ALTER COLUMN col FIRST
```



```sql
ALTER TABLE sample ALTER COLUMN nested.col AFTER other_col
```



Nullability can be changed using `SET NOT NULL` and `DROP NOT NULL`:

```sql
ALTER TABLE sample ALTER COLUMN id DROP NOT NULL
```



Note

`ALTER COLUMN` is not used to update `struct` types. Use `ADD COLUMN` and `DROP COLUMN` to add or remove struct fields.

### `ALTER TABLE ... DROP COLUMN`

To drop columns, use `ALTER TABLE ... DROP COLUMN`:

```sql
ALTER TABLE sample DROP COLUMN id
ALTER TABLE sample DROP COLUMN point.z
```



### `ALTER TABLE ... ADD PARTITION FIELD`

Add new partition fields to a spec using `ADD PARTITION FIELD`:

```sql
ALTER TABLE sample ADD PARTITION FIELD catalog -- identity transform
```



[Partition transforms](https://iceberg.apache.org/spark-ddl/#partitioned-by) are also supported:

```sql
ALTER TABLE sample ADD PARTITION FIELD bucket(16, id)
ALTER TABLE sample ADD PARTITION FIELD truncate(data, 4)
ALTER TABLE sample ADD PARTITION FIELD years(ts)
-- use optional AS keyword to specify a custom name for the partition field 
ALTER TABLE sample ADD PARTITION FIELD bucket(16, id) AS shard
```



Adding a partition field is a metadata operation and does not change any of the existing table data. New data will be written with the new partitioning, but existing data will remain in the old partition layout. Old data files will have null values for the new partition fields in metadata tables.

> 📘 
> 
> To migrate from daily to hourly partitioning with transforms, it is not necessary to drop the daily partition field. Keeping the field ensures existing metadata table queries continue to work.

> 🚧 
> 
> **Dynamic partition overwrite behavior will change** when partitioning changes For example, if you partition by days and move to partitioning by hours, overwrites will overwrite hourly partitions but not days anymore.

### `ALTER TABLE ... DROP PARTITION FIELD`

Partition fields can be removed using `DROP PARTITION FIELD`:

```sql
ALTER TABLE prod.db.sample DROP PARTITION FIELD catalog
ALTER TABLE prod.db.sample DROP PARTITION FIELD bucket(16, id)
ALTER TABLE prod.db.sample DROP PARTITION FIELD truncate(data, 4)
ALTER TABLE prod.db.sample DROP PARTITION FIELD years(ts)
ALTER TABLE prod.db.sample DROP PARTITION FIELD shard
```



Note that although the partition is removed, the column will still exist in the table schema.

Dropping a partition field is a metadata operation and does not change any of the existing table data. New data will be written with the new partitioning, but existing data will remain in the old partition layout.

> 🚧 
> 
> **Dynamic partition overwrite behavior will change** when partitioning changes For example, if you partition by days and move to partitioning by hours, overwrites will overwrite hourly partitions but not days anymore.

> 🚧 
> 
> Be careful when dropping a partition field because it will change the schema of metadata tables, like files, and may cause metadata queries to fail or produce different results.

### `ALTER TABLE ... WRITE ORDERED BY`

Iceberg tables can be configured with a sort order that is used to automatically sort data that is written to the table in some engines. For example, `MERGE INTO` in Spark will use the table ordering.

To set the write order for a table, use `WRITE ORDERED BY`:

```sql
ALTER TABLE sample WRITE ORDERED BY category, id
-- use optional ASC/DEC keyword to specify sort order of each field (default ASC)
ALTER TABLE sample WRITE ORDERED BY category ASC, id DESCß
-- use optional NULLS FIRST/NULLS LAST keyword to specify null order of each field (default FIRST)
ALTER TABLE sample WRITE ORDERED BY category ASC NULLS LAST, id DESC NULLS FIRST
```



> 📘 
> 
> Table write order does not guarantee data order for queries. It only affects how data is written to the table.
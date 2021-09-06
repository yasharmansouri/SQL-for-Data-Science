# Query Tuning

## 1. General Tips

- The first step in any query is to only choose the columns that are required
- Don't use `WHERE colum = NULL`, use `IS NULL` instead
- Don't use functions in WHERE clause unless you have a functional index
- If a plan uses index range scan, keep the range as small as possible
- `LIKE 'ABC%'` can use an index while `LIKE '%ABC'`can't
- if we need to use `ORDER BY column`, it's better to have an index on the same column
- When filtering on range conditions, especially dates, use continous conditions, such as `TRUNC(sysdate)` and `TRUNC(sysdate+1)`
- Don't separate date and time into separate columns. Use a datetime data type
- Don't store numeric data as char, varchar, or text; use a numeric type (except phone numbers)

## 2. Indexes

Indexes reduce the need for full table scanning. Creating indexes can be beneficial:

- Speed up access to data
- Help enforce constraints
- Indexes are ordered
- Typically smaller than tables
- If most queries on a table require full table scan, then indexes may not be used. However indexing the table will still be beneficial for table joins if all columns are indexed. This indexing is called Covered Index.
- Postgres by default creates indexes on primary keys.

```sql
-- creates an index on the salary column of the staff table
CREATE INDEX idx_staff_salary ON staff(salary)

EXPLAIN ANALYZE SELECT * FROM staff WHERE salary > 75000;
-- if the analyze doesn't show index is used, it could be due to query execution builder determining there are many salaries above 75000 and it's better to run query without using indexes. so if we had chosen salary > 15000 index would be used and query execution would be faster
```

PostgreSQL has multiple type of indexes:

- (Balanced) B-Tree: The most commonly used indexes. They're useful when a large number of possible values can exist in a column i.e. high cardinality.  They are similar to trees such that depending on the range of a value, the tree branches up/down, so we can always lookup where the specific value is located. Time to access is based on the depth of the tree.

```sh
b-tree example
             ->150
50 -> 75->125
             ->100
   ->25
```

```sql
--B-tree example in SQL: creating an index on the email column of the staff table
CREATE INDEX idx_staff_email ON staff(email);
```

- Bitmap: They are suited for low cardinality columns. They store a series of bits for the columns and the number of bits is as same as the number of distinct values in the columns. For example a boolean column will get yes: 0,1 and no: 0,1. Bitmap indexes perform boolean operations quickly. For example performing two boolean operations with `AND/OR/NOT` on bitmap indexes are faster. Other DBMS allow you to create bitmap indexes explicitly while PostgreSQL builds bitmap indexes on the fly. So we might see some bitmap indexes created even though we didn't index the table.

```sh
bitmap example
Condition 1: Yes:  0, 1
Condition 2: Dead: 0, 1
=>           Yes and Dead : 1, 1
```

```sql
CREATE INDEX idx_staff_job_title ON staff(job_title);

EXPLAIN ANALYZE SELECT *
FROM staff
WHERE job_title = 'Operator';
```

- Hash: Hash functions take an arbitrary lenght of data and turn it into fixed-size string.
  - Hash functions are designed in such a way that different inputs produce different outputs and the values are virtually unique.
  - Size of hash depends on the algorithm used.
  - There's no preserving order with hash functions.
  - Hash functions are used for equality comparisons and not range comparisons.

```sql
-- creating hash index on email column of the staff table
CREATE INDEX idx_staff_email ON staff USING HASH(email);
```

- Postgres Specific Indexes: Requires more research.

## 3. Join Performance

INNER JOINs are often the fastest since OUTER JOINs and FULL JOINs require additional steps to INNER JOINs. SQL Execution plan uses multiple types of joins:

- Nested loops:
  - They're a naive approach to joining tables with the advantage that they will work with any kind of join. Nested loops run two loops, one within the other. `for row in table 1 (driver) : for row in table 2 (join):`. Nested loops work with all join conditions, have low overhead, work well with small driver tables especially if the joined column is indexed. However they can be slow and if the tables do not fit in memory, then the performance would be even slower.
  - Indexes can improve the performance of nested loop joins especially if the index covers all the columns. In table joins it always helps to have indexes on columns that are matching together.

```sql
-- nested loop example
-- specifying directly which loops to choose for joins
set enable_nestloop=true;
set enable_hashjoin=true;
set enable_mergejoin=false;

EXPLAIN SELECT
s.id, s.last_name, s.job_title, cr.country
FROM staff s
INNER JOIN company_regions cr
ON s.region_id = cr.region_id
```

- Hash Joins: In first step, hash values for the smaller table in a join are generated for the primary keys and stored in the table. In the next phase which is called Probe Phase, the algorithm steps through the larger table, commputes the hash value of primary or foreign key and looks up the corresponding value in hash table.
  - Hash joins only work with equality conditions i.e. `WHERE column='value'`
  - The time it takes the joins is based on the hash table to be generated and then going through the larger table, calculating hash values and performing lookups
  - The key advnatage of hash joins are that they can perform fast lookups and can be faster than other join methods.

```sql
set enable_nestloop=false
set enable_hashjoin=true
set enable_mergejoin=false

EXPLAIN SELECT
s.id, s.last_name, s.job_title, cr.country
FROM staff s
INNER JOIN company_regions cr
ON s.region_id = cr.region_id
```

- Merge Joins: They're also known as Sort Merge Joins. They sort the input tables prior to looking up rows by key values. Sorting allows the algorithm to take advantage of ordering to reduce the number of rows checked.
  - Merge joins are only used with equality operations
  - The time required to execute the join is based on table size to sort and the time to scan
  - They're especially useful when tables don't fit into memory

```sql
set enable_nestloop=false
set enable_hashjoin=false
set enable_mergejoin=true

EXPLAIN SELECT
s.id, s.last_name, s.job_title, cr.country
FROM staff s
INNER JOIN company_regions cr
ON s.region_id = cr.region_id
```

- Subqueries vs Joins
  - Conventional wisdom says that joins are always faster than subqueries, however, query plan builders have become more effective at optimizing subqueries.
  - It's better to optimize for clarirty i.e. making the intention of the query clear, unless there's a significant loss in performace

## 4. Partitioning

Large tables can be difficult to query efficiently and it can be better to partition/split them. Partitions are manly used on several database applications such as data warehouses. Each partition can have its own indexes, constraints, rules, and triggers since they are treated like tables, however, columns are inherited from the table level.
Partition Key: Determines which partition is used for data.
Partition Bounds: Min and Max values allowed in a partition.

- Horizontal Partitioning: Splitting the table by rows. Makes each partition to be treated like a table, which means we are able to limit scans or create indexes on each partition.
  - Horizontal partitioning is mainly done on timeseries data since most recent data is the most likely data to be queried and older data is summarized in older paritions. There are other partitioning logics such as regional, product type, category, etc.
    - Range Partitioning (FROM - TO): A type of horizontal partitioning, usually done on date values (e.g. monthly, yearly) but they can also be used in alphabetical or numeric ranges.
      - Useful when usually querying latest data or running comparative queries such as this month's sales vs. last month's sales.
      - Useful when dropping older records since it can done by dropping the partition.
    - List Partitioning (IN): Data is divided based on a a partition key that is defined as a list.
      - Suitable when data logically groups into subgroups.
      - Queries are often done on subgroups
      - Good option when data is not so date/time oriented
    - Hash Partitioning (WITH): The partition key value is an input tool function that computes a value that decides which partiion should store the row. The partition key is not used directly, but instead in PostgreSQL, modulus division is applied to the partion key to compute a value that is based on the number of the partitions.
      - Useful when data does not fall into subgroups
      - When we want even distribution among partition

```sql
-- partition by range example
CREATE TABLE iot_measurement
            (location_id INT NOT NULL,
             measure_date DATE NOT NULL,
             temp_celcius INT,
             rel_humidity_pct INT
            )
    PARTITION BY RANGE (measure_date);


-- creating week long partitions from iot_measurement
CREATE TABLE iot_measurement_wk1_2019
            PARTITION OF iot_measurement
                     FOR VALUES FROM ('2019-01-01') TO ('2019-01-08');
CREATE TABLE iot_measurement_wk2_2019
            PARTITION OF iot_measurement
                     FOR VALUES FROM ('2019-01-08') TO ('2019-01-15');
CREATE TABLE iot_measurement_wk3_2019
            PARTITION OF iot_measurement
                     FOR VALUES FROM ('2019-01-15') TO ('2019-01-22');
```

```sql
-- partition by list example
CREATE TABLE products
            (prod_id INT NOT NULL,
             prod_name TEXT NOT NULL,
             prod_short_desc TEXT NOT NULL,
             prod_long_desc TEXT NOT NULL,
             prod_category VARCHAR,
            )
    PARTITION BY LIST (prod_category);

CREATE TABLE product_clothing PARTITION OF products
            PARTITION OF products
                     FOR VALUES IN ('casual_clothing', 'business_attire', 'formal_clothing');
CREATE TABLE product_electronics PARTITION OF products
            PARTITION OF products
                     FOR VALUES IN ('mobile_phones', 'tablets', 'laptop_computers');
CREATE TABLE product_kitchen PARTITION OF products
            PARTITION OF products
                     FOR VALUES IN ('food_processor', 'cutlery', 'blenders');
```

```sql
-- partion by hash example
CREATE TABLE customer_interaction
            (ci_id INT NOT NULL,
             ci_url TEXT NOT NULL,
             time_at_url INT NOT NULL,
             click_sequence INT NOT NULL)
             PARTITION BY HASH (ci_id);

CREATE TABLE customer_interaction_1
            PARTITION OF customer_interaction
                     FOR VALUES WITH (MODULUS 3, REMAINDER 0);
CREATE TABLE customer_interaction_2
            PARTITION OF customer_interaction
                     FOR VALUES WITH (MODULUS 3, REMAINDER 1);
CREATE TABLE customer_interaction_3
            PARTITION OF customer_interaction
                     FOR VALUES WITH (MODULUS 3, REMAINDER 2);
```

- Vertical Partitioning: Separates the columns of a large table into multiple tables with columns that are tend to be frequently queried together.
  - When using vertical partitioning, the same primary key is used across all paritions
  - Vertical partitioning allows increasing the number of rows that are stored in a single data block
  - Global indexes can be created on each partition
  - They can be seen in data warehouses especially in product tables since they have a lot of attributes.
  - They're also useful in data analysis after finding which subset of attributes is most important in the analysis.

## Materialized Views

Materialized views are precomputed results from expensive queries. While they save time, they take up space as well because they're duplicating data that's already stored in tables.

- Data can become stale in these views and they have to be refreshed. Update to source tables require update to the views `REFRESH MATERIALIZED VIEW`. This can lead to inconsistency.
- They should be used only if we can tolerate inconsistency
- They're useful when saving more time is more useful than saving storage.
- We can create indexes on materialzed views as well

```sql
CREATE MATERIALIZED VIEW mv_staff AS
 SELECT
  s.last_name, s.department, s.job_title,
  cr.company_regions
 FROM
  staff s
INNER JOIN
 company_regions cr
ON
 s.region_id = cr.region_id

SELECT * FROM mv_staff;

-- refreshing materialized view: rebuilds the entire view
REFRESH MATERIALIZED VIEW mv_staff;
```

## Housekeeping / Statistics

Reindexing:
`REINDEX` command rebuilds corrupt indexes.

- Isn't usually done, but helps with bugs if they occur
- Cleans up unused pages in B-tree indexes.
- `REINDEX INDEX [indexname]`: Reindexes a specific index
- `REINDEX TABLE [tablename]` : Reindexes all indexes on a specific table
- `REINDEX SCHEMA [schemaname]` : Reindexes all indexes on a specific schema

Statistics:
On top of tables, indexes, constraints, views, and materialized views, schemas also contain statistics specifically about the distribution of the data. These statistics helps the query plan builder to make the optimal choices for executing queries.

- Size of the data: Number of rows, amount of storage
- Frequency of values: Fraction of nulls, number of distinct values, frequent values
- Distribution of the data: Histograms describing spread of data

One way to get statistics is to use the `ANALYZE` command or `VACCUM`. We can also set up the AUTO VACCUM daemon.

- `VACCUM` reclaims some space of updated data
- `VACCUM(FULL) [tablename]` locks tables and reclaims more space
- `VACCUM(FULL, ANALYZE) [tablename]` performs full vaccum and collects statistics

## Query Hints

DBMS softwares such as Oracle, or MySQL take inline query hints. PostgreSQL uses `SET` command to turn on/off specific indexing or joins.

```sql
SET enable_nestloop=false
SET enable_hashjoin=true
SET enable_mergejoin=false
```

PostgreSQL does not support inline hints like other DBMS softwares and generally it is recommended to do `VACUM` and `ANALYZE` instead.

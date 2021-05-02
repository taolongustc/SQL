## SQL optimization

This guide contains tips on how to optimize Hive query performance.

### 1. Limit Query Results
Most queries return a large data set that can affect system stability. So restrict the results to a specific number of rows. For example, to return the first ten results, end the query with a `limit 10;`.

Also, try to be stingy about the data output on each step when logic allows:

1). If duplicates can be removed at an earlier stage, then do it at earlier stage.
2). Often Uber source data has a lot rows with null values. If possible **filter out nulls sooner**.

### 2. Use Filters

#### 2.1 Filter data as soon as possible.
Because of the way Hive works internally, the sooner the filter appears in a query, the smaller the data set that has to be processed later. Try to discover filter logic that can be advanced to an earlier stage. If your Hive script has multiple queries, if a join condition or where condition used in query #3 can actually be used on query #1, use it at query #1 instead.

#### 2.2 Flatten data if the source data is deeply nested and used multiple times.

Hive does not support field filtering inside structs, meaning Hive needs to read the whole struct record to read a field’s value inside the struct. If data is deeply embedded into nested constructs and the hive script needs to fetch the data out of it multiple times, you should flatten the data out once into a temporary table. Then use that temporary table instead.

### 3. Avoid timeouts on long running queries
Hive cancels queries that run longer than 12 hours by default. If your query takes longer than 12 hours to run, you can adjust this query parameter:
```
hive.query.timeout.seconds=43200;
```
The timeout is expressed in seconds. For queries that need more than 12 hours to run, simply increase the timeout value. For example, if your query takes 24 hours to run, then set the timeout parameter to 86400 seconds in your query, as shown here:
```
hive.query.timeout.seconds=86400;
```

### 4. Rewrite `count(DISTINCT)`
To avoid overloading the Hive MapReduce node reducer, rewrite queries containing `count(DISTINCT)` as follows:
Instead of `SELECT count(DISTINCT f1) FROM t1;` Write:
```
SELECT
  count(1)
FROM (
  SELECT DISTINCT f1 FROM t1
) tmp;
```
A later version of Hive does this automatically. But for the version we use at Uber, you’ll need to do it manually.

### 5. Avoid global ordering: Limit ORDER BY Queries
If you use `ORDER BY`, you must add a limit clause to restrict the number of results returned. Hive rejects `ORDER BY` queries lacking a `limit` clause with the following error message: `FAILED: SemanticException 1:84 Order by-s without limit are disabled for safety reasons`.

In strict mode (hive.mapred.mode=strict), the `ORDER BY` clause must be followed by the `LIMIT` clause.

- **Do**: `Select col1 from ta order by col1 limit 100;`
- **Don’t**: `Select col1 from ta order by col1;`

The `LIMIT` clause is unnecessary if you set: `hive.strict.checks.large.query=false`.

If you want to query entire tables and receive complete results, set: `hive.strict.checks.large.query=true or hive.mapred.mode=nonstrict`.

### 6. Distribute Data correctly using `distribute by`
If you distribute the data poorly, sometimes one reducer will take on the bulk of the computation work. This can result in long wait times. We can control data distribution by using `DISTRIBUTE BY`. Use `DISTRIBUTE BY + SORT BY` if you only need something sorted within a key (e.g. events sorted within a session).

#### 6.1 Static vs Dymanic partition loading
If you aren’t sure exactly which partitions to load, use dynamic partition loading. However, if there are <= 3 partitions to be loaded, it is faster to use static partition loading.

**Dynamic partition loading:**
```
Insert overwrite <table> partition(<part_column>)  <select_list> from table where part_column <condition>
```

Here, `select_list` has the partition column as the last item.

**Static partition loading:**
```
Insert overwrite <table> partition(<part_column=part_value>)  <select_list> from tale where part_column=part_value
```
Here, `select_list` doesn’t include the partition column.

### 7. Avoid Too Fine-Grained Partitions
Table partitioning is commonly used to help improve query performance, thanks to Hive’s partition pruning feature. Table partitioning splits data into ‘partition’ directories. Partition pruning allows the query engine to only scan data in certain partitions as opposed to scanning full tables, thus reducing the amount of data to be processed.

Your table partitioning strategy should be based on query patterns in the table.

Example: If most queries have a filter on `city_id`: `Select … from <table> where city_id {like|=|>|< }`. If the table is partitioned on `city_id`, then only those partitions qualified by the filter condition will be used.

If most queries have a filter on datestr: `Select … from <table> where datestr {>|=|<}`, then partitioning on datestr optimizes the query.

**However, too fine-grained partitioning counteracts the benefits of partition pruning.**

**Example**: if most queries have filters on `date-hour` and `city_id`, it is tempting to partition the table into 3 levels: /date/hour/city_id, or city_id/date/hour. This has 2 unwanted effects:
- Too many partitions, meaning too many directories. (#date x #hour x #city_id)
- Each partition has too little data

**Practice**: Keep the total number of partitions below in the thousands, and the file size inside each partition is not below MB. For Parquet/ORC files near GB level, file size has been shown to matter for performance.

### 8. Use `max/min(struct())` to find the first/last ranking row by value

Use `max(struct())` to find the highest-ranked row (or use `min(struct())` to find the lowest-ranked row) based on some value. `max(struct())` is an extension of the **MAX UDF** and is very similar to `max(col)`. Note that `max(struct())` returns only the top-ranking row. You cannot, for example, use it to return the top three rows.

Check out the examples below to learn how to optimize queries with `max(struct())`. Eg #1 is simple, while example #2 is more complex and Uber-specific. Both examples use `max(struct())`, but the same principles apply to `min(struct())`.

#### Eg #1
Say we have two tables: DEPT and EMP. Dept holds information about departments at a company and Emp holds information about employees:
```
TABLE DEPT (
  Deptno int,
  Dname string,
  Loc string
)

TABLE EMP (
  Empno int,
  Empname,
  Salary decimal(10, 2),
  Deptno int,
  Hire_date date
)
```














































































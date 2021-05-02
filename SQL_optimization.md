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

### 3 Avoid timeouts on long running queries
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

If you want to query entire tables and receive complete results, set: hive.strict.checks.large.query=true or hive.mapred.mode=nonstrict.












































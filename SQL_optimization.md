# SQL optimization

This guide contains tips on how to optimize Hive query performance.
## I. Tips for optimizing queries
Contents:
0. Monitor query performance: For Hive and Presto queries, open Attis and go to the Queries view.
1. Limit Query Results
(1). Use modeled tables whenever possible
2. Include as many filters as possible
3. Avoid timeouts on long running queries
4. Rewrite count(DISTINCT)
5. Avoid global ordering: Limit ORDER BY Queries
6. Distribute Data Correctly Using DISTRIBUTE BY
6. Static vs. Dynamic Partition Loading
7. Avoid Too Fine-Grained Partitions
7. Use Hive on Spark when possible
8. Use max/min(struct()) to find the first/last ranking row by value
9. Use json_tuple instead of get_json_object

**Tips for Optimizing Joins**
- Know What Size Tables You’re Joining
- Use MAPJOIN to Join a Small and Large Table
- Create a Subset to Join two Large Tables
- Put the largest table last in the JOIN Statement
- Avoid nulls in inner joins
- Magellan (for spatial joins)



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
We want to find the name of the highest-paid employee in each department. We could do this using the analytical function, row_number(), as in the query below:
```
SELECT name, salary FROM
  (SELECT name,
          Salary,
          row_number() OVER (PARTITION BY deptno ORDER BY salary DESC) rk
  FROM EMP) T
WHERE rk = 1;
```
This query groups, orders, and filters data, making it very expensive to run. We don’t need to rank every single row if we are only looking for the single, top-ranking row. We can find the highest paid employees in each department much more efficiently using max(struct()), as in the query below:
```
SELECT st.col2 AS name, st.col1 AS salary
FROM
  (SELECT MAX(STRUCT(salary, name) AS st
  FROM EMP
  GROUP BY deptno) T
```
**Note**: .col1, .col2, etc. let us access specific columns in the struct. Indices start at 1.

This query achieves our original goal of finding the highest-paid employee in each department. Additionally, it will run in less time because we’ve eliminated the ordering and filtering aspects by using max(struct()) instead of row_number().

#### Eg #2
Let’s look at a more complicated, Uber-specific example. Say we want to find the latest trip records for trips that happened after January 2nd, 2018. The unoptimized query below uses `row_number()` to do this:

```
CREATE TEMPORARY TABLE tmp_cb_bills stored AS parquet AS
  SELECT * FROM (
    SELECT `_row_key` AS uuid,
           BASE.job_uuids[0] AS trip_uuid,
           BASE.payer_type AS payer_type,
           BASE.payer_uuid AS payer_uuid,
           BASE.order_id AS order_id,
           BASE.bill_type AS bill_type,
           BASE.created_at AS created_at,
           BASE.related_bill_uuids[0] AS related_bill_uuid,
           BASE.job_types[0] AS job_type,
           BASE.territory_id AS territory_id,
           row_number() over (partition BY `_row_key` ORDER BY BASE.`_created_at` DESC) rnk
    FROM rawdata.schemaless_trifle_client_bills_cells
    WHERE datestr >= date_sub('2018-01-02 03:22:37',120)
          AND BASE IS NOT NULL
          AND BASE.job_uuids[0] IS NOT NULL ) x
  WHERE rnk = 1
  DISTRIBUTE BY (uuid);
  ```
This query will run very slowly and cost Uber a lot of money. `row_number()` is extremely costly when working with large amounts of data. We can get the same results in less time by optimizing this query to use `max(struct())` and a `GROUP BY` statement. See the optimized query below:
```
CREATE TEMPORARY TABLE tmp_cb_bills_1 stored AS parquet AS
SELECT uuid, st.col2 AS trip_uuid, st.col3 AS payer_type, st.col4 AS payer_uuid, st.col5 AS order_id, st.col6 AS bill_type, st.col7 AS created_at, st.col8 AS related_bill_uuid, st.col9 AS job_type, st.col10 AS territory_id, 1 FROM (

  SELECT `_row_key` AS uuid,
    max(struct(BASE.`_created_at`,
    BASE.job_uuids[0],
    BASE.payer_type,
    BASE.payer_uuid,
    BASE.order_id,
    BASE.bill_type,
    BASE.created_at,
    BASE.related_bill_uuids[0],
    BASE.job_types[0],
    BASE.territory_id
  )) AS st

FROM rawdata.schemaless_trifle_client_bills_cells
WHERE datestr >= date_sub('2018-01-02 04:35:05',120) AND BASE IS NOT NULL AND BASE.job_uuids[0] IS NOT NULL
GROUP BY `_row_key`) X

DISTRIBUTE BY (uuid);
```
This query runs much faster and avoids expensive operations like ordering, filtering and joining. The differences in our optimized query are summarized below:
- The column we want to order by (_created_at) becomes the first column in the struct construction (e.g. max(struct(BASE.`_created_at`, ...).
- Instead of ordering each partition (row_number() over (partition BY `_row_key` ORDER BY BASE.`_created_at` DESC) rnk) , we group by _row_key.
- The rnk=1 condition is eliminated because we’ve already found the maximum value.
- We rename the struct’s nested columns in our temporary table back to their original names (e.g. st.col3 AS payer_type).

### 9. Use json_tuple instead of get_json_object
Use `json_tuple` to retrieve more than one key from a single `JSON` string. `json_tuple` parses the string only once, making it more efficient than `get_json_object`.

For example, see the unoptimized query below:
```
select a.timestamp,
get_json_object(a.appevents, '$.eventid'),
get_json_object(a.appenvets, '$.eventname')
from log a;
```
We can optimize it using json_tuple:
```
select a.timestamp, b.f1, b.f2
from log a
lateral view json_tuple(a.appevent, 'eventid', 'eventname') b as f1, f2;
```
## II. Tips for Optimizing Joins
### 1. Know What Size Tables You’re Joining
The first step to optimizing joins is always to check the size of the tables you’re joining.

People can estimate the scale of their queries if one or more large tables are involved. For more details about tables, please see the **Schemaless tables** and **Kafka tables**.

### 2. Use `MAPJOIN` to Join a Small and Large Table
**NOTE: Enable map side join (a.k.a in-memory join) as far as possible.**

Map join is a Hive feature that is used to speed up Hive queries. It lets a table be loaded into memory so that a join can be performed within a mapper without using a Map/Reduce step. If queries frequently depend on small table joins, using map joins speed up queries’ execution.

The following query joins a small table, dim.dim_city with a large table, hdrone.mezzanine_trips. Without any optimization, Hive will shuffle records in both tables with the same key through network for realizing the join operation. In other words, if a join operation introduces any reduce stages, it will be very expensive. Therefore, Hive supports an optimization strategy of broadcasting the small table to memories of mappers instead of shuffling, greatly improving join performance:

```
SELECT
    trips.base.city_id AS city_id,
    count(*) AS requested_trips,
    cities.timezone as timezone
FROM
    dim.dim_city as cities
    RIGHT OUTER JOIN hdrone.mezzanine_trips as trips
    ON cities.city_id = trips.base.city_id
WHERE
    trips.datestr = '2016-02-06'
GROUP BY
    trips.base.city_id
```
To enable the optimization, set the configurations below in QueryBuilder or Beeline:
```
set hive.auto.convert.join=true;
set hive.auto.convert.join.noconditionaltask=true;
set hive.auto.convert.join.noconditionaltask.size=50000000;
```
Notice that the size estimation of the small table is the config value of hive.auto.convert.join.noconditionaltask.size. Enlarging the configuration value could introduce “out of memory” errors if the value is pushed too much.

### 3. Create a Subset to Join two Large Tables

#### TL;DR
- If possible, avoid joining two big tables! If you have to do it, keep reading.
- Joins occur before the ‘WHERE’ clause. We suggest you use the ‘WITH’ statement to prepare the joined table first.
- Split raw queries into multiple steps, with each step joining a small with a large table.This way, you can take advantage of MAPJOIN as far as possible.

The following example is a simplified version of a real query from fraud team which intends to find suspicious disparities between request location and event location of trips. It joins two large table, i.e. `hdrone.mezzanine_trips` and `hdrone.event_user`, with certain predicates.

**The query is NOT feasible in this raw format and will either take forever to run or throw an “out of memory” error:**

```
SELECT
    trips.`_row_key` as trip_uuid,
    trips.base.city_id as city_id,
    trips.base.driver_uuid as driver_uuid,
    trips.base.client_uuid as client_uuid,
    trips.base.request_at as request_at,
    trips.fare.total / trips.fare.usd_fx_rate as fare,
    max(esri.ST_DISTANCE(
        esri.ST_SETSRID(esri.ST_POINT(trips.base.request_lng, trips.base.request_lat), 4326),
        esri.ST_SETSRID(esri.ST_POINT(events.msg.location.lng, events.msg.location.lat), 4326)
    )) as request_dist
FROM
    hdrone.mezzanine_trips as trips
    JOIN hdrone.event_user as events
    ON trips.base.client_uuid = events.msg.rider_app.rider_id
WHERE
    trips.datestr = '2016-02-06' and events.datestr = '2016-02-06'
    and trips.base.status in ('completed', 'fare_split')
    and (trips.fare.total / trips.fare.usd_fx_rate) < 10000
    and events.msg.epoch_ms >
        (UNIX_TIMESTAMP(trips.base.request_at, 'yyyy-MM-dd\'T\'HH:mm:ss') - 45) * 1000
    and events.msg.epoch_ms <
        (UNIX_TIMESTAMP(trips.base.request_at, 'yyyy-MM-dd\'T\'HH:mm:ss') + 45) * 1000
GROUP BY
    trips.`_row_key`
```

The first caveat for joining two large tables is that Hive joins tables before applying predicates in the WHERE clause. Iif we put predicates in the same SELECT clause as the joins, Hive will join the tables before utilizing filtering conditions, making the operation more expensive than it needs to be. We want filters to be applied before the tables are joined. We can do this by preparing the table using a ‘WITH’ statement, and applying predicates before the join operation. See the following example:

```
with interested_trips as (
  SELECT
      trips.`_row_key` as trip_uuid,
      trips.base.city_id as city_id,
      trips.base.driver_uuid as driver_uuid,
      trips.base.client_uuid as client_uuid,
      trips.base.request_at as request_at,
      trips.base.request_lng as request_lng,
      trips.base.request_lat as request_lat,
      trips.fare.total / trips.fare.usd_fx_rate as fare
  FROM
      hdrone.mezzanine_trips as trips
  WHERE
      trips.datestr = '2016-02-06'
      and trips.base.status in ('completed', 'fare_split')
      and (trips.fare.total / trips.fare.usd_fx_rate) < 10000
```

#### Another example:

**Prepare a SMALL table with only required fields for joining with event_user:**
```
trips_with_join_condition as (
  SELECT
      trips.trip_uuid as trip_uuid,
      trips.client_uuid as client_uuid,
      trips.request_at as request_at,
  FROM
      interested_trips as trips
  ),
```
**Retrieve event locations via JOINING a SMALL table, trips, with a LARGE table, events:**
```
event_locations as (

  SELECT
      trips.trip_uuid as trip_uuid,
      events.msg.location.lng as event_location_lng
      events.msg.location.lat as event_location_lat
  FROM
      trips_with_join_condition as trips
      JOIN hdrone.event_user events
      ON trips.client_uuid = events.msg.rider_app.rider_id
          and events.msg.epoch_ms >
              (UNIX_TIMESTAMP(trips.request_at, 'yyyy-MM-dd\'T\'HH:mm:ss') - 45) * 1000
          and events.msg.epoch_ms <
              (UNIX_TIMESTAMP(trips.request_at, 'yyyy-MM-dd\'T\'HH:mm:ss') + 45) * 1000
  WHERE
      events.datestr = '2016-02-06'
),
```

**Prepare a SMALL table with only required fields for distance computation:**
```
trips_with_request_location as (
  SELECT
      trips.trip_uuid as trip_uuid,
      trips.request_lng as request_lng,
      trips.request_lat as request_lat
  FROM
      interested_trips as trips
),
```
**Compute max distance via JOINING a SMALL table, trips, with a LARGE table, events:**
```
trips_with_max_distance as (
  SELECT
      trips.trip_uuid as trip_uuid,
      max(esri.ST_DISTANCE(
          esri.ST_SETSRID(esri.ST_POINT(trips.request_lng, trips.request_lat), 4326),
          esri.ST_SETSRID(esri.ST_POINT(events.event_location_lng, events.event_location_lat), 4326)
      )) as request_dist
  FROM
      trips_with_request_location as trips
      JOIN event_locations as events
      ON trips.trip_uuid = events.trip_uuid
  GROUP BY
      trips.trip_uuid
)
```

**Produce the final result via JOINING a SMALL table, trips_distance, with a LARGE table, trips_info:**

```
SELECT
    trips_info.trip_uuid as trip_uuid,
    trips_info.city_id as city_id,
    trips_info.driver_uuid as driver_uuid,
    trips_info.client_uuid as client_uuid,
    trips_info.request_at as request_at,
    trips_info.fare as fare,
    trips_distance.request_dist as request_dist
FROM
    trips_with_max_distance as trips_distance
    JOIN interested_trips as trips_info
    ON trips_distance.trip_uuid = trips_info.trip_uuid
```

### 4. Put the largest table last in the JOIN Statement
When joining multiple tables on the same key, put the largest table last. **In every MapReduce stage of the join, the last table in the sequence is streamed through the reducers, whereas the others are buffered.** Therefore, we need a way to decrease the reducer memory needed to buffer the rows for a particular value of the join key. We can do this by organizing the tables such that the largest tables appear last in the sequence.

If the raw query joins a large table with a series of small tables (i.e. star schema, see here for more details), you should consider using MAPJOIN. If the joined tables include two or more large tables, as mentioned above, split the join operation into multiple joins of two tables.

#### 5. Avoid nulls in inner joins
Having nulls in inner joins skews the load because all the rows with NULL values end up in the same reducer.

**Good**
```
select count(*) from a join b where a.id = b.id and b.id is not null
```

**Bad**
```
select count(*) from a join b where a.id = b.id
```



















































































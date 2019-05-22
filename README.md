# SQL Conventions

## Data layer

* For SQL use [PostgreSQL](https://www.postgresql.org), it's the [most loved relational database (StackOverflow survey 2018)](https://insights.stackoverflow.com/survey/2018/#technology-most-loved-dreaded-and-wanted-databases) and it's a multi-model database (K/V store, FDW and much more). Any questions?

## Application layer

* If your API is only doing mainly data persistence use [Postgrest](https://postgrest.com) is the way to go and only implement the missing part in another process. You can then compose both API with the reverse-proxy.
* Otherwise, use a data-mapping library (e.g. [doobie](https://github.com/tpolecat/doobie)) not an ORM.

### Queries

* Don't use BETWEEN ([why](https://wiki.postgresql.org/wiki/Don%27t_Do_This#Don.27t_use_BETWEEN_.28especially_with_timestamps.29))

## Tables/Views

### Name

* singular ([why](https://launchbylunch.com/posts/2014/Feb/16/sql-naming-conventions/#singular-relations)), e.g. `team` not `teams`
* camelCase, even for `n-n` tables, never snake_case, it's reserved for PK and FK.

### Columns

* Column names in **camelCase**, e.g. `createdAt`
* Only use underscore for PK and FK columns e.g. (PK) `user_id`, (FK) `organization_id`
* NOT NULL by default, NULL is the exception (think of it as the [maybe Monad](https://github.com/chrissrogers/maybe#why))
* pas d'abbréviations des mots sauf pour des expressions bien connues et longue (e.g. "i18n")
(* pas de mots-clés réservé (par exemple `user` sur PGSQL).)
* utiliser des UUID en type de  PK & FK ([pourquoi](https://www.clever-cloud.com/blog/engineering/2015/05/20/why-auto-increment-is-a-terrible-idea/)), ne pas utiliser serial ([pourquoi](https://wiki.postgresql.org/wiki/Don%27t_Do_This#Don.27t_use_serial))
* `createdAt TIMESTAMPTZ DEFAULT now()` toujours gérer les temps (date, datetime, time, ...) avec timestamptz ([pourquoi](https://wiki.postgresql.org/wiki/Don%27t_Do_This#Don.27t_use_timestamp_.28without_time_zone.29)))
* `updatedAt TIMESTAMPTZ DEFAULT now()` unless you plan to leverage event-sourcing
* `deletedAt TIMESTAMPTZ DEFAULT NULL`:
  * unless you plan to leverage event-sourcing
  * don't forget to [`deletedAt`](http://stackoverflow.com/questions/8289100/create-unique-constraint-with-null-columns/8289253#8289253)
* Comment each column, explain your rational, explain your decisions, should be in plain english for internal use only
* Boolean columns must start with either `is` or `has`.
* [Ne pas utiliser char(n)](https://wiki.postgresql.org/wiki/Don%27t_Do_This#Don.27t_use_char.28n.29) [même pour des identifiants de taille fixe](https://wiki.postgresql.org/wiki/Don%27t_Do_This#Don.27t_use_char.28n.29_even_for_fixed-length_identifiers)

## Constraints


General rule is: `{tablename}_{columnname(s)}_{suffix}` (e.g. `tableName_columnNameA_pkey`) where the suffix is one of the following:
  * Primary Key constraint: `pk`
  * Foreign key: `fk`
  * Unique constraint: `key`
  * Check constraint: `chk`
  * Exclusion constraint: `exl`
  * Any other kind of index: `idx`

### PK - Primary Key

* `tableName_columnName_pk` in case of a single column PK
* `tableName_columnName1_columnName2_columnName3_pk` in case of multiple columns as primary key (`columnName1`, `columnName2`, `columnName3`)


### FK - Foreign key

* `tableNameFrom_column_tableNameTo_column_fk`

### Unique

* `tableNameFrom_column_key` in case of a single column unique constraint
* `tableName_columnName1_columnName2_columnName3_key` in case of multiple columns as unique (`columnName1`, `columnName2`, `columnName3`)

## Functions

### Name

They are 3 types of functions, `notiy` functions and `private` functions and `public` functions
- **notify**, format: notify[*SchemaName*][*TableName*][*Event*] (e.g. `notifyAuthenticationUserCreated(user_id)`): should only format the notification message underneath and use pg_notify. Beware of the [8000 characters limit](http://stackoverflow.com/a/41059797/745121), only send metadata (ids), data should be asked by workers through the API. If you really wish to send data then [pg_kafka](https://github.com/xstevens/pg_kafka) might be a better alternative.
- **private**, format: _[*functionName*] (e.g. `_resetFailedLogin`): must never be exposed through the public schema. Used mainly for consistency and business-rules
- **public**, format [*functionName*] (e.g. `logIn(email, password)`): must be exposed through the public schema.


## Types

Enum types should be in *singular*, in *camelCase**.

## Triggers

### Name

(translation in progress)

### Columns

* utiliser BNCF (au dessus de la 3NF) (cf normal form)

* leverage `using`, so instead of:

```sql
select <fields> from
  table_1
  inner join table_2
    on table_1.table_1_id =
       table_2.table_1_id
```

use:


```sql
select <fields> from
  table_1
  inner join table_2
    using (table_1_id)
```


* utiliser les enum PG qui sont des types
* use the right PostgreSQL types:

```
inet (IP address)
timestamp with time zone
point (2D point)
tstzrange (time range)
interval (duration)
```

* utiliser les tableaux si besoin (permet de gérer la notion "d'ordre" facilement)
* constraint should be inside your database as much as possible:

```sql
create table reservation(
    reservation_id uuid primary key,
    dates tstzrange not null,
    exclude using gist (dates with &&)
);
```

* use row-level-security to ensure R/U/D access on each table rows

([source](http://stackoverflow.com/questions/4107915/postgresql-default-constraint-names/4108266#4108266))

## Policies

### Name


### Zero-down time migrations

- [Best practices](https://medium.com/braintree-product-technology/postgresql-at-scale-database-schema-changes-without-downtime-20d3749ed680)


# Things to monitor

> Your cache hit ratio tells you how often your data is served from in memory vs. having to go to disk. Serving from memory vs. going to disk will be orders of magnitude faster, thus the more you can keep in memory the better. Of course you could provision an instance with as much memory as you have data, but you don’t necessarily have to. Instead watching your cache hit ratio and ensuring it is at 99% is a good metric for proper performance. ([Source](https://www.citusdata.com/blog/2019/03/29/health-checks-for-your-postgres-database/))

```sql
SELECT 
  sum(heap_blks_read) as heap_read,
  sum(heap_blks_hit)  as heap_hit,
  sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) as ratio
FROM 
  pg_statio_user_tables;
```


> Under the covers Postgres is essentially a giant append only log. When you write data it appends to the log, when you update data it marks the old record as invalid and writes a new one, when you delete data it just marks it invalid. Later Postgres comes through and vacuums those dead records (also known as tuples).
> All those unvacuumed dead tuples are what is known as bloat. Bloat can slow down other writes and create other issues. Paying attention to your bloat and when it is getting out of hand can be key for tuning vacuum on your database. ([Source](https://www.citusdata.com/blog/2019/03/29/health-checks-for-your-postgres-database/))

```sql
WITH constants AS (
  SELECT current_setting('block_size')::numeric AS bs, 23 AS hdr, 4 AS ma
), bloat_info AS (
  SELECT
    ma,bs,schemaname,tablename,
    (datawidth+(hdr+ma-(case when hdr%ma=0 THEN ma ELSE hdr%ma END)))::numeric AS datahdr,
    (maxfracsum*(nullhdr+ma-(case when nullhdr%ma=0 THEN ma ELSE nullhdr%ma END))) AS nullhdr2
  FROM (
    SELECT
      schemaname, tablename, hdr, ma, bs,
      SUM((1-null_frac)*avg_width) AS datawidth,
      MAX(null_frac) AS maxfracsum,
      hdr+(
        SELECT 1+count(*)/8
        FROM pg_stats s2
        WHERE null_frac<>0 AND s2.schemaname = s.schemaname AND s2.tablename = s.tablename
      ) AS nullhdr
    FROM pg_stats s, constants
    GROUP BY 1,2,3,4,5
  ) AS foo
), table_bloat AS (
  SELECT
    schemaname, tablename, cc.relpages, bs,
    CEIL((cc.reltuples*((datahdr+ma-
      (CASE WHEN datahdr%ma=0 THEN ma ELSE datahdr%ma END))+nullhdr2+4))/(bs-20::float)) AS otta
  FROM bloat_info
  JOIN pg_class cc ON cc.relname = bloat_info.tablename
  JOIN pg_namespace nn ON cc.relnamespace = nn.oid AND nn.nspname = bloat_info.schemaname AND nn.nspname <> 'information_schema'
), index_bloat AS (
  SELECT
    schemaname, tablename, bs,
    COALESCE(c2.relname,'?') AS iname, COALESCE(c2.reltuples,0) AS ituples, COALESCE(c2.relpages,0) AS ipages,
    COALESCE(CEIL((c2.reltuples*(datahdr-12))/(bs-20::float)),0) AS iotta -- very rough approximation, assumes all cols
  FROM bloat_info
  JOIN pg_class cc ON cc.relname = bloat_info.tablename
  JOIN pg_namespace nn ON cc.relnamespace = nn.oid AND nn.nspname = bloat_info.schemaname AND nn.nspname <> 'information_schema'
  JOIN pg_index i ON indrelid = cc.oid
  JOIN pg_class c2 ON c2.oid = i.indexrelid
)
SELECT
  type, schemaname, object_name, bloat, pg_size_pretty(raw_waste) as waste
FROM
(SELECT
  'table' as type,
  schemaname,
  tablename as object_name,
  ROUND(CASE WHEN otta=0 THEN 0.0 ELSE table_bloat.relpages/otta::numeric END,1) AS bloat,
  CASE WHEN relpages < otta THEN '0' ELSE (bs*(table_bloat.relpages-otta)::bigint)::bigint END AS raw_waste
FROM
  table_bloat
    UNION
SELECT
  'index' as type,
  schemaname,
  tablename || '::' || iname as object_name,
  ROUND(CASE WHEN iotta=0 OR ipages=0 THEN 0.0 ELSE ipages/iotta::numeric END,1) AS bloat,
  CASE WHEN ipages < iotta THEN '0' ELSE (bs*(ipages-iotta))::bigint END AS raw_waste
FROM
  index_bloat) bloat_summary
ORDER BY raw_waste DESC, bloat DESC
```

> Postgres makes it simply to query for unused indexes so you can easily give yourself back some performance by removing them ([Source](https://www.citusdata.com/blog/2019/03/29/health-checks-for-your-postgres-database/))

```sql
SELECT
            schemaname || '.' || relname AS table,
            indexrelname AS index,
            pg_size_pretty(pg_relation_size(i.indexrelid)) AS index_size,
            idx_scan as index_scans
FROM pg_stat_user_indexes ui
         JOIN pg_index i ON ui.indexrelid = i.indexrelid
WHERE NOT indisunique AND idx_scan < 50 AND pg_relation_size(relid) > 5 * 8192
ORDER BY pg_relation_size(i.indexrelid) / nullif(idx_scan, 0) DESC NULLS FIRST,
         pg_relation_size(i.indexrelid) DESC;
```


> pg_stat_statements is useful for monitoring your database query performance. It records a lot of valuable stats about which queries are run, how fast they return, how many times their run, etc. Checking in on this set of queries regularly can tell you where is best to add indexes or optimize your application so your query calls may not be so excessive. ([Source](https://www.citusdata.com/blog/2019/03/29/health-checks-for-your-postgres-database/))

```sql
SELECT query, 
       calls, 
       total_time, 
       total_time / calls as time_per, 
       stddev_time, 
       rows, 
       rows / calls as rows_per,
       100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent
FROM pg_stat_statements
WHERE query not similar to '%pg_%'
and calls > 500
--ORDER BY calls
--ORDER BY total_time
order by time_per
--ORDER BY rows_per
DESC LIMIT 20;
```

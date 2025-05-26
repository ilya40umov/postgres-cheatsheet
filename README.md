# postgres-cheatsheet

PostgreSQL cheat sheet

* [Database size](#database-size)
* [Size of each table](#size-of-each-table)
* [Size of each index](#size-of-each-index)
* [Running queries](#running-queries)
* [Locked queries](#locked-queries)
* [Query termination](#query-termination)
* [Specific Issues](#specific-issues)

### Database size

```sql
SELECT pg_size_pretty(pg_database_size('mydb'));
```

### Size of each table

```sql
SELECT 
    ut.schemaname || '.' || ut.relname AS table_name,
    c.reltoastrelid::regclass AS toast_table_name,
    pg_size_pretty(pg_total_relation_size(ut.relid)) AS total_size,
    pg_size_pretty(pg_relation_size(ut.relid)) AS table_size,
    pg_size_pretty(pg_relation_size(c.reltoastrelid)) AS toast_size,
    pg_size_pretty(pg_total_relation_size(ut.relid) - pg_relation_size(ut.relid)) AS index_size
FROM pg_catalog.pg_statio_user_tables ut LEFT JOIN pg_class c ON ut.relid = c.oid
ORDER BY pg_total_relation_size(ut.relid) DESC;
``` 

### Size of each index

```sql
SELECT
    pg_stat_all_indexes.relname,
    indexrelname,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size,
    pg_size_pretty(pg_relation_size(relid)) as table_size,
    pg_size_pretty(pg_total_relation_size(relid)) as total_relation_size,
    reltuples::bigint as table_row_count
FROM pg_stat_all_indexes JOIN pg_class ON pg_stat_all_indexes.relid = pg_class.oid
WHERE pg_stat_all_indexes.schemaname = 'public'
ORDER BY pg_total_relation_size(relid) DESC, pg_relation_size(indexrelid) DESC;
```

### Running queries

```sql
SELECT
    pid,
    usename,
    state,
    wait_event_type,
    wait_event,
    query,
    query_start,
    now() - query_start AS duration
FROM pg_stat_activity
WHERE (state IN ('active') OR wait_event_type = 'Lock')
   AND query NOT ILIKE '%pg_stat_activity%'
ORDER BY query_start;
```

### Locked queries

```sql
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocked.query_start AS blocked_query_start,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query,
    blocking.query_start AS blocking_query_start,
    now() - blocked.query_start AS blocked_duration,
    now() - blocking.query_start AS blocking_duration
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked
    ON blocked_locks.pid = blocked.pid
JOIN pg_catalog.pg_locks blocking_locks
    ON blocked_locks.locktype = blocking_locks.locktype
    AND blocked_locks.database IS NOT DISTINCT FROM blocking_locks.database
    AND blocked_locks.relation IS NOT DISTINCT FROM blocking_locks.relation
    AND blocked_locks.page IS NOT DISTINCT FROM blocking_locks.page
    AND blocked_locks.tuple IS NOT DISTINCT FROM blocking_locks.tuple
    AND blocked_locks.virtualxid IS NOT DISTINCT FROM blocking_locks.virtualxid
    AND blocked_locks.transactionid IS NOT DISTINCT FROM blocking_locks.transactionid
    AND blocked_locks.classid IS NOT DISTINCT FROM blocking_locks.classid
    AND blocked_locks.objid IS NOT DISTINCT FROM blocking_locks.objid
    AND blocked_locks.objsubid IS NOT DISTINCT FROM blocking_locks.objsubid
    AND blocked_locks.pid != blocking_locks.pid
JOIN pg_catalog.pg_stat_activity blocking
    ON blocking_locks.pid = blocking.pid
WHERE NOT blocked_locks.granted;
```

### Query termination

```sql
-- cancel a query by pid (e.g. kill)
select pg_cancel_backend(1234);

-- terminate a query by pid (e.g. kill -9)
select pg_terminate_backend(1234);
```

## Specific issues

### Get invalid indices

* [Problems with concurrent Postgres indexes](https://medium.com/carwow-product-engineering/problems-with-concurrent-postgres-indexes-and-how-to-solve-them-c57f7656c852)

```sql
-- get invalid indices
SELECT
    c.relname AS table_name,
	ic.relname AS index_name
FROM pg_catalog.pg_index i
JOIN pg_catalog.pg_class c ON c.oid = i.indrelid
JOIN pg_catalog.pg_class ic ON ic.oid = i.indexrelid
WHERE i.indisvalid = FALSE;
```

### Find all GIN indices

[Debugging random slow writes in PostgreSQL](https://iamsafts.com/posts/postgres-gin-performance/)

```sql
-- find all GIN indices
SELECT pg_get_indexdef(indexrelid) from pg_index
WHERE pg_get_indexdef(indexrelid) ~ 'USING (gin )';
```

### Autovacuum taking an access exclusive lock 

[PostgreSQL VACUUM taking an access exclusive lock](https://blog.summercat.com/postgres-vacuum-taking-an-access-exclusive-lock.html)

```sql
-- disable truncate operation on autovacuum 
ALTER TABLE mytable SET (vacuum_truncate = off, toast.vacuum_truncate = off);
```

```sql
-- list tables where options (e.g. autovacuum) are set
select relname, reloptions, pg_namespace.nspname
from pg_class join pg_namespace on pg_namespace.oid = pg_class.relnamespace
where pg_namespace.nspname = 'public' and reloptions is not null;
```

```sql
-- running VACUUM with TRUNCATE operation manually
VACUUM (TRUNCATE) mytable;
```

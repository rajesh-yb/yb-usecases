# Widening `varchar(5)` to `varchar(15)`

`ALTER TABLE ... ALTER COLUMN ... TYPE varchar(15)` backfills some indexes. It does not copy the table.

`REBUILT` means that index was backfilled: `pg_class.relfilenode` changed. Yugabyte built a new index and filled it from the rows already in the table. `not rebuilt` means `relfilenode` stayed the same.

Measured on a local YugabyteDB, database `yugabyte`, schema `public`, table `t`, 2,000,000 rows.

## Test case

### Create the table and load data

```sql
CREATE TABLE t (
    id int PRIMARY KEY,
    c  varchar(5),     -- column to change
    v  varchar(30),    -- other varchar column
    n  numeric(15,0),  -- other numeric column
    f  varchar(5)      -- INCLUDE column
);

INSERT INTO t
SELECT g, 'AB', 'x' || g, g, 'F'
FROM generate_series(1, 2000000) g;
```

### Create the indexes

```sql
CREATE INDEX i1_c_plain           ON t (c);
CREATE INDEX i2_c_where_v_text    ON t (v HASH, c ASC) WHERE v <> '' AND v IS NOT NULL;
CREATE INDEX i3_c_where_n_numeric ON t (n HASH, c ASC) INCLUDE (f) WHERE n <> 0 AND n IS NOT NULL;
CREATE INDEX i4_where_on_c        ON t (v) WHERE c <> '';
CREATE INDEX i5_expr_on_c         ON t (lower(c));
CREATE INDEX i6_c_include         ON t (v) INCLUDE (c);
CREATE INDEX i7_v_plain           ON t (v);
CREATE INDEX i8_v_where_v_text    ON t (v) WHERE v <> '' AND v IS NOT NULL;
CREATE INDEX i9_expr_on_v         ON t (lower(v));
```

### Capture `relfilenode` before the `ALTER`

```sql
CREATE TABLE snap AS
SELECT c.relname, c.oid, c.relfilenode
FROM pg_class c
JOIN pg_index i ON i.indexrelid = c.oid
WHERE i.indrelid = 't'::regclass;
```

### Alter the column

```sql
ALTER TABLE t ALTER COLUMN c TYPE varchar(15);
```

### Compare `relfilenode` after the `ALTER`

```sql
SELECT b.relname,
       b.relfilenode AS before,
       a.relfilenode AS after,
       CASE
         WHEN a.relfilenode IS DISTINCT FROM b.relfilenode THEN 'REBUILT'
         ELSE 'not rebuilt'
       END AS result
FROM snap b
LEFT JOIN (
    SELECT c.relname, c.relfilenode
    FROM pg_class c
    JOIN pg_index i ON i.indexrelid = c.oid
    WHERE i.indrelid = 't'::regclass
) a USING (relname)
ORDER BY 1;
```

## Result

| Index | Condition | Before | After | Result |
|---|---|---:|---:|---|
| `i1_c_plain` | Key `(c)`. No `WHERE`. | 16389 | 16389 | not rebuilt |
| `i2_c_where_v_text` | Key `(v, c)`. `WHERE v`. | 16390 | 16402 | REBUILT |
| `i3_c_where_n_numeric` | Key `(n, c) INCLUDE (f)`. `WHERE n`. | 16391 | 16404 | REBUILT |
| `i4_where_on_c` | Key `(v)`. `WHERE c`. | 16392 | 16401 | REBUILT |
| `i5_expr_on_c` | Key `lower(c)`. | 16393 | 16403 | REBUILT |
| `i6_c_include` | Key `(v) INCLUDE (c)`. No `WHERE`. | 16394 | 16394 | not rebuilt |
| `i7_v_plain` | Key `(v)`. No `WHERE`. | 16395 | 16395 | not rebuilt |
| `i8_v_where_v_text` | Key `(v)`. `WHERE v`. `c` is not used. | 16396 | 16396 | not rebuilt |
| `i9_expr_on_v` | Key `lower(v)`. `c` is not used. | 16397 | 16397 | not rebuilt |
| `t_pkey` | Primary key `(id)`. | 16387 | 16387 | not rebuilt |

Four indexes were backfilled: `i2_c_where_v_text`, `i3_c_where_n_numeric`, `i4_where_on_c`, and `i5_expr_on_c`.

An index was backfilled when it is partial or an expression and it uses `c`. That includes a partial index with `c` in the key, a partial covering index with `c` in the key, a partial index whose `WHERE` is on `c`, and an expression index on `c`.

A plain index was not backfilled, including a plain index on `c`. A covering index that only `INCLUDE`s `c` and has no `WHERE` was not backfilled. A partial or expression index that does not use `c` was not backfilled. The primary key was not backfilled.

## Monitor the `ALTER`

Run this in a second session while the `ALTER` is running. In `ysqlsh`, add `\watch 2` on the next line to refresh it.

```sql
SELECT a.pid,
       a.state,
       a.wait_event_type,
       a.wait_event,
       p.phase,
       CASE
         WHEN p.index_relid = 0 THEN NULL
         ELSE p.index_relid::regclass::text
       END AS index_name,
       p.tuples_done,
       p.tuples_total,
       round(100.0 * p.tuples_done / NULLIF(p.tuples_total, 0), 1) AS pct,
       now() - a.query_start AS running_for
FROM pg_stat_activity a
LEFT JOIN pg_stat_progress_create_index p ON p.pid = a.pid
WHERE a.pid <> pg_backend_pid()
  AND (
        a.query ILIKE '%ALTER TABLE%public.t%'
        OR p.relid = 'public.t'::regclass
      );
```

`phase = backfilling` means an index is being filled from the existing rows. `tuples_done` and `tuples_total` are the progress of the index currently being filled.

`index_name` stays blank during this phase. `index_relid` is `0` until that backfill finishes, so the progress view does not name the index. The name comes from the `relfilenode` query after the `ALTER`.

The four backfills ran one after another. Each sample below is near the end of one index, and `running_for` steps forward for the next index:

| pid | state | wait_event_type | wait_event | phase | index_name | tuples_done | tuples_total | pct | running_for |
|---:|---|---|---|---|---|---:|---:|---:|---|
| 46483 | active | Cpu | OnCpu_Active | backfilling | | 1996444 | 2000000 | 99.8 | 00:01:11.142114 |
| 46483 | active | RPCWait | StorageFlush | backfilling | | 1952768 | 2000000 | 97.6 | 00:02:11.141726 |
| 46483 | active | RPCWait | StorageFlush | backfilling | | 1995776 | 2000000 | 99.8 | 00:03:05.144235 |
| 46483 | active | Cpu | OnCpu_Active | backfilling | | 1987584 | 2000000 | 99.4 | 00:04:05.142608 |

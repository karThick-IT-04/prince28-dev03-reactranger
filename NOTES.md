# Engineering notes: ReactRanger

## D03 · PostgreSQL

### Migrations (upgrade, downgrade, upgrade)

```text
paste the output of:
uv run alembic downgrade -1
Running downgrade 6cc8f2122547 -> 940f4d35f303, add gin index on metadata

uv run alembic current
940f4d35f303

uv run alembic upgrade head
Running upgrade 940f4d35f303 -> 6cc8f2122547, add gin index on metadata

uv run alembic history
940f4d35f303 -> 6cc8f2122547 (head), add gin index on metadata
<base> -> 940f4d35f303, create documents and pages
```

### The loader is idempotent

```text
run 1: records=20 inserted=20 updated=0 unchanged=0
run 2: records=20 inserted=0 updated=0 unchanged=20
run 3 (after editing one record): records=20 inserted=0 updated=1 unchanged=19
```

### EXPLAIN for ?page=VALUE

```text
1) What the planner chooses on its own:
Seq Scan on documents  (cost=0.00..11.00 rows=1 width=36) (actual time=0.589..0.590 rows=1.00loops=1)
  Filter: (metadata @> '{"page": "reference/react/useState"}'::jsonb)
  Rows Removed by Filter: 19
  Buffers: shared read=1 dirtied=1
Planning:
  Buffers: shared hit=143 read=8 dirtied=1
Planning Time: 5.321 ms
Execution Time: 0.615 ms

2) Seq scan switched off, to prove the index can be used:
Bitmap Heap Scan on documents  (cost=12.81..16.82 rows=1 width=36) (actual time=1.643..1.644 rows=1.00 loops=1)
  Recheck Cond: (metadata @> '{"page": "reference/react/useState"}'::jsonb)
  Heap Blocks: exact=1
  Buffers: shared hit=2 read=2
  ->  Bitmap Index Scan on ix_documents_metadata_gin  (cost=0.00..12.81 rows=1 width=0) (actual time=1.605..1.605 rows=4.00 loops=1)
        Index Cond: (metadata @> '{"page": "reference/react/useState"}'::jsonb)
        Index Searches: 1
        Buffers: shared hit=1 read=2
Planning:
  Buffers: shared hit=1
Planning Time: 0.080 ms
Execution Time: 1.689 ms
```

Why did the planner choose a Seq Scan on my table? (2 lines, your own words)
My table contains only 20 records, so scanning the whole table is inexpensive. PostgreSQL estimates that a sequential scan is cheaper than using an index.

When would it switch to the GIN index? (1 line)
When the table grows and a metadata filter is selective enough that using the GIN index is estimated to be cheaper than scanning the entire table.

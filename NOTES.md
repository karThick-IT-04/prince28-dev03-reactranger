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


## D04 · Docker, Compose and tests

### The image

```text
IMAGE             ID             DISK USAGE   CONTENT SIZE
reactranger:dev   ca8f65e7cd9b    300MB        70MB

uid=999(app) gid=999(app) groups=999(app)
```

- Size: 300 MB (target: under 400 MB)
- First build: 53.2 s. After changing one line of code: 21.8 s.
- Which step said CACHED, and why? (1 line, your own words)
The `uv sync --locked --no-install-project --no-dev` step showed `CACHED` because only the application source code changed, while `uv.lock` and `pyproject.toml` remained unchanged.

### The stack

```text
NAME                    IMAGE                    SERVICE   STATUS
reactranger-api-1       reactranger-api          api       Up 3 minutes (healthy)
reactranger-db-1        pgvector/pgvector:pg17   db        Up 3 minutes (healthy)
reactranger-migrate-1   reactranger-migrate      migrate   Exited (0) 3 minutes ago
reactranger-redis-1     redis:7-alpine           redis     Up 3 minutes (healthy)

First up:
records=20 inserted=20 updated=0 unchanged=0

After down + up:
records=20 inserted=0 updated=0 unchanged=20
```

### Breaking the database on purpose

```text
db stopped:  /health → {"status":"ok","app":"reactranger","env":"local"}
db stopped:  /ready  → {"status":"not_ready","checks":{"database":false}}
db started:  /ready  → {"status":"ready","checks":{"database":true}}
```

Why should a load balancer stop sending traffic here but not restart the app? (2 lines)
The load balancer should stop sending traffic because the application cannot handle database-dependent requests while the database is unavailable.
It should not restart the application because the application is still alive and becomes ready again when the database recovers.



### Tests and coverage

```text
uv run poe test              → 16 passed in 0.60s
uv run poe test-integration  → 7 passed in 6.76s
coverage TOTAL, unit only    → TOTAL    254    36    26    2    83%
coverage TOTAL, with integration tests → TOTAL    254    15    26    2    92%
```

Two files or lines my coverage report says are never run, and whether they matter: (2 lines)
api/middleware.py (lines 22–24): These lines are not covered. They matter because middleware behavior should be tested.
db/repository.py (lines 32, 36, 66): These lines are not covered. They matter because untested database operation paths may contain bugs.

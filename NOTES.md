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


## D05 · CI/CD and going live

- Live URL: https://reactranger.onrender.com
- CI run for pull request 1: https://github.com/karThick-IT-04/prince28-dev03-reactranger/actions/runs/44700991279
- First deploy that ran by itself: https://github.com/karThick-IT-04/prince28-dev03-reactranger/actions/runs/44708170368
- Merge clicked at 2:36 pm → "Smoke test passed" at 2:39 pm → lead time: 3 min

### Smoke test output from that CD run

```text
ok   /health -> 200, version 82577d77f3770974b2b228f5d6dd18a0e39b8d32
ok   /ready -> 200 {'status': 'ready', 'checks': {'database': True}}
ok   /v1/documents?limit=1 -> 200
Smoke test passed
```

### Rollback drill

- Rolled back to sha 4d3fa684ebb29aac82b016722b9000be605d45e9 : started 10:49 pm, smoke passed 10:50 (1 min)
- Rolled forward to sha 82577d77f3770974b2b228f5d6dd18a0e39b8d32 : 1 min
- During the rollback, the database stayed on the newer schema. Why is that safe here?
The older application passed the smoke test against the newer database schema, indicating compatibility for the tested endpoints.
The rollback only changed the Docker image; it did not reverse migrations or delete existing database data.

### Where my secrets live

- Neon string: GitHub Secret DATABASE_URL + Render environment. Not in Git, not in the image.
- Render deploy hook: GitHub Secret RENDER_DEPLOY_HOOK only.

### Monitoring

- UptimeRobot checks /health every 5 minutes; alerts go to my email.
- One thing that would make my live app faster on its first request: Use a Render instance that stays awake instead of spinning down when idle.


## D06 · Collecting the corpus

- Start target: Learn + Reference sections (about 300 pages)
- Collected today: 192 files, 1134 pages, 7.3 MB (from DATASET.md)
- Doors used: URL list / Git repository

### Licence check

| Source | Licence or terms, in my words | May I show full text in answers? |
| react-dev | React documentation is licensed under CC BY 4.0. I must provide attribution and link to the original source. |Prefer short attributed excerpts and links rather than reproducing entire pages. |
| react-docs-repo | React documentation content uses CC BY 4.0, while React code uses the MIT licence. I must preserve attribution and applicable licence notices. | Prefer short attributed excerpts and links rather than reproducing entire pages. |

### robots.txt

- Site: https://react.dev/robots.txt . Its robots.txt says (2 lines):
User-agent: * — the rules apply to all web crawlers.
Disallow: is empty — no paths are blocked by these rules.
- Was anything blocked for my bot? What did I do about it?
No. The downloader reported 0 blocked requests. I respected robots.txt and used the configured request delay.

### Kill and resume

- Run 1: I pressed Ctrl+C after 4 files
- Run 2: downloaded=6 skipped=4
- Run 3: downloaded=0 skipped=10

### Capacity (from docs/CAPACITY.md)

- Tokens per page, measured:  750 (estimated using 4 characters per token)
- Full corpus: 0.60 M tokens, 1725 chunks, embedding about Rs 10, vectors 5 MB
- The number that surprised me most, and why (2 lines):
The estimated embedding cost was only Rs 10 for the 800-page target.
This showed me that storing and embedding the initial React documentation corpus could be inexpensive.

### One thing that went wrong today, and how I fixed it

The Git repository fetch initially failed because of a network/DNS issue. After the connection worked, I successfully fetched 182 React documentation files. I also learned that the URL-list and repository fetches needed different source names to avoid a directory collision.

## D07 · Parser comparison

<!-- parsers:start -->
| File | Parsers (A / B) | Pages | A found | B found | Time (A / B) | Better, why |
|---|---|---|---|---|---|---|
| useeffect-4d23b49b.html | trafilatura / strip tags | 14 | 35,290 chars, 27 headings, 0 tables | 44,451 chars, 0 headings, 0 tables | 1.35 s / 0.01 s | Trafilatura: meaningful headings and readable examples verified. |
| useactionstate-f7733182.html | trafilatura / strip tags | 11 | 24,542 chars, 29 headings, 0 tables | 33,068 chars, 0 headings, 0 tables | 0.41 s / 0.01 s | Trafilatura: preserves 29 headings with less text. |
| usememo-210c042a.html | trafilatura / strip tags | 11 | 27,975 chars, 13 headings, 0 tables | 33,753 chars, 0 headings, 0 tables | 0.25 s / 0.01 s | Trafilatura: better section structure. |
| usestate-d02f06ba.html | trafilatura / strip tags | 8 | 21,356 chars, 28 headings, 0 tables | 27,265 chars, 0 headings, 0 tables | 0.71 s / 0.01 s | Trafilatura: retains 28 headings. |
| you-might-not-need-an-effect-8b892800.html | trafilatura / strip tags | 14 | 36,242 chars, 11 headings, 0 tables | 40,469 chars, 0 headings, 0 tables | 0.41 s / 0.03 s | Trafilatura: readable explanations and examples verified. |
| form-8dc2f808.html | trafilatura / strip tags | 8 | 19,109 chars, 23 headings, 0 tables | 25,487 chars, 0 headings, 0 tables | 0.38 s / 0.01 s | Trafilatura: preserves section structure with less text. |
| usereducer-be89a551.html | trafilatura / strip tags | 6 | 15,894 chars, 18 headings, 0 tables | 21,564 chars, 0 headings, 0 tables | 0.27 s / 0.01 s | Trafilatura: retains 18 headings. |
| useref-1b61fba5.html | trafilatura / strip tags | 3 | 7,826 chars, 15 headings, 0 tables | 12,735 chars, 0 headings, 0 tables | 0.16 s / 0.00 s |  Trafilatura: meaningful headings and code examples verified. |
| uselayouteffect-41d47fa5.html | trafilatura / strip tags | 5 | 10,259 chars, 11 headings, 0 tables | 14,083 chars, 0 headings, 0 tables | 0.15 s / 0.00 s | Trafilatura: better heading preservation.|
| sharing-state-between-components-b2474c67.html | trafilatura / strip tags | 4 | 10,534 chars, 10 headings, 0 tables | 14,101 chars, 0 headings, 0 tables | 0.15 s / 0.00 s | Trafilatura: retains useful section metadata. |
<!-- parsers:end -->


### D07 · Parsing my corpus

- Files in the manifest: 192 · parsed: 192 · failed: 0 · pages in Postgres: 1247
- Parsing coverage: 100 % of files

### Failures by reason

| Reason | Files | What I did |
|---|---|---|
| None | 0 | Verified that the parse_failures table is empty. |

- 0 failures out of 192 files.
- The one I fixed: Not applicable because no parsing failures occurred.
- No retry was required.

### What I saw when I read my parsed pages

- A page where the heading path is right (document, page, path):
  Document: useEffect
  Page: 3
  Path: ["useEffect", "Usage", "Connecting to an external system"]
- Something that came out wrong (document, page, what):
Document: Consolidate state logic with a reducer
  Page: 23
  MDX anchor comments remain in titles and heading paths.
  Some pages begin inside JavaScript code blocks.
  MDX components such as Sandpack and Hint remain in the text.
- A table: Markdown grid or loose text? (document, page):
  No tables were detected in the ten compared HTML documents.
  Table extraction could not be evaluated using these samples.

### My parser choice

- For PDFs: Not applicable because my corpus contains no PDF files.
- For HTML I use Trafilatura because it preserves heading structure.
  For useEffect, it extracted 35,290 characters and 27 headings,
  compared to 44,451 characters and 0 headings with strip tags.

- A kind of document in my corpus that neither parser handles well:
 React MDX documents containing interactive code examples,anchor comments and custom components.

### One thing that went wrong today, and how I fixed it

- Ruff reported formatting issues in models.py after
  adding the Page headings and ParseFailure models.
- I Fixed using:
  uv run ruff format src/reactranger/db/models.py
- Ran lint again successfully.
- All 52 unit tests passed.


### D08 · LLM gateway

- Primary: gemini-3.5-flash (Gemini) · fallback: openai/gpt-oss-20b (Groq)
- First question on my laptop: first words after 6.18 s, total 8.90 s, tokens in 165 / out 131, cost Rs ___
- Same question on my live URL (second, awake request): first words after ___ s. Day 1 target: under 1.5 s

### Tokens: English against Tamil

| Message | tokens in |
|---|---|
| "Hi" (baseline: mostly my system prompt) | 159 |
| The English sentence | 167 |
| The same sentence in Tamil | 175 |

- The Tamil message needs about 2 times the tokens of the English one.

### What the model does without my documents

- Question 6 ("Why is our client's website slow on their office Wi-Fi?"): It declined the question because it was outside its React subject area. Its exact words during the retest: "This question is outside ReactRanger's scope, as I only answer questions related to React and the official React documentation." It did not invent a cause, but it did not explicitly acknowledge that it lacked sufficient information to diagnose the problem.
- A detail it got wrong or invented (question → what it said → what is true):
"What is the difference between useEffect and useLayoutEffect?" → It said useEffect runs after the browser paints → React generally allows painting first, but interaction-triggered effects may run before painting.
Another incorrect detail: Groq described React.memo as performing a deep comparison, whereas it uses shallow prop comparison by default.
- The off-topic question: did it decline in one sentence?
"Who won the 2011 cricket world cup?" — It declined the unrelated question.

### Fallback and circuit breaker

- With the primary broken: provider Groq , fallback YES , first words after 19.96 s
- The question at which "llm call gemini … error" stopped appearing in the log: number 4.
All four consecutive requests successfully used Groq.
After the 30-second cooldown, Gemini was attempted again, returned HTTP 404, and Groq successfully answered.
Restored gemini-3.5-flash and restarted the server. Gemini answered successfully with fallback=no.

### The bill

- /v1/usage today: calls 23 , errors 4 , fallbacks 5 , cost Rs 0.0511
- Average per answered question: Rs 0.002689 (my Day 1 target: under Rs 0.50)
The recorded average is below the target, but gemini-3.5-flash is missing from the pricing table. Therefore, the cost target cannot be conclusively verified.

### One thing that went wrong today, and how I fixed it

The original Gemini model (gemini-3.8-flash) encountered a timeout and HTTP 503 due to provider unavailability. I switched to gemini-3.5-flash using the LLM_PRIMARY_MODEL environment variable and verified that both Gemini and Groq worked successfully.

The fallback mechanism, circuit breaker, live streaming and usage endpoint were verified successfully. Response latency and the missing pricing entry remain areas for improvement.

## D09 · Prompt versions compared

<!-- prompts:start -->

| Prompt version | Correct | Accuracy | Invalid replies | Avg tokens in |
|---|---|---|---|---|
| classify_v1 | 10/20 | 50% | 0 | 100 |
| classify_v2 | 17/20 | 85% | 0 | 264 |
| classify_v3 | 18/20 | 90% | 0 | 375 |
| classify_v4 | 20/20 | 100% | 0 | 410 |

<!-- prompts:end -->


### D09 · Prompts and structured output

- My best router prompt: classify_v4 with 20 / 20
- What I changed in v4, and what happened to the score: I improved the personal label definition to include application-specific, client-specific, network, device and runtime problems that general React documentation cannot answer. The accuracy increased from 90% to 100%.
- The question my best version still gets wrong (line, wanted, got), and why I think so:None. All 20 questions were classified correctly in v4.
- A label in my test set that I changed after seeing the results (or "none"):None
- Input tokens per question: v1 100 , best version 410 . Is the extra accuracy worth it?Yes. Although v4 uses 4.1 times more input tokens, its accuracy improved from 50% to 100% on my test set.

### My contract (DocMeta)

- Fields I kept from my project card:section, api_name, react_version and is_legacy.
- Fields I deleted, and why they make no sense per document:I removed page because the document title and source already identify the documentation page.
- The field I tightened, and how:I restricted is_legacy to a list containing at most one value: legacy, modern or mixed. An empty list represents an unknown status.

### Extraction run

- Prompt version: extract_v1 · documents: 81 ok, atleast 3 repaired, 0 failed (of 192 parsed)
- Not finished? Why, and how many are left:111 documents remain because the LLM providers repeatedly encountered HTTP 429 quota limits. I reduced the processing rate to 6 documents per minute, but the quota issue persisted. The extraction is resumable and has exceeded the minimum requirement of 50 documents.
- Most common failure reason, and what I did about it:Provider rate limits interrupted extraction. Validation issues included summaries exceeding 300 characters and a section list exceeding 8 items. The automatic repair mechanism corrected the observed validation errors.
- A record that was valid but wrong (document, field, what it said, what is true):Error-boundaries (document ID 89), summary field. It stated that "use hooks suspend rather than throw." The original documentation specifically explains that the use API suspends component execution when it encounters a pending promise.
- Cost at list price: Rs 2.9292 in total, Rs 0.0362 per document.The actual total may be higher because some usage logging operations failed during a database connection interruption.

### One thing that went wrong today, and how I fixed it

During metadata extraction, a DNS resolution error interrupted the database connection. I verified the saved extraction progress in Neon and restarted the script without reprocessing completed documents. I also reduced the processing rate to handle provider rate limits. The extraction successfully reached 81 documents with zero recorded failures, although 111 documents remain pending due to quota limits.


### D10 · Queue and worker

- First run: 10 documents queued, finished in 52 s. Done 10 , skipped 0 , failed 0
- Pace: 5.2 s per document (INGEST_RPM = 12 , INGEST_CONCURRENCY = 2 )

### Kill test

- Queued: 20. At the kill: queued 13 , running 2 , done 5 , failed 0
- After the new worker: done 19 + skipped 0 + failed 1 = 20 (must be 20)
- Documents in the database: 192 , different ids: 192 (must be equal)
- The interrupted items came back after 336 s. Why that long:including the delay before restarting the worker. Exact recovery time was not measured. Redis job locks need to expire before interrupted items can be processed again.
- Why nothing was doubled (one sentence, in my own words):The worker checks existing document IDs before inserting documents again, so retrying a job does not create duplicate documents.

### Dead-letter list

| Stage (parse / extract / download / gave up) | Items | What I did |
|---|---|---|
| Gave up (extract) | 1 | Gemini circuit breaker was open and Groq returned HTTP 429. Waited for the rate limit to reset and ran the retry command. The document completed successfully in 10 seconds. |

### Live ingestion

- ReactRanger on Render: 5 of 5 documents done, 0 failed. The reasons: All documents completed successfully in 71 seconds, producing 67 pages.
- A request without the token gets status 401 Unauthorized (Missing or wrong X-Admin-Token header).git status --short

### Week 2 against Monday's estimate

- docs/CAPACITY.md said: 1134 pages, 850582 tokens. docs/INGEST_REPORT.md says: 1247 pages, 792803 tokens
- The difference, and why: Pages increased by 113 (9.96%), while estimated tokens decreased by 57,779 (6.79%). This is likely due to differences between Monday's character-based estimates and the parser's actual page splitting and text processing.

### One thing that went wrong today, and how I fixed it

During the kill test, one document failed after three attempts because of LLM rate limits. I inspected the dead-letter list, waited for the rate limit to reset, and retried the failed document using uv run python scripts/ingest.py --retry-failed. It completed successfully in 10 seconds. The database contained 192 documents with 192 distinct external IDs, confirming that no duplicate documents were created.

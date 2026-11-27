# Ingestion report

<!-- generated:start -->
_Generated 27 Nov 2026 14:58 from the database._

| What | Count |
|---|---|
| Documents in the database | 192 |
| Pages | 1247 |
| Tokens of text (characters ÷ 4) | 792,803 |
| Files the parser gave up on | 0 |
| Documents with an extracted record | 141 |
| Documents whose extraction failed | 0 |

| Queue | Count |
|---|---|
| Ingest jobs started | 5 |
| Items finished (done or skipped) | 60 |
| Items that needed more than one try | 11 |
| Items in the dead-letter list | 1 |

Model calls for extraction: 237 · tokens in 297,721 · tokens out 91,595 · cost at list price Rs 4.30

Last job `ec88b0a08d6f467b8f3c238e71e671d4`: 1 requested, 1 ok, 0 failed, 0.2 minutes.

Dead-letter list by stage: gave up after 3 tries 1
<!-- generated:end -->

## What I learnt

- Background ingestion can recover after a worker crash without losing job items or creating duplicate documents.
- LLM rate limits can cause ingestion failures, but the dead-letter list and retry mechanism help recover failed documents.
- The database contains 192 documents and 1,247 pages, with 141 documents successfully enriched with metadata.

# Ingestion report

<!-- generated:start -->
_Generated 28 Nov 2026 17:09 from the database._

| What | Count |
|---|---|
| Documents in the database | 193 |
| Pages | 1256 |
| Tokens of text (characters ÷ 4) | 799,080 |
| Files the parser gave up on | 0 |
| Documents with an extracted record | 193 |
| Documents whose extraction failed | 0 |

| Queue | Count |
|---|---|
| Ingest jobs started | 7 |
| Items finished (done or skipped) | 66 |
| Items that needed more than one try | 12 |
| Items in the dead-letter list | 1 |

Model calls for extraction: 323 · tokens in 416,452 · tokens out 119,229 · cost at list price Rs 5.58

Last job `86d56ea01aab40a09c448a3f512b161e`: 1 requested, 1 ok, 0 failed, 0.1 minutes.

Dead-letter list by stage: gave up after 3 tries 1
<!-- generated:end -->

## What I learnt

- Background ingestion can recover after a worker crash without losing job items or creating duplicate documents.
- LLM rate limits can cause ingestion failures, but the dead-letter list and retry mechanism help recover failed documents.
- The database contains 193 documents and 1,256 pages, with all 193 documents successfully enriched with metadata.The previously failed ingestion item was successfully retried on the live application in 7 seconds. The dead-letter count remains 1 despite the successful retry.

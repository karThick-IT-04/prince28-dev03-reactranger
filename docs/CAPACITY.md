# Capacity plan: ReactRanger

Full corpus target: + blog posts, changelog and legacy docs for comparison (800+ pages)

<!-- generated:start -->
## Measured on my start corpus (2026-11-23 16:21 UTC)

| What | Value |
|---|---|
| Files | 192 |
| Size on disk | 7.3 MB |
| Pages (PDF pages + 1 per 3,000 characters of text/HTML) | 1,134 |
| Characters of text | 3,402,327 |
| Estimated tokens (characters / 4) | 850,582 |
| Tokens per page | 750 |
| PDFs with almost no text (scanned? OCR is Day 7) | 0 |
| Files not measured (zip, xlsx, unreadable) | 0 |

## Estimate for the full corpus (target: 800 pages)

| What | How | Value |
|---|---|---|
| Scale factor | target / measured | x 0.7 |
| Tokens | measured tokens x scale | 0.60 M |
| Raw files | measured MB x scale | 5 MB |
| Chunks | tokens / 400 x 1.15 overlap | 1,725 |
| Embedding cost, one time | tokens x 1.15 x $0.15 per 1M | $0.10 = Rs 10 |
| Vector storage | chunks x 768 numbers x 4 bytes | 5 MB |
| Text in Postgres | characters x scale x 1.2 | 3 MB |

Assumptions: 4 characters per token, 400-token chunks, 768-dimension embeddings at $0.15 per 1M tokens, Rs 95 per US dollar.
<!-- generated:end -->

## What I conclude

1. Is embedding cost a problem for this product?
No. The estimated one-time embedding cost for the 800-page target is only $0.10 (approximately ₹10), so it is affordable.
2. Do the vectors and text fit in my free Postgres?
Estimated vector storage is 5 MB and text storage is 3 MB, totaling approximately 8 MB. This fits if my database has at least 8 MB of available storage; I still need to confirm the actual free-space limit on my database dashboard.
3. Which number above am I least sure about, and how will I measure it better?
The token and chunk estimates are least certain because they use character-based assumptions. I will measure actual tokens using the tokenizer and chunking pipeline in a later build lab.
4. What changes if the corpus grows 10 times?
Estimated embedding cost increases from $0.10 to $1.00, and combined vector and text storage increases from approximately 8 MB to 80 MB.
I would need to monitor database capacity, indexing performance, and ingestion time as the corpus grows.

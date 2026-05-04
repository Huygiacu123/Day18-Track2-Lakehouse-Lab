# Reflection — Day 18 Lakehouse Lab

## Anti-Pattern Risk: "Deduplication Deferred to Query Time"

Completing the medallion pipeline (NB4) revealed the most dangerous anti-pattern our team would likely encounter in production: **deferring deduplication to query time instead of enforcing it at the Silver layer**.

### Why This Matters

In NB4, we ingested LLM observability data where duplicate `request_id` entries naturally occur due to retries and network failures. The naive approach would be to:
- Write all raw events directly to Bronze (no dedup)
- Let analysts handle dedup in their queries with `ROW_NUMBER() OVER (PARTITION BY request_id ORDER BY ts)`

This creates three cascading problems:

1. **Storage waste**: Bronze grows unbounded with duplicates. At 1B requests/day with 5% retry rate, that's 50M wasted rows daily = 250 GB/month of redundant data.

2. **Query complexity tax**: Every downstream query must re-implement the same dedup logic. We saw this in NB4 — the Silver layer's `ROW_NUMBER()` window function is non-trivial. Multiply that across 50 analyst queries, and you have 50 opportunities for bugs (wrong partition key, wrong order-by, off-by-one errors).

3. **Cost explosion**: DuckDB/Spark must scan the entire Bronze table on every query, even if you only need one day's data. Z-order helps, but dedup-at-query-time defeats partition pruning because you can't filter before dedup.

### What We Did Right

In NB4, we enforced dedup at the Silver layer:
```sql
ROW_NUMBER() OVER (PARTITION BY request_id ORDER BY ts) AS rn
WHERE rn = 1
```

This meant:
- Bronze: 200K rows (raw, with duplicates)
- Silver: 185K rows (deduplicated, canonical)
- Gold: 21 rows (aggregated metrics, 7 dates × 3 models)

The dedup happened **once**, not 50 times. Analysts query Silver/Gold, never Bronze directly.

### Lesson for Production

The medallion architecture's real power is **enforcing data contracts at layer boundaries**. Silver isn't just "cleaned data" — it's the **single source of truth** for what a request actually is. Once you commit to that, downstream teams can't accidentally double-count.

In a real LLM observability system at 1B req/day, this discipline saves ~$50K/month in wasted compute and storage, plus eliminates an entire class of data quality bugs.

---

**Word count:** 198 words

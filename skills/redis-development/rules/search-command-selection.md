---
title: Choose the Right FT Command for the Job
impact: HIGH
impactDescription: Picking FT.SEARCH vs FT.AGGREGATE vs FT.HYBRID up front avoids a full rewrite later
tags: search, ft.search, ft.aggregate, ft.hybrid, command-selection, hybrid
description: Choose the Right FT Command for the Job
alwaysApply: true
---

## Choose the Right FT Command for the Job

The first decision before any query syntax is *which command to run*. Redis Search exposes three query commands with different design intents — picking the wrong one means rewriting the query later when you discover the command cannot express what you need.

| Command | Use when... | Mental model | Min. Redis |
|---------|-------------|--------------|------------|
| `FT.SEARCH` | Straightforward document retrieval — agent wants matching docs back. | Ready-to-use: returns matching documents directly. | 2.0 module / 8.0 built-in |
| `FT.AGGREGATE` | Faceting, analytics, computed fields, grouped or reshaped output. | Declarative result shaping: explicit `LOAD`, `APPLY`, `GROUPBY`, `REDUCE`, `SORTBY`. | 2.0 module / 8.0 built-in |
| `FT.HYBRID` | Relevance must blend lexical (text) and semantic (vector) ranking with explicit fusion. | Declarative hybrid retrieval: `SEARCH` leg + `VSIM` leg + `COMBINE` fusion (RRF or LINEAR). | **8.4.0** (Redis Open Source) |

**Correct:** Pick the command that matches the shape of the answer you need.

```
# FT.SEARCH — "give me matching bicycles"
FT.SEARCH idx:bicycle "@type:{mountain} @price:[100 500]"
    LIMIT 0 10
    RETURN 3 model brand price
    DIALECT 2

# FT.AGGREGATE — "what is the average price per brand?"
FT.AGGREGATE idx:bicycle "@type:{mountain}"
    GROUPBY 1 @brand
    REDUCE AVG 1 @price AS avg_price
    SORTBY 2 @avg_price DESC
    DIALECT 2

# FT.HYBRID (Redis ≥ 8.4.0) — "blend lexical relevance with vector similarity"
FT.HYBRID idx:bicycle
    SEARCH "mountain bicycle"
    VSIM @description_embeddings $query_vec
    KNN 2 K 10
    COMBINE RRF 10                          # RRF <count> — number of fused results to keep
    PARAMS 2 query_vec "<vector_blob>"
    DIALECT 2
```

**Version gate — FT.HYBRID requires Redis ≥ 8.4.0.** For older Redis, fall back to the pre-filter + KNN pattern via `FT.SEARCH` (see `search-vector-query.md`):

```
# Fallback for Redis < 8.4.0 — pre-filter + KNN inside FT.SEARCH
FT.SEARCH idx:bicycle "(@type:{mountain})=>[KNN 10 @description_embeddings $query_vec AS score]"
    SORTBY score
    PARAMS 2 query_vec "<vector_blob>"
    DIALECT 2
```

**When to use FT.HYBRID's COMBINE modes:**
- `COMBINE RRF` — Reciprocal Rank Fusion, rank-based fusion. Robust default; no tuning required.
- `COMBINE LINEAR ALPHA <a> BETA <b>` — weighted score blend. Use when you have calibrated scores and want explicit control over the lexical/vector trade-off.

**Incorrect:** Using `FT.SEARCH` and then post-processing in the client to compute groups, averages, or score fusion. That work belongs inside Redis — pushing it client-side defeats the index.

```python
# Bad: pulling raw docs and grouping in Python — defeats the index, blows up over the wire.
docs = r.ft("idx:bicycle").search("@type:{mountain}").docs
brands = collections.Counter(d.brand for d in docs)
```

**Decision tree:**

1. Need computed fields, grouping, or custom output shape? → `FT.AGGREGATE`.
2. Need blended lexical + vector ranking with explicit fusion? → `FT.HYBRID` (Redis ≥ 8.4.0).
3. Otherwise (including filter-narrowed vector search) → `FT.SEARCH`.

Cross-links:
- Syntax of the query expression: `search-query-syntax.md`
- KNN, range, and pre-filter vector queries: `search-vector-query.md`
- Aggregate pipeline stages: `search-aggregate-pipeline.md`

**Client mirrors:**

```python
# redis-py — STEP_START command_selection
# Mirrors doctests/search_quickstart.py
from redis import Redis

r = Redis()
# FT.SEARCH for retrieval
results = r.ft("idx:bicycle").search("@type:{mountain} @price:[100 500]")
# FT.AGGREGATE for grouped analytics
from redis.commands.search.aggregation import AggregateRequest, Reducers
agg = AggregateRequest("@type:{mountain}").group_by("@brand", Reducers.avg("@price").alias("avg_price"))
totals = r.ft("idx:bicycle").aggregate(agg)
# STEP_END
```

```java
// Jedis — STEP_START command_selection
// Mirrors SearchQuickstartExample.java
import redis.clients.jedis.UnifiedJedis;
import redis.clients.jedis.search.Query;
import redis.clients.jedis.search.aggr.AggregationBuilder;
import redis.clients.jedis.search.aggr.Reducers;

try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    jedis.ftSearch("idx:bicycle", new Query("@type:{mountain} @price:[100 500]"));
    AggregationBuilder agg = new AggregationBuilder("@type:{mountain}")
        .groupBy("@brand", Reducers.avg("@price").as("avg_price"));
    jedis.ftAggregate("idx:bicycle", agg);
}
// STEP_END
```

**Client mirrors — read exactly one:**
- For raw redis-py targets, read `references/clients/python-redis-py.md`.
- For Jedis (Java) targets, read `references/clients/java-jedis.md`.
- For RedisVL targets, read `references/clients/python-redisvl.md`.
- Do not read more than one client reference.

Upstream sources:
- redis-py: [`doctests/search_quickstart.py`](https://github.com/redis/redis-py/blob/master/doctests/search_quickstart.py)
- Jedis: [`SearchQuickstartExample.java`](https://github.com/redis/jedis/blob/master/src/test/java/io/redis/examples/SearchQuickstartExample.java)

Reference: [FT.HYBRID](https://redis.io/docs/latest/commands/ft.hybrid/), [FT.SEARCH](https://redis.io/docs/latest/commands/ft.search/), [FT.AGGREGATE](https://redis.io/docs/latest/commands/ft.aggregate/)

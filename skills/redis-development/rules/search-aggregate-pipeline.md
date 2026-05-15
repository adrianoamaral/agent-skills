---
title: Build FT.AGGREGATE Pipelines in the Correct Stage Order
impact: HIGH
impactDescription: Pipeline stage order determines correctness — wrong order silently returns wrong results
tags: search, ft.aggregate, pipeline, groupby, reduce, apply, sortby, load
description: Build FT.AGGREGATE Pipelines in the Correct Stage Order
alwaysApply: true
---

## Build FT.AGGREGATE Pipelines in the Correct Stage Order

`FT.AGGREGATE` runs stages in the order you write them, like a Unix pipeline. The canonical order is `LOAD → APPLY → FILTER → GROUPBY/REDUCE → APPLY → SORTBY → LIMIT`. Swapping stages doesn't error — it silently changes what your query computes. For paginating large aggregates, see `search-aggregate-cursors.md`.

**Correct:** Canonical pipeline against the Bicycle dataset — load needed fields, project a derived field, filter, group, sort, limit.

```
# Average price per brand for mountain bicycles, top 5 brands
FT.AGGREGATE idx:bicycle "@type:{mountain}"
    LOAD 3 @brand @price @condition
    APPLY "@price * 0.9" AS sale_price
    FILTER "@condition == 'new'"
    GROUPBY 1 @brand
        REDUCE COUNT 0 AS bike_count
        REDUCE AVG 1 @price AS avg_price
        REDUCE AVG 1 @sale_price AS avg_sale_price
    SORTBY 2 @avg_price DESC
    LIMIT 0 5
    DIALECT 2
```

**Stages, in order:**

| Stage | Purpose | Notes |
|-------|---------|-------|
| `LOAD n @f1 @f2 ...` | Hydrate fields from the source doc into the pipeline. | Only loaded fields are visible to later stages. `LOAD *` pulls everything (expensive). |
| `APPLY <expr> AS alias` | Project a computed field. | Operates row-by-row before grouping. |
| `FILTER <expr>` | Drop rows that fail a predicate. | Filters *pipeline rows*, not the underlying index. Index-level filters belong in the query string. |
| `GROUPBY n @f1 ... REDUCE <fn> ...` | Collapse rows that share group keys. | Reducers: `COUNT`, `COUNT_DISTINCT`, `SUM`, `AVG`, `MIN`, `MAX`, `STDDEV`, `QUANTILE`, `TOLIST`, `FIRST_VALUE`, `RANDOM_SAMPLE`. |
| `APPLY` (post-group) | Compute derived fields over reducer output. | E.g. `APPLY "@bike_count / @brand_count" AS share`. |
| `SORTBY n @f1 ASC ...` | Order the result. | The `n` is the count of (field, direction) tokens. |
| `LIMIT offset num` | Slice the result. | For result sets > 1000 rows, use `WITHCURSOR` (see cursors rule). |

**Common reducers — quick reference:**

```
REDUCE COUNT 0 AS n                          # count rows in group
REDUCE COUNT_DISTINCT 1 @user_id AS uniq     # distinct values of @user_id
REDUCE SUM 1 @price AS total
REDUCE AVG 1 @price AS mean
REDUCE MIN 1 @price AS lo
REDUCE MAX 1 @price AS hi
REDUCE QUANTILE 2 @price 0.95 AS p95
REDUCE TOLIST 1 @model AS models             # collect into a list
REDUCE FIRST_VALUE 1 @model BY @price DESC AS top_model
```

**Incorrect:** Filtering *after* grouping when you meant to filter the source rows; mismatched `n` count on `GROUPBY`/`SORTBY`; loading every field "just in case."

```
# Bad: FILTER after GROUPBY filters group rows, not source rows.
# Intent was "only new bikes," but here you keep all groups and trim brand rows by mean price.
FT.AGGREGATE idx:bicycle "*"
    GROUPBY 1 @brand REDUCE AVG 1 @price AS avg_price
    FILTER "@condition == 'new'"     # @condition no longer exists post-group!
    DIALECT 2

# Bad: GROUPBY count mismatched — RESP parse error or surprising grouping
FT.AGGREGATE idx:bicycle "*"
    GROUPBY 2 @brand               # said 2 fields but only listed 1
        REDUCE COUNT 0 AS n
    DIALECT 2

# Bad: LOAD * inflates the pipeline payload on every doc
FT.AGGREGATE idx:bicycle "*" LOAD * GROUPBY 1 @brand REDUCE COUNT 0 AS n DIALECT 2
```

**Client mirrors:**

```python
# redis-py — STEP_START aggregate_pipeline
# Mirrors doctests/query_agg.py
from redis import Redis
from redis.commands.search.aggregation import AggregateRequest
from redis.commands.search.reducers import count, avg, sort_by

r = Redis()
req = (
    AggregateRequest("@type:{mountain}")
    .load("@brand", "@price", "@condition")
    .apply(sale_price="@price * 0.9")
    .filter("@condition == 'new'")
    .group_by("@brand", count().alias("bike_count"), avg("@price").alias("avg_price"))
    .sort_by(("@avg_price", "DESC"))
    .limit(0, 5)
    .dialect(2)
)
results = r.ft("idx:bicycle").aggregate(req)
# STEP_END
```

```java
// Jedis — STEP_START aggregate_pipeline
// Mirrors QueryAggExample.java
import redis.clients.jedis.UnifiedJedis;
import redis.clients.jedis.search.aggr.AggregationBuilder;
import redis.clients.jedis.search.aggr.Reducers;
import redis.clients.jedis.search.aggr.SortedField;

try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    AggregationBuilder agg = new AggregationBuilder("@type:{mountain}")
        .load("@brand", "@price", "@condition")
        .apply("@price * 0.9", "sale_price")
        .filter("@condition == 'new'")
        .groupBy("@brand",
            Reducers.count().as("bike_count"),
            Reducers.avg("@price").as("avg_price"))
        .sortBy(SortedField.desc("@avg_price"))
        .limit(0, 5)
        .dialect(2);
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
- redis-py: [`doctests/query_agg.py`](https://github.com/redis/redis-py/blob/master/doctests/query_agg.py)
- Jedis: [`QueryAggExample.java`](https://github.com/redis/jedis/blob/master/src/test/java/io/redis/examples/QueryAggExample.java)

Reference: [FT.AGGREGATE](https://redis.io/docs/latest/commands/ft.aggregate/), [Aggregations](https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/aggregations/)

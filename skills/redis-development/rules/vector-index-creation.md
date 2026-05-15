---
title: Configure Vector Indexes Properly
impact: HIGH
impactDescription: Correct dimensions, algorithm, and distance metric are required for vector search to work at all
tags: vector, index, hnsw, flat, embeddings, search, ft.create
description: Configure Vector Indexes Properly
alwaysApply: true
---

## Configure Vector Indexes Properly

A vector field needs three things stated correctly at index time: `TYPE` (almost always `FLOAT32`), `DIM` (must equal your embedding model's output size), and `DISTANCE_METRIC` (`COSINE`, `L2`, or `IP`). Mismatching any of these silently produces wrong results or refuses inserts — there is no runtime warning.

For the algorithm choice (HNSW vs FLAT), see `vector-algorithm-choice.md`.

**Correct:** Canonical CLI form against the Bicycle dataset — 1536-dim OpenAI-style embeddings on a HASH index.

```
FT.CREATE idx:bicycle ON HASH PREFIX 1 bicycle:
    SCHEMA
        model         TEXT WEIGHT 2.0
        brand         TAG
        description   TEXT
        condition     TAG
        price         NUMERIC SORTABLE
        description_embeddings VECTOR HNSW 6
            TYPE FLOAT32
            DIM 1536
            DISTANCE_METRIC COSINE
```

**For JSON documents** the vector field is a JSONPath plus `AS alias` (see `search-json-indexing.md`):

```
FT.CREATE idx:bicycle ON JSON PREFIX 1 bicycle:
    SCHEMA
        $.description_embeddings AS description_embeddings VECTOR HNSW 6
            TYPE FLOAT32
            DIM 1536
            DISTANCE_METRIC COSINE
```

**Required attributes:**

| Attribute | Values | Notes |
|-----------|--------|-------|
| `TYPE` | `FLOAT32`, `FLOAT64`, `BFLOAT16`, `FLOAT16` | `FLOAT32` is the standard. Lower-precision types save memory on very large indexes. |
| `DIM` | integer | Must match the embedding model exactly — 1536 for OpenAI `text-embedding-3-small` / `ada-002`, 3072 for `text-embedding-3-large`, 768 for many open-source models. |
| `DISTANCE_METRIC` | `COSINE`, `L2`, `IP` | Match the metric your embedding model was trained for. Normalized embeddings work with all three but COSINE is the typical choice. |

**Incorrect:** Dim mismatch, wrong metric for normalized embeddings, or inlining the vector blob at query time (use PARAMS — see `search-vector-query.md`).

```
# Bad: DIM mismatch — inserts silently truncated/padded, queries return junk
FT.CREATE idx:bicycle ON HASH PREFIX 1 bicycle:
    SCHEMA description_embeddings VECTOR HNSW 6 TYPE FLOAT32 DIM 768 DISTANCE_METRIC COSINE
# ... but the embeddings inserted are 1536 floats

# Bad: L2 on normalized embeddings — works but obscures interpretability (use COSINE)
```

**Verifying the index after creation:**

```
FT.INFO idx:bicycle
# Look for "attributes" — confirm vector field shows correct DIM/TYPE/DISTANCE_METRIC
```

**Client mirrors:**

```python
# redis-py — STEP_START vector_index_create
# Mirrors doctests/search_vss.py
from redis import Redis
from redis.commands.search.field import TextField, TagField, NumericField, VectorField
from redis.commands.search.indexDefinition import IndexDefinition, IndexType

r = Redis()
schema = (
    TextField("model", weight=2.0),
    TagField("brand"),
    TagField("condition"),
    NumericField("price", sortable=True),
    VectorField("description_embeddings",
                algorithm="HNSW",
                attributes={"TYPE": "FLOAT32", "DIM": 1536, "DISTANCE_METRIC": "COSINE"}),
)
r.ft("idx:bicycle").create_index(
    schema,
    definition=IndexDefinition(prefix=["bicycle:"], index_type=IndexType.HASH))
# STEP_END
```

```java
// Jedis — STEP_START vector_index_create
// Mirrors VectorSearchExample.java
import redis.clients.jedis.UnifiedJedis;
import redis.clients.jedis.search.FTCreateParams;
import redis.clients.jedis.search.IndexDataType;
import redis.clients.jedis.search.schemafields.*;
import java.util.Map;

try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    jedis.ftCreate("idx:bicycle",
        FTCreateParams.createParams().on(IndexDataType.HASH).prefix("bicycle:"),
        TextField.of("model").weight(2.0),
        TagField.of("brand"),
        NumericField.of("price").sortable(),
        VectorField.builder()
            .fieldName("description_embeddings")
            .algorithm(VectorField.VectorAlgorithm.HNSW)
            .attributes(Map.of("TYPE", "FLOAT32", "DIM", 1536, "DISTANCE_METRIC", "COSINE"))
            .build());
}
// STEP_END
```

**Client mirrors — read exactly one:**
- For raw redis-py targets, read `references/clients/python-redis-py.md`.
- For Jedis (Java) targets, read `references/clients/java-jedis.md`.
- For RedisVL targets, read `references/clients/python-redisvl.md`.
- Do not read more than one client reference.

**RedisVL coverage:** higher-level schema-from-dict and `SearchIndex` usage are covered in `references/clients/python-redisvl.md` (forthcoming, spec 0004). RedisVL examples are intentionally omitted from this rule — read the RedisVL reference when targeting that SDK.

Upstream sources:
- redis-py: [`doctests/search_vss.py`](https://github.com/redis/redis-py/blob/master/doctests/search_vss.py)
- Jedis: [`VectorSearchExample.java`](https://github.com/redis/jedis/blob/master/src/test/java/io/redis/examples/VectorSearchExample.java)

Reference: [Vector Reference](https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/vectors/)

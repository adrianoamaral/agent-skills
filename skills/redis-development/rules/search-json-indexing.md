---
title: Index JSON Documents with JSONPath and Aliases
impact: MEDIUM
impactDescription: Correct JSONPath + AS alias is the difference between queryable and unreachable fields
tags: search, json, jsonpath, ft.create, vector, alias
description: Index JSON Documents with JSONPath and Aliases
alwaysApply: true
---

## Index JSON Documents with JSONPath and Aliases

For JSON documents, the schema declares `ON JSON` and each field is a JSONPath plus an `AS <alias>`. The alias is what you query against (`@alias:...`) — without `AS`, Redis Search generates one from the path that is awkward to type and easy to typo. Array elements (`$.tags[*]`) and nested objects (`$.address.city`) work seamlessly.

**Correct:** Index a JSON Bicycle catalog: TEXT, TAG, NUMERIC, an array of TAGs, and a vector.

```
# Source documents
JSON.SET bicycle:0 $ '{
  "model": "Hyperion",
  "brand": "Velorim",
  "description": "Lightweight mountain bicycle for trail riding",
  "price": 1299,
  "condition": "new",
  "categories": ["mountain", "trail", "lightweight"],
  "store_location": "-122.4,37.7",
  "description_embeddings": [/* 1536 floats */]
}'

# Index — each path declared with AS <alias>, alias is what queries reference
FT.CREATE idx:bicycle ON JSON PREFIX 1 bicycle:
    SCHEMA
        $.model              AS model             TEXT  WEIGHT 2.0
        $.brand              AS brand             TAG
        $.description        AS description       TEXT
        $.price              AS price             NUMERIC SORTABLE
        $.condition          AS condition         TAG
        $.categories[*]      AS categories        TAG
        $.store_location     AS store_location    GEO
        $.description_embeddings AS description_embeddings VECTOR HNSW 6
            TYPE FLOAT32
            DIM 1536
            DISTANCE_METRIC COSINE
```

**Query against the aliases, not the paths:**

```
FT.SEARCH idx:bicycle "@brand:{Velorim} @categories:{mountain} @price:[100 1500]"
    DIALECT 2
```

**JSONPath syntax that works inside FT.CREATE:**

| Pattern | Meaning | Example |
|---------|---------|---------|
| `$.field` | Scalar at the top level. | `$.price AS price NUMERIC` |
| `$.nested.field` | Scalar inside a nested object. | `$.address.city AS city TAG` |
| `$.array[*]` | Each element of an array as a TAG/TEXT value. | `$.tags[*] AS tags TAG` |
| `$.array[*].field` | A field from each object in an array. | `$.variants[*].sku AS skus TAG` |

**Incorrect:** Omitting `AS` (forces awkward generated aliases), trying to query the raw path, or pointing a vector field at a non-array JSON value.

```
# Bad: no AS — field is queryable as @"$.price" which is fragile and ugly.
FT.CREATE idx:bicycle ON JSON PREFIX 1 bicycle:
    SCHEMA
        $.price NUMERIC

# Bad: querying by JSON path instead of alias — wrong field identifier
FT.SEARCH idx:bicycle "@$.price:[100 500]"   # use @price:[100 500]
```

**JSON + vector pairing:**
- Embeddings must be stored as a JSON array of numbers.
- `TYPE FLOAT32` + `DIM` must match the embedding model exactly (e.g., 1536 for OpenAI `text-embedding-3-small`, 768 for many open-source models).
- `JSON.SET ... '[...]' '$.embedding'` accepts the array; the indexer encodes to FLOAT32 on read.

**Gotcha:** an array path indexed as `TAG` makes every element a discrete tag. The same path indexed as `TEXT` would *tokenize* each element. For categorical filters, prefer `TAG`.

**Client mirrors:**

```python
# redis-py — STEP_START json_indexing
# Mirrors doctests/home_json.py + dt_json.py
from redis import Redis
from redis.commands.search.field import TextField, TagField, NumericField, VectorField
from redis.commands.search.indexDefinition import IndexDefinition, IndexType

r = Redis()
schema = (
    TextField("$.model", as_name="model", weight=2.0),
    TagField("$.brand", as_name="brand"),
    TextField("$.description", as_name="description"),
    NumericField("$.price", as_name="price", sortable=True),
    TagField("$.categories[*]", as_name="categories"),
    VectorField("$.description_embeddings", as_name="description_embeddings",
                algorithm="HNSW",
                attributes={"TYPE": "FLOAT32", "DIM": 1536, "DISTANCE_METRIC": "COSINE"}),
)
r.ft("idx:bicycle").create_index(schema, definition=IndexDefinition(prefix=["bicycle:"], index_type=IndexType.JSON))
# STEP_END
```

```java
// Jedis — STEP_START json_indexing
// Mirrors JsonExample.java + HomeJsonExample.java
import redis.clients.jedis.UnifiedJedis;
import redis.clients.jedis.search.FTCreateParams;
import redis.clients.jedis.search.IndexDataType;
import redis.clients.jedis.search.schemafields.*;

try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    jedis.ftCreate("idx:bicycle",
        FTCreateParams.createParams().on(IndexDataType.JSON).prefix("bicycle:"),
        TextField.of("$.model").as("model").weight(2.0),
        TagField.of("$.brand").as("brand"),
        TextField.of("$.description").as("description"),
        NumericField.of("$.price").as("price").sortable(),
        TagField.of("$.categories[*]").as("categories"));
}
// STEP_END
```

**Client mirrors — read exactly one:**
- For raw redis-py targets, read `references/clients/python-redis-py.md`.
- For Jedis (Java) targets, read `references/clients/java-jedis.md`.
- For RedisVL targets, read `references/clients/python-redisvl.md`.
- Do not read more than one client reference.

Upstream sources:
- redis-py: [`doctests/home_json.py`](https://github.com/redis/redis-py/blob/master/doctests/home_json.py), [`dt_json.py`](https://github.com/redis/redis-py/blob/master/doctests/dt_json.py)
- Jedis: [`HomeJsonExample.java`](https://github.com/redis/jedis/blob/master/src/test/java/io/redis/examples/HomeJsonExample.java), [`JsonExample.java`](https://github.com/redis/jedis/blob/master/src/test/java/io/redis/examples/JsonExample.java)

Reference: [Index JSON documents](https://redis.io/docs/latest/develop/interact/search-and-query/indexing/json/), [JSONPath](https://redis.io/docs/latest/develop/data-types/json/path/)

---
title: Choose the Correct Field Type
impact: HIGH
impactDescription: Use TAG instead of TEXT for filtering to improve query speed 10x
tags: search, field-types, text, tag, numeric, geo, geoshape, vector
description: Choose the Correct Field Type
alwaysApply: true
---

## Choose the Correct Field Type

Each field type has different capabilities and performance characteristics.

| Field Type | Use When | Notes |
|------------|----------|-------|
| TEXT | Full-text search needed | Tokenized, stemmed |
| TAG | Exact match, filtering | Faster than TEXT for filtering |
| NUMERIC | Range queries, sorting | Use for prices, counts, timestamps |
| GEO | Point location queries | Lat/long coordinates (single points) |
| GEOSHAPE | Area/region queries | Polygons, circles, rectangles |
| VECTOR | Similarity search | HNSW or FLAT algorithm |

**Correct:** Use TAG for exact matching (Bicycle dataset).

```
FT.CREATE idx:bicycle ON HASH PREFIX 1 bicycle:
    SCHEMA
        model        TEXT WEIGHT 2.0
        description  TEXT
        brand        TAG
        condition    TAG
        price        NUMERIC SORTABLE

# Query: exact-match TAG filter on brand
FT.SEARCH idx:bicycle "@brand:{Velorim} @condition:{new}" DIALECT 2
```

**Incorrect:** Using TEXT when you don't need full-text features.

```
# Overkill: TEXT for brand/condition adds unnecessary tokenization
FT.CREATE idx:bicycle ON HASH PREFIX 1 bicycle:
    SCHEMA
        model       TEXT
        brand       TEXT
        condition   TEXT
```

**Correct:** Use GEO for points, GEOSHAPE for areas.

```
# GEO for point locations (stores, users)
FT.CREATE idx:bicycle ON HASH PREFIX 1 bicycle:
    SCHEMA
        store_location GEO

# GEOSHAPE for areas (delivery zones, boundaries)
FT.CREATE idx:zones ON JSON PREFIX 1 zone:
    SCHEMA
        $.boundary AS boundary GEOSHAPE
```

**Client mirrors:**

```python
# redis-py — STEP_START field_types
# Mirrors doctests/search_quickstart.py
from redis import Redis
from redis.commands.search.field import TextField, TagField, NumericField, GeoField
from redis.commands.search.indexDefinition import IndexDefinition, IndexType

r = Redis()
schema = (
    TextField("model", weight=2.0),
    TextField("description"),
    TagField("brand"),
    TagField("condition"),
    NumericField("price", sortable=True),
    GeoField("store_location"),
)
r.ft("idx:bicycle").create_index(
    schema,
    definition=IndexDefinition(prefix=["bicycle:"], index_type=IndexType.HASH))
# STEP_END
```

```java
// Jedis — STEP_START field_types
// Mirrors SearchQuickstartExample.java
import redis.clients.jedis.UnifiedJedis;
import redis.clients.jedis.search.FTCreateParams;
import redis.clients.jedis.search.IndexDataType;
import redis.clients.jedis.search.schemafields.*;

try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    jedis.ftCreate("idx:bicycle",
        FTCreateParams.createParams().on(IndexDataType.HASH).prefix("bicycle:"),
        TextField.of("model").weight(2.0),
        TextField.of("description"),
        TagField.of("brand"),
        TagField.of("condition"),
        NumericField.of("price").sortable(),
        GeoField.of("store_location"));
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

Reference: [Redis Search Field Types](https://redis.io/docs/latest/develop/interact/search-and-query/indexing/geoindex/)

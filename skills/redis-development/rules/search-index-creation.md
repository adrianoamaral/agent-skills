---
title: Index Only Fields You Query
impact: HIGH
impactDescription: Reduces index size and improves write performance
tags: search, ft.create, index, schema
description: Index Only Fields You Query
alwaysApply: true
---

## Index Only Fields You Query

Create indexes with only the fields you need to search, filter, or sort on. Every indexed field costs memory on every write, even if no query ever touches it.

**Correct:** Index specific fields and constrain by prefix.

```
FT.CREATE idx:bicycle ON HASH PREFIX 1 bicycle:
    SCHEMA
        model        TEXT WEIGHT 2.0
        description  TEXT
        brand        TAG
        condition    TAG
        price        NUMERIC SORTABLE
        store_location GEO
```

For JSON documents, see `search-json-indexing.md` — the same principles apply, but paths use the `$.path AS alias` form.

For FT.CREATE flag options (`SKIPINITIALSCAN`, `NOOFFSETS`, `NOFIELDS`, etc.) and their memory trade-offs, see `search-ft-create-options.md`.

**Incorrect:** Over-indexing every field "just in case," or creating an index without a prefix.

```
# Bad: every field indexed, regardless of whether queries use it
FT.CREATE idx:bicycle ON HASH PREFIX 1 bicycle:
    SCHEMA
        model TEXT description TEXT brand TEXT subcategory TEXT
        sku TEXT cost NUMERIC margin NUMERIC supplier_id TAG ...

# Bad: no prefix — every hash in the database gets indexed
FT.CREATE idx:everything ON HASH SCHEMA model TEXT
```

**Tips:**
- Start with the minimum required fields; add via `FT.ALTER` (subject to `MAXTEXTFIELDS` capacity) as new query patterns emerge.
- Use `FT.INFO` to monitor `inverted_sz_mb` and `num_records`.
- Always specify a prefix to avoid indexing unrelated keys.
- Consider field-type alternatives: TAG beats TEXT for exact-match filters; SORTABLE on NUMERIC fields you'll use in `SORTBY`.

**Client mirrors:**

```python
# redis-py — STEP_START create_index
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
// Jedis — STEP_START create_index
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

Reference: [FT.CREATE](https://redis.io/docs/latest/commands/ft.create/), [Indexing](https://redis.io/docs/latest/develop/interact/search-and-query/indexing/)

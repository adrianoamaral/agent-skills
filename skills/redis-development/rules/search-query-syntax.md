---
title: Master Redis Search Query Syntax
impact: HIGH
impactDescription: Correct operators and escaping avoid silent empty-result bugs
tags: search, query, syntax, operators, escaping, tag, text, numeric, geo
description: Master Redis Search Query Syntax
alwaysApply: true
---

## Master Redis Search Query Syntax

The Redis Search query DSL composes operators (AND, OR, NOT, optional), field-scoped predicates (`@field:value`), and delimiter-specific value forms (TAG `{}`, NUMERIC `[]`, TEXT phrase `""`). Most "empty result" bugs come from picking the wrong delimiter or forgetting to escape special characters in TAG values.

Before writing the query expression, anchor terminology in `references/search-syntax-primitives.md` (Query Term, Field Identifier, Delimiters, Operators).

**Correct:** Operator and delimiter reference, against the canonical Bicycle dataset.

```
# Field scoping — TEXT (free-text, tokenized + stemmed)
FT.SEARCH idx:bicycle "@description:wireless"                 DIALECT 2

# TAG — exact match with { }; pipe = OR
FT.SEARCH idx:bicycle "@condition:{new|refurbished}"          DIALECT 2

# NUMERIC range — inclusive [], exclusive ( prefix, +inf/-inf supported
FT.SEARCH idx:bicycle "@price:[100 500]"                      DIALECT 2
FT.SEARCH idx:bicycle "@price:[(100 (500]"                    DIALECT 2
FT.SEARCH idx:bicycle "@price:[-inf 200]"                     DIALECT 2

# TEXT phrase — quotes for exact ordering
FT.SEARCH idx:bicycle "\"mountain bicycle\""                  DIALECT 2

# TEXT prefix / suffix / infix wildcards
FT.SEARCH idx:bicycle "@model:bik*"                           DIALECT 2
FT.SEARCH idx:bicycle "@model:*ike*"                          DIALECT 2

# Fuzzy match — %term% (1 edit), %%term%% (2 edits), %%%term%%% (3 edits)
FT.SEARCH idx:bicycle "@model:%bicycle%"                      DIALECT 2

# Boolean — implicit AND (space), | OR, - NOT, ~ optional, () grouping
FT.SEARCH idx:bicycle "@type:{mountain} -@condition:{used}"   DIALECT 2
FT.SEARCH idx:bicycle "(@type:{mountain}|@type:{road}) @price:[-inf 500]" DIALECT 2

# GEO — point + radius
FT.SEARCH idx:bicycle "@store_location:[-122.4 37.7 50 km]"   DIALECT 2

# GEOSHAPE — WITHIN polygon (DIALECT 3+, but FT.CREATE marks the field)
FT.SEARCH idx:zones "@boundary:[WITHIN $poly]" PARAMS 2 poly "POLYGON((...))" DIALECT 3
```

**Correct: TAG escaping rules** — these are the single biggest source of empty-result bugs. TAG values are *not* tokenized; hyphens, dots, commas, `@`, `:`, and spaces inside a tag must be escaped with a leading backslash, and the whole value lives inside `{}`.

```
# TAG with hyphen — must escape
FT.SEARCH idx:bicycle "@brand:{Giant\\-Cycles}"               DIALECT 2

# TAG with dot — must escape
FT.SEARCH idx:bicycle "@email:{user\\@example\\.com}"         DIALECT 2

# TAG with space — escape the space (or use double-quotes inside the braces)
FT.SEARCH idx:bicycle "@brand:{Trek\\ Bicycles}"              DIALECT 2
```

**Incorrect:** Using `()` for TAG values, `{}` for TEXT, forgetting to escape hyphens, or mixing delimiters.

```
# Bad: () around a TAG value — parses as a TEXT clause, returns nothing
FT.SEARCH idx:bicycle "@condition:(new)"

# Bad: unescaped hyphen in a TAG — RQE treats the dash as NOT
FT.SEARCH idx:bicycle "@brand:{Giant-Cycles}"   # returns 0 results

# Bad: NUMERIC values inside {} — silently empty
FT.SEARCH idx:bicycle "@price:{100 500}"
```

| Delimiter | Use | Example |
|-----------|-----|---------|
| `( )` | TEXT phrase grouping / boolean grouping | `(@type:{product} \| @type:{post})` |
| `{ }` | TAG exact-match (with `\|` for alternatives) | `@condition:{new\|refurbished}` |
| `[ ]` | NUMERIC range, GEO, GEOSHAPE, VECTOR_RANGE | `@price:[100 500]`, `@price:[-inf 200]` |
| `" "` | exact phrase match in TEXT | `"mountain bicycle"` |

**Client mirrors:**

```python
# redis-py — STEP_START query_syntax
# Mirrors doctests/query_ft.py + query_em.py
from redis import Redis
r = Redis()
# TAG with escaped hyphen
r.ft("idx:bicycle").search(r"@brand:{Giant\-Cycles}")
# NUMERIC range
r.ft("idx:bicycle").search("@price:[100 500]")
# Boolean: type mountain OR road, exclude used
r.ft("idx:bicycle").search("(@type:{mountain}|@type:{road}) -@condition:{used}")
# STEP_END
```

```java
// Jedis — STEP_START query_syntax
// Mirrors QueryFtExample.java + QueryEmExample.java
import redis.clients.jedis.UnifiedJedis;
import redis.clients.jedis.search.Query;
try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    // TAG with escaped hyphen — note Java requires double-escaping the backslash
    jedis.ftSearch("idx:bicycle", new Query("@brand:{Giant\\-Cycles}"));
    jedis.ftSearch("idx:bicycle", new Query("@price:[100 500]"));
    jedis.ftSearch("idx:bicycle",
        new Query("(@type:{mountain}|@type:{road}) -@condition:{used}"));
}
// STEP_END
```

**Client mirrors — read exactly one:**
- For raw redis-py targets, read `references/clients/python-redis-py.md`.
- For Jedis (Java) targets, read `references/clients/java-jedis.md`.
- For RedisVL targets, read `references/clients/python-redisvl.md`.
- Do not read more than one client reference.

Upstream sources:
- redis-py: [`doctests/query_ft.py`](https://github.com/redis/redis-py/blob/master/doctests/query_ft.py), [`query_em.py`](https://github.com/redis/redis-py/blob/master/doctests/query_em.py), [`query_geo.py`](https://github.com/redis/redis-py/blob/master/doctests/query_geo.py), [`query_range.py`](https://github.com/redis/redis-py/blob/master/doctests/query_range.py)
- Jedis: [`QueryFtExample.java`](https://github.com/redis/jedis/blob/master/src/test/java/io/redis/examples/QueryFtExample.java), [`QueryEmExample.java`](https://github.com/redis/jedis/blob/master/src/test/java/io/redis/examples/QueryEmExample.java), [`QueryGeoExample.java`](https://github.com/redis/jedis/blob/master/src/test/java/io/redis/examples/QueryGeoExample.java)

Reference: [Query Syntax](https://redis.io/docs/latest/develop/interact/search-and-query/query/), [Escaping](https://redis.io/docs/latest/develop/interact/search-and-query/query/#tokenization)

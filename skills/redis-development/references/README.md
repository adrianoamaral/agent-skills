---
title: Redis Search references — when to read which
scope: router
appliesTo: all-clients
---

# Redis Search references — when to read which

Reference docs are loaded **on demand**, not bundled into rules. Rules link in by relative path; an agent should read only the reference(s) that match the current task. Reading every reference wastes context and risks mixing client idioms in generated code.

## Router table

| If your task is... | Read this reference |
|--------------------|---------------------|
| Writing any `FT.SEARCH` / `FT.AGGREGATE` / `FT.HYBRID` query expression | [`search-syntax-primitives.md`](./search-syntax-primitives.md) |
| Generating Python (raw `redis-py`) code | [`clients/python-redis-py.md`](./clients/python-redis-py.md) |
| Generating Java (`Jedis`) code | [`clients/java-jedis.md`](./clients/java-jedis.md) *(forthcoming — spec 0003)* |
| Generating Python code that uses the `redisvl` SDK | [`clients/python-redisvl.md`](./clients/python-redisvl.md) *(forthcoming — spec 0004)* |
| Unsure which client | Stay with the canonical CLI form inside the rule itself; no client reference is needed. |

## Mutual exclusion

The client references describe different API shapes. **Read at most one client reference per task.** Reading both `python-redis-py.md` and `java-jedis.md` is wasteful and risks producing hybrid pseudo-code that doesn't compile against either API. The same applies to `python-redis-py.md` vs `python-redisvl.md` — they are different SDKs with different surface areas.

## How rules signal which reference to load

Every `search-*` / `vector-*` rule with client mirrors emits a directive block of the form:

```markdown
**Client mirrors — read exactly one:**
- For raw redis-py targets, read `references/clients/python-redis-py.md`.
- For Jedis (Java) targets, read `references/clients/java-jedis.md`.
- For RedisVL targets, read `references/clients/python-redisvl.md`.
- Do not read more than one client reference.
```

The validator (`packages/redis-development-build/src/validate.ts`) requires this block on every search/vector rule that contains client mirrors.

## Status of references

| File | Status | Spec |
|------|--------|------|
| `search-syntax-primitives.md` | Available now | [`0001`](../../../spec/0001-search-syntax-coverage.md) |
| `clients/python-redis-py.md` | Available now | [`0002`](../../../spec/0002-redis-py-client-reference.md) |
| `clients/java-jedis.md` | Forthcoming | [`0003`](../../../spec/0003-jedis-client-reference.md) |
| `clients/python-redisvl.md` | Forthcoming | [`0004`](../../../spec/0004-redisvl-client-reference.md) |

Until a forthcoming reference lands, fall back to the CLI form inside the rule plus the short L2 client mirror at the bottom of the rule. Both are sufficient for most generation tasks.

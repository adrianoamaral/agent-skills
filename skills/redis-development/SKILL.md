---
name: redis-development
description: Redis performance optimization and best practices. Use this skill when working with Redis data structures, Redis Search, vector search with RedisVL, semantic caching with LangCache, or optimizing Redis performance.
license: MIT
metadata:
  author: redis
  version: "1.0.0"
---

# Redis Best Practices

Comprehensive performance optimization guide for Redis, including Redis Search, vector search, and semantic caching. Contains 39 rules across 11 categories, prioritized by impact to guide automated optimization and code generation.

## When to Apply

Reference these guidelines when:
- Designing Redis data models and key structures
- Implementing caching, sessions, or real-time features
- Using Redis Search (FT.CREATE, FT.SEARCH, FT.AGGREGATE)
- Building vector search or RAG applications with RedisVL
- Implementing semantic caching with LangCache
- Optimizing Redis performance and memory usage

## Rule Categories by Priority

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| 1 | Data Structures & Keys | HIGH | `data-` |
| 2 | Memory & Expiration | HIGH | `ram-` |
| 3 | Connection & Performance | HIGH | `conn-` |
| 4 | JSON Documents | MEDIUM | `json-` |
| 5 | Redis Search | HIGH | `search-` |
| 6 | Vector Search & RedisVL | HIGH | `vector-` |
| 7 | Semantic Caching | MEDIUM | `semantic-cache-` |
| 8 | Streams & Pub/Sub | MEDIUM | `stream-` |
| 9 | Clustering & Replication | MEDIUM | `cluster-` |
| 10 | Security | HIGH | `security-` |
| 11 | Observability | MEDIUM | `observe-` |

## Quick Reference

### 1. Data Structures & Keys (HIGH)

- `data-choose-structure` - Choose the Right Data Structure
- `data-key-naming` - Use Consistent Key Naming Conventions

### 2. Memory & Expiration (HIGH)

- `ram-limits` - Configure Memory Limits and Eviction Policies
- `ram-ttl` - Set TTL on Cache Keys

### 3. Connection & Performance (HIGH)

- `conn-blocking` - Avoid Slow Commands in Production
- `conn-pipelining` - Use Pipelining for Bulk Operations
- `conn-pooling` - Use Connection Pooling or Multiplexing
- `conn-timeouts` - Configure Connection Timeouts

### 4. JSON Documents (MEDIUM)

- `json-partial-updates` - Use JSON Paths for Partial Updates
- `json-vs-hash` - Choose JSON vs Hash Appropriately

### 5. Redis Search (HIGH)

- `search-command-selection` - Choose the Right FT Command for the Job (FT.SEARCH vs FT.AGGREGATE vs FT.HYBRID)
- `search-query-syntax` - Master Redis Search Query Syntax (operators, escaping, delimiters)
- `search-vector-query` - Run KNN, Range, and Pre-Filtered Vector Queries
- `search-aggregate-pipeline` - Build FT.AGGREGATE Pipelines in the Correct Stage Order
- `search-aggregate-cursors` - Paginate Large Aggregations with FT.CURSOR
- `search-json-indexing` - Index JSON Documents with JSONPath and Aliases
- `search-result-shaping` - Shape Search Results with RETURN, SORTBY, HIGHLIGHT, SUMMARIZE
- `search-debugging` - Debug Queries with FT.EXPLAIN, FT.PROFILE, FT.INFO
- `search-text-tokenization` - Control Tokenization with NOSTEM, LANGUAGE, STOPWORDS, PHONETIC
- `search-ft-create-options` - Tune FT.CREATE Options for Memory and Indexing Cost
- `search-dialect` - Use DIALECT 2 for Query Syntax
- `search-field-types` - Choose the Correct Field Type
- `search-index-creation` - Index Only Fields You Query
- `search-index-management` - Manage Indexes for Zero-Downtime Updates
- `search-query-optimization` - Write Performant Queries

### 6. Vector Search & RedisVL (HIGH)

- `vector-algorithm-choice` - Choose HNSW vs FLAT Based on Requirements
- `vector-hybrid-search` - Combine Lexical and Vector Search Correctly
- `vector-index-creation` - Configure Vector Indexes Properly
- `vector-rag-pattern` - Implement RAG Retrieval Against Redis Correctly

### 7. Semantic Caching (MEDIUM)

- `semantic-cache-best-practices` - Configure Semantic Cache Properly
- `semantic-cache-langcache-usage` - Use LangCache for LLM Response Caching

### 8. Streams & Pub/Sub (MEDIUM)

- `stream-choosing-pattern` - Choose Streams vs Pub/Sub Appropriately

### 9. Clustering & Replication (MEDIUM)

- `cluster-hash-tags` - Use Hash Tags for Multi-Key Operations
- `cluster-read-replicas` - Use Read Replicas for Read-Heavy Workloads

### 10. Security (HIGH)

- `security-acls` - Use ACLs for Fine-Grained Access Control
- `security-auth` - Always Use Authentication in Production
- `security-network` - Secure Network Access

### 11. Observability (MEDIUM)

- `observe-commands` - Use Observability Commands for Debugging
- `observe-metrics` - Monitor Key Redis Metrics

## How to Use

Read individual rule files for detailed explanations and code examples:

```
rules/search-index-creation.md
rules/vector-rag-pattern.md
```

Each rule file contains:
- Brief explanation of why it matters
- Correct example(s) with explanation
- Either an "Incorrect" example (for anti-patterns that cause real harm) or "When to use / When NOT needed" guidance (for optional features)
- Additional context and references

## References (Progressive Disclosure)

The `references/` folder holds docs loaded on demand by rule directives, not always-on like rules. Start at the router and only load the references your task needs.

| If your task is... | Read this reference |
|--------------------|---------------------|
| Writing any `FT.SEARCH` / `FT.AGGREGATE` / `FT.HYBRID` query expression | [`references/search-syntax-primitives.md`](references/search-syntax-primitives.md) |
| Generating Python (raw `redis-py`) code | `references/clients/python-redis-py.md` *(forthcoming, spec 0002)* |
| Generating Java (`Jedis`) code | `references/clients/java-jedis.md` *(forthcoming, spec 0003)* |
| Generating Python code using the `redisvl` SDK | `references/clients/python-redisvl.md` *(forthcoming, spec 0004)* |

Router file: [`references/README.md`](references/README.md). Mutual exclusion: read at most one client reference per task.

## Full Compiled Document

For the complete guide with all rules expanded: `AGENTS.md`

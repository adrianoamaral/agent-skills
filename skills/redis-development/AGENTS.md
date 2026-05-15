# Redis Development

**Version 1.0.0**  
Redis, Inc.  
January 2026

> **Note:**  
> This document is mainly for agents and LLMs to follow when maintaining,  
> generating, or refactoring Redis applications. Humans  
> may also find it useful, but guidance here is optimized for automation  
> and consistency by AI-assisted workflows.

---

## Abstract

Best practices for Redis including data structures, memory management, Redis Search, vector search with RedisVL, semantic caching with LangCache, and performance optimization. Optimized for AI agents and LLMs.

---

## Table of Contents

1. [Data Structures & Keys](#1-data-structures--keys) — **HIGH**
   - 1.1 [Choose the Right Data Structure](#11-choose-the-right-data-structure)
   - 1.2 [Use Consistent Key Naming Conventions](#12-use-consistent-key-naming-conventions)
   - 1.3 [Use Hash Field Expiration for Per-Field TTL](#13-use-hash-field-expiration-for-per-field-ttl)
   - 1.4 [Use INCR for Atomic Counters](#14-use-incr-for-atomic-counters)
   - 1.5 [Use Transactions for Atomic Multi-Command Operations](#15-use-transactions-for-atomic-multi-command-operations)
2. [Memory & Expiration](#2-memory--expiration) — **HIGH**
   - 2.1 [Configure Memory Limits and Eviction Policies](#21-configure-memory-limits-and-eviction-policies)
   - 2.2 [Set TTL on Cache Keys](#22-set-ttl-on-cache-keys)
3. [Connection & Performance](#3-connection--performance) — **HIGH**
   - 3.1 [Avoid Slow Commands in Production](#31-avoid-slow-commands-in-production)
   - 3.2 [Configure Connection Timeouts](#32-configure-connection-timeouts)
   - 3.3 [Use Client-Side Caching for Frequently Read Data](#33-use-client-side-caching-for-frequently-read-data)
   - 3.4 [Use Connection Pooling or Multiplexing](#34-use-connection-pooling-or-multiplexing)
   - 3.5 [Use Pipelining for Bulk Operations](#35-use-pipelining-for-bulk-operations)
4. [JSON Documents](#4-json-documents) — **MEDIUM**
   - 4.1 [Choose JSON vs Hash vs String Appropriately](#41-choose-json-vs-hash-vs-string-appropriately)
   - 4.2 [Use JSON Paths for Partial Updates](#42-use-json-paths-for-partial-updates)
5. [Redis Search](#5-redis-search) — **HIGH**
   - 5.1 [Build FT.AGGREGATE Pipelines in the Correct Stage Order](#51-build-ftaggregate-pipelines-in-the-correct-stage-order)
   - 5.2 [Choose the Correct Field Type](#52-choose-the-correct-field-type)
   - 5.3 [Choose the Right FT Command for the Job](#53-choose-the-right-ft-command-for-the-job)
   - 5.4 [Control Tokenization with NOSTEM, LANGUAGE, STOPWORDS, PHONETIC](#54-control-tokenization-with-nostem-language-stopwords-phonetic)
   - 5.5 [Debug Queries with FT.EXPLAIN, FT.PROFILE, FT.INFO](#55-debug-queries-with-ftexplain-ftprofile-ftinfo)
   - 5.6 [Index JSON Documents with JSONPath and Aliases](#56-index-json-documents-with-jsonpath-and-aliases)
   - 5.7 [Index Only Fields You Query](#57-index-only-fields-you-query)
   - 5.8 [Manage Indexes for Zero-Downtime Updates](#58-manage-indexes-for-zero-downtime-updates)
   - 5.9 [Master Redis Search Query Syntax](#59-master-redis-search-query-syntax)
   - 5.10 [Paginate Large Aggregations with FT.CURSOR](#510-paginate-large-aggregations-with-ftcursor)
   - 5.11 [Run KNN, Range, and Pre-Filtered Vector Queries](#511-run-knn-range-and-pre-filtered-vector-queries)
   - 5.12 [Shape Search Results with RETURN, SORTBY, HIGHLIGHT, SUMMARIZE](#512-shape-search-results-with-return-sortby-highlight-summarize)
   - 5.13 [Tune FT.CREATE Options for Memory and Indexing Cost](#513-tune-ftcreate-options-for-memory-and-indexing-cost)
   - 5.14 [Use DIALECT 2 for Query Syntax](#514-use-dialect-2-for-query-syntax)
   - 5.15 [Write Performant Queries](#515-write-performant-queries)
6. [Vector Search & RedisVL](#6-vector-search--redisvl) — **HIGH**
   - 6.1 [Choose HNSW vs FLAT Based on Requirements](#61-choose-hnsw-vs-flat-based-on-requirements)
   - 6.2 [Combine Lexical and Vector Search Correctly](#62-combine-lexical-and-vector-search-correctly)
   - 6.3 [Configure Vector Indexes Properly](#63-configure-vector-indexes-properly)
   - 6.4 [Implement RAG Retrieval Against Redis Correctly](#64-implement-rag-retrieval-against-redis-correctly)
7. [Semantic Caching](#7-semantic-caching) — **MEDIUM**
   - 7.1 [Configure Semantic Cache Properly](#71-configure-semantic-cache-properly)
   - 7.2 [Use LangCache for LLM Response Caching](#72-use-langcache-for-llm-response-caching)
8. [Streams & Pub/Sub](#8-streams--pub/sub) — **MEDIUM**
   - 8.1 [Choose Streams vs Pub/Sub Appropriately](#81-choose-streams-vs-pubsub-appropriately)
9. [Clustering & Replication](#9-clustering--replication) — **MEDIUM**
   - 9.1 [Use Hash Tags for Multi-Key Operations](#91-use-hash-tags-for-multi-key-operations)
   - 9.2 [Use Read Replicas for Read-Heavy Workloads](#92-use-read-replicas-for-read-heavy-workloads)
10. [Security](#10-security) — **HIGH**
   - 10.1 [Always Use Authentication in Production](#101-always-use-authentication-in-production)
   - 10.2 [Secure Network Access](#102-secure-network-access)
   - 10.3 [Use ACLs for Fine-Grained Access Control](#103-use-acls-for-fine-grained-access-control)
11. [Observability](#11-observability) — **MEDIUM**
   - 11.1 [Monitor Key Redis Metrics](#111-monitor-key-redis-metrics)
   - 11.2 [Use Observability Commands for Debugging](#112-use-observability-commands-for-debugging)

---

## 1. Data Structures & Keys

**Impact: HIGH**

Choosing the right Redis data type and key naming conventions. Foundation for efficient Redis usage.

### 1.1 Choose the Right Data Structure

**Impact: HIGH (Optimal memory usage and operation performance)**

Selecting the appropriate Redis data type for your use case is fundamental to performance and memory efficiency.

| Use Case | Recommended Type | Why |
|----------|------------------|-----|
| Simple values, counters | String | Fast, atomic operations |
| Object with fields | Hash | Memory efficient, partial updates, field-level expiration |
| Queue, recent items | List | O(1) push/pop at ends |
| Unique items, membership | Set | O(1) add/remove/check |
| Rankings, ranges | Sorted Set | Score-based ordering |
| Nested/hierarchical data | JSON | Path queries, nested structures, geospatial indexing with Redis Search |
| Event logs, messaging | Stream | Persistent, consumer groups |
| Similarity search | Vector Set | Native vector storage with built-in HNSW indexing |

**Incorrect: Using strings for everything.**

**Python** (redis-py):**

```python
# Storing object as JSON string loses atomic field updates
redis.set("user:1001", json.dumps({"name": "Alice", "email": "alice@example.com"}))

# To update email, must fetch, parse, modify, and rewrite entire object
user = json.loads(redis.get("user:1001"))
user["email"] = "new@example.com"
redis.set("user:1001", json.dumps(user))
```

**Java** (Jedis):**

```java
// Bad: Storing as delimited string requires manual parsing
jedis.set("bicycle", "Deimos;Ergonom;Enduro bikes;4972");
String bike = jedis.get("bicycle");
String[] fields = bike.split(";");
String model = fields[0];  // Fragile and error-prone
```

**Correct: Use Hash for objects with fields.**

**Python** (redis-py):**

```python
# Hash allows atomic field updates
redis.hset("user:1001", mapping={"name": "Alice", "email": "alice@example.com"})

# Update single field without touching others
redis.hset("user:1001", "email", "new@example.com")
```

**Java** (Jedis):**

```java
import java.util.Map;
import java.util.HashMap;

// Good: Hash models properties naturally
Map<String, String> hashFields = new HashMap<>();
hashFields.put("model", "Deimos");
hashFields.put("brand", "Ergonom");
hashFields.put("type", "Enduro bikes");
hashFields.put("price", "4972");

jedis.hset("bicycle", hashFields);

// Read individual field
String model = jedis.hget("bicycle", "model");
```

Reference: [https://redis.io/docs/latest/develop/data-types/compare-data-types/](https://redis.io/docs/latest/develop/data-types/compare-data-types/)

### 1.2 Use Consistent Key Naming Conventions

**Impact: MEDIUM (Improved maintainability and debugging)**

Well-structured key names improve code maintainability, debugging, and enable efficient key scanning.

**Correct: Use colons as separators with a consistent hierarchy.**

```python
# Pattern: service:entity:id:attribute
user:1001:profile
user:1001:settings
order:2024:items
cache:api:users:list
session:abc123
```

**Python** (redis-py):**

```python
# Good: Short, meaningful key
redis.set("product:8361", cached_html)
page = redis.get("product:8361")
```

**Java** (Jedis):**

```java
// Good: Short, meaningful key derived from URL
jedis.set("product:8361", "<some cached HTML>");
String page = jedis.get("product:8361");
```

**Incorrect: Inconsistent naming, spaces, or very long keys.**

```python
# These cause confusion and waste memory
User_1001_Profile
my key with spaces
com.mycompany.myapp.production.users.profile.data.1001
```

**Java** (Jedis):**

```java
// Bad: Using full URL as key wastes memory and slows comparisons
jedis.set("http://www.verylongurlkey.com/store/products/product.html?id=8361",
          "<some cached HTML>");
```

**Key naming tips:**

- Keep keys short but readable—they consume memory

- Consider key prefixes for multi-tenant applications

- Extract short identifiers from URLs or long strings rather than using the whole thing

- For large binary values, consider using a hash digest as the key instead of the value itself

- Use consistent separators (colons are conventional)

Reference: [https://redis.io/docs/latest/develop/use/keyspace/](https://redis.io/docs/latest/develop/use/keyspace/)

### 1.3 Use Hash Field Expiration for Per-Field TTL

**Impact: MEDIUM (Fine-grained expiration without managing timers)**

Use hash field expiration (Redis 7.4+) to delete individual fields automatically from a hash after a specific period of time. This is useful for caching scenarios where different fields have different lifetimes, and is easier than managing expiration from your own code.

**Correct: Use HEXPIRE to set per-field TTL on hash fields.**

**Python** (redis-py):**

```python
import redis

client = redis.Redis(host='localhost', port=6379)

# Set hash fields
client.hset("sensor:sensor1", mapping={
    "air_quality": "256",
    "battery_level": "89"
})

# Set 60-second TTL on specific fields (Redis 7.4+)
client.hexpire("sensor:sensor1", 60, "air_quality", "battery_level")
```

**Java** (Jedis):**

```java
import redis.clients.jedis.UnifiedJedis;
import java.util.Map;
import java.util.HashMap;

try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    Map<String, String> hashFields = new HashMap<>();
    hashFields.put("air_quality", "256");
    hashFields.put("battery_level", "89");

    jedis.hset("sensor:sensor1", hashFields);
    
    // Set 60-second TTL on specific fields (Redis 7.4+)
    jedis.hexpire("sensor:sensor1", 60, "air_quality", "battery_level");
}
```

**When to use:**

- Sensor data or metrics that become stale after a period

- Session attributes where different fields have different lifetimes

- Cached values within a hash that should auto-expire independently

- Temporary flags or tokens stored alongside persistent data

**When NOT needed:**

- Persistent user profiles or configuration

- Data where the entire hash should expire together (use `EXPIRE` on the key instead)

- Fields managed by application logic with explicit deletion

Reference: [https://redis.io/docs/latest/commands/hexpire/](https://redis.io/docs/latest/commands/hexpire/)

### 1.4 Use INCR for Atomic Counters

**Impact: MEDIUM (Atomic increment avoids race conditions)**

If a string represents an integer value, use the `INCR` command to increment the number directly. The increment is atomic and always returns the new value. Use `INCRBY` to increment by any integer (positive or negative). This is more efficient and race-condition-free than reading, incrementing in code, and writing back.

**Correct: Use INCR/INCRBY for atomic counter updates.**

**Python** (redis-py):**

```python
import redis

client = redis.Redis(host='localhost', port=6379)

# Initialize counter
client.set("counter", "0")

# Atomic increment - returns new value
new_value = client.incr("counter")  # Returns 1

# Increment by specific amount
new_value = client.incrby("counter", 10)  # Returns 11
```

**Java** (Jedis):**

```java
import redis.clients.jedis.UnifiedJedis;

try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    jedis.set("counter", "0");
    
    // Atomic increment - returns new value
    long newValue = jedis.incr("counter");  // Returns 1
    
    // Increment by specific amount
    newValue = jedis.incrBy("counter", 10);  // Returns 11
}
```

**Incorrect: Read-modify-write pattern creates race conditions.**

**Python** (redis-py):**

```python
import redis

client = redis.Redis(host='localhost', port=6379)

client.set("counter", "0")

# BAD: Race condition - another client could modify between GET and SET
curr_value = int(client.get("counter"))
client.set("counter", str(curr_value + 1))  # Not atomic!
```

**Java** (Jedis):**

```java
import redis.clients.jedis.UnifiedJedis;

try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    jedis.set("counter", "0");
    
    // BAD: Race condition between GET and SET
    long currValue = Long.parseLong(jedis.get("counter"));
    jedis.set("counter", Long.toString(currValue + 1));  // Not atomic!
}
```

Reference: [https://redis.io/docs/latest/commands/incr/](https://redis.io/docs/latest/commands/incr/)

### 1.5 Use Transactions for Atomic Multi-Command Operations

**Impact: MEDIUM (Prevents race conditions and data inconsistency)**

Use the `MULTI`/`EXEC` commands to create a transaction when you need to execute multiple commands atomically. No other client requests will be processed while the transaction is executing, preventing other clients from modifying the keys used in the transaction and avoiding inconsistent data.

**Correct: Use transactions when multiple related keys must be updated together.**

**Python** (redis-py):**

```python
import redis

client = redis.Redis(host='localhost', port=6379)

# Transaction ensures all commands execute atomically
pipe = client.pipeline(transaction=True)
pipe.set("person:1:name", "Alex")
pipe.set("person:1:rank", "Captain")
pipe.set("person:1:serial", "AB1234")
pipe.execute()  # All commands execute as one atomic unit
```

**Java** (Jedis):**

```java
import redis.clients.jedis.UnifiedJedis;
import redis.clients.jedis.Transaction;

try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    Transaction tran = (Transaction) jedis.multi();

    tran.set("person:1:name", "Alex");
    tran.set("person:1:rank", "Captain");
    tran.set("person:1:serial", "AB1234");

    tran.exec();  // All commands execute atomically
}
```

**Incorrect: Executing related commands individually when atomicity is required.**

**Python** (redis-py):**

```python
import redis

client = redis.Redis(host='localhost', port=6379)

# BAD when atomicity matters - another client could read partial state
client.set("person:1:name", "Alex")
# Another client could read here and see incomplete data
client.set("person:1:rank", "Captain")
client.set("person:1:serial", "AB1234")
```

**When to use transactions:**

- Multiple keys must be updated as a single atomic unit

- Other clients reading partial state would cause bugs

- Implementing patterns like "transfer balance between accounts"

**When transactions are NOT needed:**

- Independent operations that don't need to be atomic

- Single-command operations (already atomic)

- When using pipelining purely for performance (use `pipeline(transaction=False)`)

**Note: Transactions add overhead. Only use them when atomicity is actually required.**

Reference: [https://redis.io/docs/latest/develop/interact/transactions/](https://redis.io/docs/latest/develop/interact/transactions/)

---

## 2. Memory & Expiration

**Impact: HIGH**

Memory limits, eviction policies, TTL strategies, and memory optimization techniques.

### 2.1 Configure Memory Limits and Eviction Policies

**Impact: HIGH (Prevents out-of-memory crashes and unpredictable behavior)**

Always configure `maxmemory` and an eviction policy to prevent Redis from consuming all available memory.

**Correct: Set explicit memory limits.**

```python
maxmemory 2gb
maxmemory-policy allkeys-lru
```

| Policy | Use Case |
|--------|----------|
| `volatile-lru` | Evict keys with TTL, least recently used first |
| `allkeys-lru` | Evict any key, least recently used first |
| `volatile-ttl` | Evict keys closest to expiration |
| `noeviction` | Return errors when memory is full (use for critical data) |

**Incorrect: Running Redis without memory limits.**

```python
# No maxmemory set - Redis will use all available RAM
# Can cause OOM killer to terminate Redis or other processes
```

**Memory optimization tips:**

- Use Hashes for small objects (more memory-efficient than separate keys)

- Use `OBJECT ENCODING key` to check how Redis stores your data

- Use `MEMORY USAGE key` to check individual key memory consumption

- Enable compression in your client for large values

Reference: [https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/memory-optimization/](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/memory-optimization/)

### 2.2 Set TTL on Cache Keys

**Impact: HIGH (Prevents unbounded memory growth)**

Always set expiration times on cache keys to prevent unbounded memory growth.

**Correct: Set TTL at write time.**

**Python** (redis-py):**

```python
# Good: TTL set atomically with the value
redis.setex("cache:user:1001", 3600, user_json)

# Good: For hashes, set TTL after
redis.hset("session:abc", mapping=session_data)
redis.expire("session:abc", 1800)
```

**Java** (Jedis):**

```java
import redis.clients.jedis.params.SetParams;

// Good: TTL set atomically with SetParams
jedis.set("cachedItem:1", "fe8c357903ac9", new SetParams().ex(120));
```

**Incorrect: Forgetting TTL on cache keys.**

**Python** (redis-py):**

```python
# Risk: This key may live forever
redis.set("cache:user:1001", user_json)
```

**Java** (Jedis):**

```java
// Risk: This key may live forever
jedis.set("cachedItem:1", "fe8c357903ac9");
```

**TTL strategies:**

- Cache data: 1-24 hours depending on freshness requirements

- Sessions: 30 minutes to 24 hours

- Rate limiting: Seconds to minutes

- Temporary locks: Seconds with automatic release

Reference: [https://redis.io/commands/expire/](https://redis.io/commands/expire/)

---

## 3. Connection & Performance

**Impact: HIGH**

Connection pooling, pipelining, timeouts, and avoiding blocking commands.

### 3.1 Avoid Slow Commands in Production

**Impact: HIGH (Prevents Redis from becoming unresponsive)**

Some Redis commands are slow because they scan large datasets. Use incremental alternatives to avoid blocking the server.

| Avoid | Use Instead |
|-------|-------------|
| `KEYS *` | `SCAN` with cursor |
| `SMEMBERS` on large sets | `SSCAN` |
| `HGETALL` on large hashes | `HSCAN` |
| `LRANGE 0 -1` on large lists | Paginate with `LRANGE 0 100` |

**Correct: Use SCAN for iteration.**

**Python** (redis-py):**

```python
# Good: Non-blocking iteration
cursor = 0
while True:
    cursor, keys = redis.scan(cursor, match="user:*", count=100)
    for key in keys:
        process(key)
    if cursor == 0:
        break
```

**Java** (Jedis):**

```java
import redis.clients.jedis.ScanIteration;
import redis.clients.jedis.UnifiedJedis;
import java.util.List;

try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    // ScanIteration manages the cursor automatically
    ScanIteration scan = jedis.scanIteration(10, "user:*", "hash");

    while (!scan.isIterationCompleted()) {
        List<String> result = scan.nextBatch().getResult();
        for (String key : result) {
            process(key);
        }
    }
}
```

**Incorrect: Using KEYS in production.**

**Python** (redis-py):**

```python
# Bad: Scans all keys, slow on large datasets
keys = redis.keys("user:*")
```

**Java** (Jedis):**

```java
// Bad: Scans all keys, blocks the server
Set<String> result = jedis.keys("*");
```

**Note: Truly blocking commands (like `BLPOP`, `BRPOP`, `BLMOVE`) that wait indefinitely for data are appropriate for some use cases like job queues, but should be used with timeouts.**

```python
# Blocking pop with timeout - appropriate for queue consumers
result = redis.blpop("task_queue", timeout=5)
```

Reference: [https://redis.io/docs/latest/commands/scan/](https://redis.io/docs/latest/commands/scan/)

### 3.2 Configure Connection Timeouts

**Impact: MEDIUM (Improves connection resilience and failure recovery)**

Configure appropriate timeout values to improve your application's connection resilience. While most Redis clients set default timeouts, choosing well-tuned values based on your application's usage patterns leads to better failure recovery.

**Correct: Set timeouts based on your application needs.**

```python
r = redis.Redis(
    host='localhost',
    socket_timeout=5.0,         # Read/write timeout - tune based on expected operation time
    socket_connect_timeout=2.0,  # Connection timeout - shorter for fast failure detection
    retry_on_timeout=True        # Automatic retry on timeout
)
```

**Incorrect: Relying solely on defaults without considering your use case.**

```python
# Not ideal: Default timeouts may not match your application's needs
r = redis.Redis(host='localhost')

# For example, if your app needs fast failure detection,
# the default timeouts might be too generous
```

**Considerations:**

- Set `socket_connect_timeout` shorter than `socket_timeout` for quick connection failure detection

- For latency-sensitive apps, use tighter timeouts with retry logic

- For batch operations, allow longer timeouts to complete large operations

- Consider using health checks alongside timeouts for robust failure handling

Reference: [https://redis.io/docs/latest/develop/clients/](https://redis.io/docs/latest/develop/clients/)

### 3.3 Use Client-Side Caching for Frequently Read Data

**Impact: HIGH (Reduces network round-trips for repeated reads)**

Use a connection with client-side caching enabled for any data that will be read frequently but written only occasionally. Client-side caching avoids contacting the server for repeated access to data that has recently been read, reducing network traffic and improving performance.

**Correct: Enable client-side caching with RESP3 protocol for frequently accessed data.**

**Python** (redis-py):**

```python
import redis

# Enable client-side caching with RESP3
client = redis.Redis(
    host='localhost',
    port=6379,
    protocol=3,  # RESP3 required for client-side caching
    cache_config=redis.CacheConfig(max_size=1000)
)

# Cached reads avoid server round-trips
value = client.get("frequently:read:key")
```

**Java** (Jedis):**

```java
import redis.clients.jedis.DefaultJedisClientConfig;
import redis.clients.jedis.UnifiedJedis;
import redis.clients.jedis.HostAndPort;
import redis.clients.jedis.CacheConfig;

HostAndPort endpoint = new HostAndPort("localhost", 6379);

DefaultJedisClientConfig config = DefaultJedisClientConfig
    .builder()
    .password("secretPassword")
    .protocol(RedisProtocol.RESP3)
    .build();

CacheConfig cacheConfig = CacheConfig.builder().maxSize(1000).build();

UnifiedJedis client = new UnifiedJedis(endpoint, config, cacheConfig);
```

**When to use:**

- Configuration data read frequently, updated rarely

- User session data accessed on every request

- Feature flags or settings checked repeatedly

- Any read-heavy workload with low write frequency

**When NOT needed:**

- Data that changes frequently (cache invalidation overhead outweighs benefits)

- Write-heavy workloads

- Simple applications where network latency is not a bottleneck

- When you need guaranteed real-time consistency

**Trade-offs:**

- Adds memory overhead on the client

- Requires RESP3 protocol

- Cache invalidation adds complexity for frequently changing data

Reference: [https://redis.io/docs/latest/develop/clients/client-side-caching/](https://redis.io/docs/latest/develop/clients/client-side-caching/)

### 3.4 Use Connection Pooling or Multiplexing

**Impact: HIGH (Reduces connection overhead by 10x or more)**

Reuse connections via a pool or multiplexing instead of creating new connections per request.

**Correct: Use a connection pool.**

**Python** (redis-py):**

```python
import redis

# Good: Connection pool - reuses existing connections
pool = redis.ConnectionPool(host='localhost', port=6379, max_connections=50)
r = redis.Redis(connection_pool=pool)
```

**Java** (Jedis):**

```java
import redis.clients.jedis.JedisPooled;

// JedisPooled manages a connection pool internally
try (JedisPooled jedis = new JedisPooled("redis://localhost:6379")) {
    jedis.set("testKey", "testValue");
}
```

**Correct: Use multiplexing (Lettuce, NRedisStack).**

```java
// Lettuce uses multiplexing by default - single connection handles all traffic
RedisClient client = RedisClient.create("redis://localhost:6379");
StatefulRedisConnection<String, String> connection = client.connect();

// All commands share the single connection efficiently
connection.sync().set("key", "value");
```

**Incorrect: Creating new connections per request.**

**Python** (redis-py):**

```python
# Bad: New connection every time
def get_user(user_id):
    r = redis.Redis(host='localhost', port=6379)  # Don't do this
    return r.get(f"user:{user_id}")
```

**Java** (Jedis):**

```java
// Bad: Creating new client per request
public String getUser(String userId) {
    try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
        return jedis.get("user:" + userId);  // Don't do this
    }
}
```

**Pooling vs Multiplexing:**

- **Pooling**: Multiple connections shared across requests (redis-py, Jedis, go-redis)

- **Multiplexing**: Single connection handles all traffic (NRedisStack, Lettuce)

- Multiplexing cannot support blocking commands (BLPOP, etc.) as they would stall all callers

Reference: [https://redis.io/docs/latest/develop/clients/pools-and-muxing/](https://redis.io/docs/latest/develop/clients/pools-and-muxing/)

### 3.5 Use Pipelining for Bulk Operations

**Impact: HIGH (Reduces round trips, 5-10x faster for batch operations)**

Batch multiple commands into a single round trip to reduce network latency.

**Correct: Use pipeline for multiple commands.**

**Python** (redis-py):**

```python
# Good: Single round trip for multiple commands
pipe = redis.pipeline()
for user_id in user_ids:
    pipe.get(f"user:{user_id}")
results = pipe.execute()
```

**Java** (Jedis):**

```java
import redis.clients.jedis.Pipeline;

// Good: Buffer commands and send as single batch
Pipeline pipe = (Pipeline) jedis.pipelined();

pipe.set("person:1:name", "Alex");
pipe.set("person:1:rank", "Captain");
pipe.set("person:1:serial", "AB1234");

pipe.sync();
```

**Incorrect: Sequential commands in a loop.**

**Python** (redis-py):**

```python
# Bad: N round trips
results = []
for user_id in user_ids:
    results.append(redis.get(f"user:{user_id}"))
```

**Java** (Jedis):**

```java
// Bad: 3 separate round trips
jedis.set("person:1:name", "Alex");
jedis.set("person:1:rank", "Captain");
jedis.set("person:1:serial", "AB1234");
```

Reference: [https://redis.io/docs/latest/develop/use/pipelining/](https://redis.io/docs/latest/develop/use/pipelining/)

---

## 4. JSON Documents

**Impact: MEDIUM**

Using Redis JSON for nested structures, partial updates, and integration with Redis Search.

### 4.1 Choose JSON vs Hash vs String Appropriately

**Impact: MEDIUM (Optimal data model for your use case)**

Redis offers three ways to store structured data: JSON, Hash, and serialized strings. Each has distinct trade-offs around atomic partial operations and indexability.

| Feature | JSON | Hash | String (serialized JSON) |
|---------|------|------|--------------------------|
| **Structure** | Nested objects and arrays | Flat key-value pairs | Any structure |
| **Atomic partial reads** | Yes (`$.field`) | Yes (`HGET`) | No (must fetch entire value) |
| **Atomic partial writes** | Yes (`JSON.SET $.field`) | Yes (`HSET`) | No (must rewrite entire value) |
| **Search indexing** | Yes | Yes | No |
| **Geospatial indexing** | Yes | Yes | No |
| **Memory efficiency** | Higher overhead | More efficient | Most compact |
| **Field-level expiration** | No | Yes (HEXPIRE) | No |

**When to use each:**

- **JSON**: Nested structures with atomic partial updates and indexing needs

- **Hash**: Flat objects with atomic field access, field-level expiration, or memory efficiency

- **String**: Simple caching where you always read/write the entire object and don't need indexing

**Correct: Use JSON for nested structures with atomic partial updates.**

**Python** (redis-py):**

```python
# JSON supports nested structures and atomic deep updates
redis.json().set("user:1001", "$", {
    "name": "Alice",
    "preferences": {"theme": "dark", "notifications": True}
})

# Atomic update of nested field - no read-modify-write needed
redis.json().set("user:1001", "$.preferences.theme", "light")
```

**Java** (Jedis):**

```java
import redis.clients.jedis.UnifiedJedis;
import redis.clients.jedis.json.Path2;
import org.json.JSONObject;

try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    JSONObject user = new JSONObject();
    user.put("name", "Alice");
    user.put("preferences", new JSONObject().put("theme", "dark"));

    jedis.jsonSet("user:1001", new Path2("$"), user);

    // Atomic update of nested field
    jedis.jsonSet("user:1001", new Path2("$.preferences.theme"), "light");
}
```

**Correct: Use Hash for flat objects with atomic field access.**

**Python** (redis-py):**

```python
# Hash is efficient for flat data with atomic field operations
redis.hset("session:abc", mapping={
    "user_id": "1001",
    "created_at": "2024-01-01",
    "ip": "192.168.1.1"
})

# Atomic field read and update
ip = redis.hget("session:abc", "ip")
redis.hset("session:abc", "ip", "10.0.0.1")
```

**Correct: Use String for simple caching without partial updates.**

**Python** (redis-py):**

```python
import json

# String is fine when you always read/write the entire object
# and don't need indexing or partial updates
config = {"feature_flags": {"dark_mode": True}, "version": "1.0"}
redis.set("config:app", json.dumps(config), ex=3600)

# Must fetch and parse entire object
config = json.loads(redis.get("config:app"))
```

**Incorrect: Using String when you need atomic partial updates.**

**Python** (redis-py):**

```python
import json

# BAD: Must fetch, parse, modify, serialize, and rewrite entire object
data = json.loads(redis.get("user:1001"))
data["preferences"]["theme"] = "light"  # Not atomic!
redis.set("user:1001", json.dumps(data))
# Another client could have modified the object between GET and SET
```

Reference: [https://redis.io/docs/latest/develop/data-types/compare-data-types/#documents](https://redis.io/docs/latest/develop/data-types/compare-data-types/#documents)

### 4.2 Use JSON Paths for Partial Updates

**Impact: MEDIUM (Avoids fetching and rewriting entire documents)**

Use JSON path syntax to update specific fields without fetching the entire document.

**Correct: Use JSON paths for targeted updates.**

```python
# Store JSON document
redis.json().set("user:1001", "$", {
    "name": "Alice",
    "email": "alice@example.com",
    "preferences": {"theme": "dark", "notifications": True}
})

# Update nested field without fetching entire document
redis.json().set("user:1001", "$.preferences.theme", "light")

# Get specific field
theme = redis.json().get("user:1001", "$.preferences.theme")

# Increment numeric field atomically
redis.json().numincrby("user:1001", "$.preferences.volume", 5)

# Append to array
redis.json().arrappend("user:1001", "$.tags", "premium")
```

**Incorrect: Storing JSON as a string and parsing client-side.**

```python
# Bad: Loses queryability and atomic updates
redis.set("user:1001", json.dumps(user_data))

# Must fetch, parse, modify, serialize, and rewrite
data = json.loads(redis.get("user:1001"))
data["preferences"]["theme"] = "light"
redis.set("user:1001", json.dumps(data))
```

Reference: [https://redis.io/docs/latest/develop/data-types/json/path/](https://redis.io/docs/latest/develop/data-types/json/path/)

---

## 5. Redis Search

**Impact: HIGH**

FT.CREATE, FT.SEARCH, FT.AGGREGATE, index design, field types, and query optimization.

### 5.1 Build FT.AGGREGATE Pipelines in the Correct Stage Order

**Impact: HIGH (Pipeline stage order determines correctness — wrong order silently returns wrong results)**

`FT.AGGREGATE` runs stages in the order you write them, like a Unix pipeline. The canonical order is `LOAD → APPLY → FILTER → GROUPBY/REDUCE → APPLY → SORTBY → LIMIT`. Swapping stages doesn't error — it silently changes what your query computes. For paginating large aggregates, see `search-aggregate-cursors.md`.

**Correct: Canonical pipeline against the Bicycle dataset — load needed fields, project a derived field, filter, group, sort, limit.**

```python
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

```python
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

**Incorrect: Filtering *after* grouping when you meant to filter the source rows; mismatched `n` count on `GROUPBY`/`SORTBY`; loading every field "just in case."**

```python
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

Reference: [https://redis.io/docs/latest/commands/ft.aggregate/](https://redis.io/docs/latest/commands/ft.aggregate/), [https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/aggregations/](https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/aggregations/)

### 5.2 Choose the Correct Field Type

**Impact: HIGH (Use TAG instead of TEXT for filtering to improve query speed 10x)**

Each field type has different capabilities and performance characteristics.

| Field Type | Use When | Notes |
|------------|----------|-------|
| TEXT | Full-text search needed | Tokenized, stemmed |
| TAG | Exact match, filtering | Faster than TEXT for filtering |
| NUMERIC | Range queries, sorting | Use for prices, counts, timestamps |
| GEO | Point location queries | Lat/long coordinates (single points) |
| GEOSHAPE | Area/region queries | Polygons, circles, rectangles |
| VECTOR | Similarity search | HNSW or FLAT algorithm |

**Correct: Use TAG for exact matching (Bicycle dataset).**

```python
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

**Incorrect: Using TEXT when you don't need full-text features.**

```python
# Overkill: TEXT for brand/condition adds unnecessary tokenization
FT.CREATE idx:bicycle ON HASH PREFIX 1 bicycle:
    SCHEMA
        model       TEXT
        brand       TEXT
        condition   TEXT
```

**Correct: Use GEO for points, GEOSHAPE for areas.**

```python
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

Reference: [https://redis.io/docs/latest/develop/interact/search-and-query/indexing/geoindex/](https://redis.io/docs/latest/develop/interact/search-and-query/indexing/geoindex/)

### 5.3 Choose the Right FT Command for the Job

**Impact: HIGH (Picking FT.SEARCH vs FT.AGGREGATE vs FT.HYBRID up front avoids a full rewrite later)**

The first decision before any query syntax is *which command to run*. Redis Search exposes three query commands with different design intents — picking the wrong one means rewriting the query later when you discover the command cannot express what you need.

| Command | Use when... | Mental model | Min. Redis |
|---------|-------------|--------------|------------|
| `FT.SEARCH` | Straightforward document retrieval — agent wants matching docs back. | Ready-to-use: returns matching documents directly. | 2.0 module / 8.0 built-in |
| `FT.AGGREGATE` | Faceting, analytics, computed fields, grouped or reshaped output. | Declarative result shaping: explicit `LOAD`, `APPLY`, `GROUPBY`, `REDUCE`, `SORTBY`. | 2.0 module / 8.0 built-in |
| `FT.HYBRID` | Relevance must blend lexical (text) and semantic (vector) ranking with explicit fusion. | Declarative hybrid retrieval: `SEARCH` leg + `VSIM` leg + `COMBINE` fusion (RRF or LINEAR). | **8.4.0** (Redis Open Source) |

**Correct: Pick the command that matches the shape of the answer you need.**

```python
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

**Version gate — FT.HYBRID requires Redis ≥ 8.4.0.** For older Redis, fall back to the pre-filter + KNN pattern via `FT.SEARCH` (see `search-vector-query.md`):**

```python
# Fallback for Redis < 8.4.0 — pre-filter + KNN inside FT.SEARCH
FT.SEARCH idx:bicycle "(@type:{mountain})=>[KNN 10 @description_embeddings $query_vec AS score]"
    SORTBY score
    PARAMS 2 query_vec "<vector_blob>"
    DIALECT 2
```

**When to use FT.HYBRID's COMBINE modes:**

- `COMBINE RRF` — Reciprocal Rank Fusion, rank-based fusion. Robust default; no tuning required.

- `COMBINE LINEAR ALPHA <a> BETA <b>` — weighted score blend. Use when you have calibrated scores and want explicit control over the lexical/vector trade-off.

**Incorrect: Using `FT.SEARCH` and then post-processing in the client to compute groups, averages, or score fusion. That work belongs inside Redis — pushing it client-side defeats the index.**

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

Reference: [https://redis.io/docs/latest/commands/ft.hybrid/](https://redis.io/docs/latest/commands/ft.hybrid/), [https://redis.io/docs/latest/commands/ft.search/](https://redis.io/docs/latest/commands/ft.search/), [https://redis.io/docs/latest/commands/ft.aggregate/](https://redis.io/docs/latest/commands/ft.aggregate/)

### 5.4 Control Tokenization with NOSTEM, LANGUAGE, STOPWORDS, PHONETIC

**Impact: MEDIUM (Tokenization choices determine recall — wrong stemmer or stopword set silently drops correct matches)**

TEXT fields are tokenized, stemmed, and stopword-filtered at index time. Defaults work for English prose, but they silently drop matches when you index product SKUs, code identifiers, or non-English text. Tokenization is the most common reason `FT.EXPLAIN` shows a token expansion you didn't expect.

**Correct: Pick tokenization options per field, based on the kind of text in it.**

```python
# A schema mixing prose, identifiers, and a non-English field
FT.CREATE idx:bicycle ON HASH PREFIX 1 bicycle:
    SCHEMA
        # Prose — stem so "running" matches "run"
        description    TEXT WEIGHT 1.0
        # Model codes — don't stem, don't tokenize aggressively
        model          TEXT NOSTEM
        # Brand name — boost it in scoring
        brand          TEXT WEIGHT 3.0
        # Phonetic match for misspellings ("smyth" → "Smith")
        owner_name     TEXT PHONETIC dm:en

# Index-wide options
FT.CREATE idx:bicycle_de ON HASH PREFIX 1 bicycle:
    LANGUAGE german                              # default stemmer for all TEXT fields
    STOPWORDS 3 der die und                      # custom stopword list (0 disables)
    SCHEMA
        description TEXT
```

**Option-by-option:**

| Option | Scope | Effect |
|--------|-------|--------|
| `NOSTEM` | per TEXT field | Skip stemming. Use for SKUs, model codes, identifiers — anything where `running` ≠ `run`. |
| `WEIGHT n` | per TEXT field | Multiplier on TF/IDF contribution. Default 1.0; raise for high-signal fields like `title` or `brand`. |
| `LANGUAGE <lang>` | index-wide (or per-doc) | Selects the stemmer. Defaults to `english`. Supported: english, arabic, chinese, danish, dutch, finnish, french, german, hungarian, italian, norwegian, portuguese, romanian, russian, spanish, swedish, tamil, turkish. |
| `STOPWORDS n w1 w2 ...` | index-wide | Override the default English stopword list. `STOPWORDS 0` disables stopword removal entirely (necessary when stopwords are meaningful in your domain, e.g., `"to be"`). |
| `PHONETIC <matcher>` | per TEXT field | Index phonetic codes for fuzzy-name matching. Matchers: `dm:en` (English), `dm:fr`, `dm:pt`, `dm:es`. |

**Diagnose tokenization with `FT.EXPLAIN`:**

```python
FT.EXPLAIN idx:bicycle "running shoes"
# → INTERSECT { UNION{run, running} UNION{shoe, shoes} }
# Stemming is expanding the terms. If "running" should be literal, mark the field NOSTEM.
```

**Incorrect: Using TEXT for identifiers (loses recall on SKUs), forgetting to disable stopwords for short queries that include them, or setting LANGUAGE on the wrong layer.**

```python
# Bad: SKU as TEXT without NOSTEM — "BIKE-2024" gets tokenized + stemmed
FT.CREATE idx:bicycle ON HASH PREFIX 1 bicycle:
    SCHEMA
        sku TEXT                          # use NOSTEM, or use TAG

# Bad: querying "to be" against an index with default stopwords
FT.SEARCH idx:books "to be or not to be"
# → effectively searches "" — every stopword is dropped.

# Bad: putting LANGUAGE on a single field — it is an index-wide option
FT.CREATE idx:bicycle ON HASH
    SCHEMA description TEXT LANGUAGE french      # this is rejected
```

**Choosing TEXT vs TAG:**

- TEXT: prose, descriptions, anything users type into a search box.

- TAG: identifiers, categories, statuses, anything where exact match is what you want and tokenization is harmful.

- A SKU like `BIKE-2024-XL` is almost always better as TAG.

**Client mirrors:**

```java
// Jedis — STEP_START tokenization
import redis.clients.jedis.UnifiedJedis;
import redis.clients.jedis.search.FTCreateParams;
import redis.clients.jedis.search.schemafields.*;

try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    jedis.ftCreate("idx:bicycle",
        FTCreateParams.createParams(),
        TextField.of("description"),
        TextField.of("model").noStem(),
        TextField.of("brand").weight(3.0),
        TextField.of("owner_name").phonetic("dm:en"),
        TagField.of("sku"));
}
// STEP_END
```

**Client mirrors — read exactly one:**

- For raw redis-py targets, read `references/clients/python-redis-py.md`.

- For Jedis (Java) targets, read `references/clients/java-jedis.md`.

- For RedisVL targets, read `references/clients/python-redisvl.md`.

- Do not read more than one client reference.

Upstream sources: No direct upstream example — authored from official Redis Search command documentation (https://redis.io/commands/ft.create/) and tokenization docs.

Reference: [https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/stemming/](https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/stemming/), [https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/stopwords/](https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/stopwords/), [https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/phonetic_matching/](https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/phonetic_matching/)

### 5.5 Debug Queries with FT.EXPLAIN, FT.PROFILE, FT.INFO

**Impact: MEDIUM (Targeted diagnostics turn "empty results" or "slow query" guesses into 10-second answers)**

Three commands cover ~95% of search debugging: `FT.EXPLAIN` shows how the parser interpreted the query expression, `FT.PROFILE` measures stage-by-stage execution, and `FT.INFO` reports on the index itself (size, doc count, indexing failures, configuration). Reach for them *before* tweaking schema or rewriting queries.

**Correct: Run the right diagnostic for the symptom.**

```python
# Symptom: "my query returns nothing" — see how the parser actually read it
FT.EXPLAIN idx:bicycle "@brand:{Giant-Cycles}"
#  → INTERSECT { @brand:TAG{Giant} NOT TAG{Cycles} }   ← the hyphen was treated as NOT!

# Stemming surprise — see token expansion
FT.EXPLAIN idx:bicycle "running shoes"
#  → INTERSECT { UNION{run, running} UNION{shoe, shoes} }

# Symptom: "slow query" — full stage timing
FT.PROFILE idx:bicycle SEARCH QUERY "@type:{mountain} @price:[100 500]" LIMIT 0 20

# Same for aggregate
FT.PROFILE idx:bicycle AGGREGATE QUERY "@type:{mountain}"
    GROUPBY 1 @brand REDUCE COUNT 0 AS n

# Symptom: "I changed the schema and queries look weird" — inspect the index
FT.INFO idx:bicycle
```

**What to look for in `FT.INFO`:**

| Field | Means | What to do if it's off |
|-------|-------|------------------------|
| `num_docs` | Indexed doc count. | If lower than expected, check `hash_indexing_failures`. |
| `num_records` | Total indexed terms (across all fields). | High vs `num_docs` may indicate over-indexing TEXT. |
| `hash_indexing_failures` | Documents that failed indexing. | Inspect a failing doc with `JSON.GET` or `HGETALL`; usually a type mismatch on a NUMERIC field, or non-FLOAT32 vector blob. |
| `inverted_sz_mb` | Memory used by the inverted index. | If large, consider `NOOFFSETS`, `NOFREQS`, `NOHL` (see `search-ft-create-options.md`). |
| `indexing` | `1` if a background indexing job is running. | Wait for `0` before benchmarking. |
| `percent_indexed` | Progress of initial scan. | `1.0` = fully indexed. |
| `gc_stats` | Garbage-collector activity. | Frequent runs usually mean lots of deletes/updates. |
| `attributes` | Per-field schema. | Verify a field is actually present at the alias you're querying. |

**Reading `FT.PROFILE` output:**

- Top-level `Total profile time` is the wall-clock cost.

- The `Iterators profile` tree shows which query clause did how much work; a giant `Counter` on a TEXT term means it matched a huge fraction of docs.

- `Parsing time` + `Pipeline creation time` + `Iterators profile` should account for ~all the time. If `Iterators profile` is small but `Total` is large, the bottleneck is post-processing (SORT, RETURN, LIMIT).

**Incorrect: Editing schema or guessing at perf fixes before running diagnostics.**

```python
# Bad: "let me just add SORTABLE to every field and see what happens"
# Worse: "let me re-create the index" before checking hash_indexing_failures
```

**Common errors and what they mean:**

| Error | Likely cause |
|-------|--------------|
| `Unknown index name` | Typo, or index dropped. List with `FT._LIST`. |
| `Syntax error at offset N` | Unbalanced `()` / `{}` / `[]`, or unescaped `-`/`.` inside a TAG. |
| `Vector index initialization failed` | DIM mismatch, wrong TYPE, or non-array path. |
| `Document already in index` | Duplicate key on `FT.ADD` (legacy); not produced by modern HSET/JSON.SET flow. |
| `Document is already in index` after `JSON.SET` | Same key indexed by two indexes with overlapping prefixes — narrow the prefixes. |

**Client mirrors:**

```java
// Jedis — STEP_START debugging
import redis.clients.jedis.UnifiedJedis;
import redis.clients.jedis.search.Query;

try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    String explain = jedis.ftExplain("idx:bicycle", new Query("@brand:{Giant-Cycles}"));
    System.out.println(explain);
    System.out.println(jedis.ftProfileSearch("idx:bicycle", null,
        new Query("@type:{mountain}").limit(0, 20)));
    System.out.println(jedis.ftInfo("idx:bicycle"));
}
// STEP_END
```

**Client mirrors — read exactly one:**

- For raw redis-py targets, read `references/clients/python-redis-py.md`.

- For Jedis (Java) targets, read `references/clients/java-jedis.md`.

- For RedisVL targets, read `references/clients/python-redisvl.md`.

- Do not read more than one client reference.

Upstream sources: No direct upstream example — authored from official Redis Search command documentation (https://redis.io/commands/ft.explain/, https://redis.io/commands/ft.profile/, https://redis.io/commands/ft.info/).

Reference: [https://redis.io/docs/latest/commands/ft.explain/](https://redis.io/docs/latest/commands/ft.explain/), [https://redis.io/docs/latest/commands/ft.profile/](https://redis.io/docs/latest/commands/ft.profile/), [https://redis.io/docs/latest/commands/ft.info/](https://redis.io/docs/latest/commands/ft.info/)

### 5.6 Index JSON Documents with JSONPath and Aliases

**Impact: MEDIUM (Correct JSONPath + AS alias is the difference between queryable and unreachable fields)**

For JSON documents, the schema declares `ON JSON` and each field is a JSONPath plus an `AS <alias>`. The alias is what you query against (`@alias:...`) — without `AS`, Redis Search generates one from the path that is awkward to type and easy to typo. Array elements (`$.tags[*]`) and nested objects (`$.address.city`) work seamlessly.

**Correct: Index a JSON Bicycle catalog: TEXT, TAG, NUMERIC, an array of TAGs, and a vector.**

```python
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

```python
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

**Incorrect: Omitting `AS` (forces awkward generated aliases), trying to query the raw path, or pointing a vector field at a non-array JSON value.**

```python
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

**Gotcha: an array path indexed as `TAG` makes every element a discrete tag. The same path indexed as `TEXT` would *tokenize* each element. For categorical filters, prefer `TAG`.**

**Client mirrors:**

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

Reference: [https://redis.io/docs/latest/develop/interact/search-and-query/indexing/json/](https://redis.io/docs/latest/develop/interact/search-and-query/indexing/json/), [https://redis.io/docs/latest/develop/data-types/json/path/](https://redis.io/docs/latest/develop/data-types/json/path/)

### 5.7 Index Only Fields You Query

**Impact: HIGH (Reduces index size and improves write performance)**

Create indexes with only the fields you need to search, filter, or sort on. Every indexed field costs memory on every write, even if no query ever touches it.

**Correct: Index specific fields and constrain by prefix.**

```python
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

**Incorrect: Over-indexing every field "just in case," or creating an index without a prefix.**

```python
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

Reference: [https://redis.io/docs/latest/commands/ft.create/](https://redis.io/docs/latest/commands/ft.create/), [https://redis.io/docs/latest/develop/interact/search-and-query/indexing/](https://redis.io/docs/latest/develop/interact/search-and-query/indexing/)

### 5.8 Manage Indexes for Zero-Downtime Updates

**Impact: MEDIUM (Aliases enable seamless index swaps; knowing FT.ALTER limits avoids re-indexing surprises)**

Use index *aliases* so applications query a stable name while you swap the underlying index on schema changes. `FT.ALTER` can append fields to an existing index but cannot change a field's type, options, or remove it — anything beyond *adding* a field requires building a new index and swapping the alias.

**Correct: Build the new index in parallel, then atomically swap the alias.**

```python
# 1. Build the new version of the index from scratch
FT.CREATE idx:bicycle_v2 ON HASH PREFIX 1 bicycle:
    SCHEMA
        model TEXT WEIGHT 2.0
        brand TAG
        price NUMERIC SORTABLE

# Wait until percent_indexed = 1.0
FT.INFO idx:bicycle_v2

# 2. Point the application alias at the new index in one atomic step
FT.ALIASUPDATE bicycle idx:bicycle_v2

# 3. Drop the old version
FT.DROPINDEX idx:bicycle_v1
```

**Adding a field is in-place — use `FT.ALTER`:**

```python
# Add a TEXT field with WEIGHT to an existing index — no rebuild needed
FT.ALTER idx:bicycle SCHEMA ADD subtitle TEXT WEIGHT 1.5
```

**`FT.ALTER` limitations — when you must rebuild:**

| Change | Can FT.ALTER do it? |
|--------|---------------------|
| Add a new field | Yes — `FT.ALTER ... SCHEMA ADD ...` |
| Remove a field | **No** — must rebuild. |
| Change a field's type (TEXT → TAG, etc.) | **No** — must rebuild. |
| Change SORTABLE, NOSTEM, WEIGHT, PHONETIC | **No** — must rebuild. |
| Change the index `PREFIX` | **No** — must rebuild. |
| Change `LANGUAGE`, `STOPWORDS`, `NOFIELDS`, `NOOFFSETS` | **No** — must rebuild. |
| Grow beyond `MAXTEXTFIELDS` capacity | **No** — must rebuild (set `MAXTEXTFIELDS` upfront on indexes you expect to grow). |

**Useful management commands:**

```python
# List every search index
FT._LIST

# Inspect schema, doc count, indexing progress, memory
FT.INFO idx:bicycle

# Create an alias up front (so application code always uses the alias)
FT.ALIASADD bicycle idx:bicycle_v1

# Atomic swap when a v2 is ready
FT.ALIASUPDATE bicycle idx:bicycle_v2

# Drop an index (non-blocking)
FT.DROPINDEX idx:bicycle_v1

# Drop the index AND delete every indexed document
FT.DROPINDEX idx:bicycle_v1 DD
```

**Incorrect: Dropping the live index before the new one is ready, or relying on a hard-coded index name in application code.**

```python
# Bad: drop-and-recreate while traffic is hitting the index
FT.DROPINDEX idx:bicycle
FT.CREATE idx:bicycle ...            # queries during the rebuild return errors

# Bad: application queries idx:bicycle_v1 directly — no painless way to roll forward
```

**Client mirrors:**

```java
// Jedis — STEP_START index_management
import redis.clients.jedis.UnifiedJedis;
import redis.clients.jedis.search.schemafields.TextField;

try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    // Atomic alias swap
    jedis.ftAliasUpdate("bicycle", "idx:bicycle_v2");
    jedis.ftDropIndex("idx:bicycle_v1");
    // Add a field in place
    jedis.ftAlter("idx:bicycle", TextField.of("subtitle").weight(1.5));
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

Reference: [https://redis.io/docs/latest/commands/ft.aliasadd/](https://redis.io/docs/latest/commands/ft.aliasadd/), [https://redis.io/docs/latest/commands/ft.alter/](https://redis.io/docs/latest/commands/ft.alter/), [https://redis.io/docs/latest/commands/ft.dropindex/](https://redis.io/docs/latest/commands/ft.dropindex/)

### 5.9 Master Redis Search Query Syntax

**Impact: HIGH (Correct operators and escaping avoid silent empty-result bugs)**

The Redis Search query DSL composes operators (AND, OR, NOT, optional), field-scoped predicates (`@field:value`), and delimiter-specific value forms (TAG `{}`, NUMERIC `[]`, TEXT phrase `""`). Most "empty result" bugs come from picking the wrong delimiter or forgetting to escape special characters in TAG values.

Before writing the query expression, anchor terminology in `references/search-syntax-primitives.md` (Query Term, Field Identifier, Delimiters, Operators).

**Correct: Operator and delimiter reference, against the canonical Bicycle dataset.**

```python
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

**Correct: TAG escaping rules** — these are the single biggest source of empty-result bugs. TAG values are *not* tokenized; hyphens, dots, commas, `@`, `:`, and spaces inside a tag must be escaped with a leading backslash, and the whole value lives inside `{}`.**

```python
# TAG with hyphen — must escape
FT.SEARCH idx:bicycle "@brand:{Giant\\-Cycles}"               DIALECT 2

# TAG with dot — must escape
FT.SEARCH idx:bicycle "@email:{user\\@example\\.com}"         DIALECT 2

# TAG with space — escape the space (or use double-quotes inside the braces)
FT.SEARCH idx:bicycle "@brand:{Trek\\ Bicycles}"              DIALECT 2
```

**Incorrect: Using `()` for TAG values, `{}` for TEXT, forgetting to escape hyphens, or mixing delimiters.**

```python
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

Reference: [https://redis.io/docs/latest/develop/interact/search-and-query/query/](https://redis.io/docs/latest/develop/interact/search-and-query/query/), [https://redis.io/docs/latest/develop/interact/search-and-query/query/#tokenization](https://redis.io/docs/latest/develop/interact/search-and-query/query/#tokenization)

### 5.10 Paginate Large Aggregations with FT.CURSOR

**Impact: MEDIUM (Cursors stream million-row aggregates without blowing memory; releasing them avoids server-side leaks)**

`FT.AGGREGATE ... LIMIT 0 1000000` materializes the whole result on the server before responding. For large aggregates (millions of groups, long fan-outs), use `WITHCURSOR` and stream batches via `FT.CURSOR READ`. Cursors that aren't read or deleted live until `MAXIDLE` elapses and then are GC'd — explicitly `FT.CURSOR DEL` when you're done.

**Correct: Open a cursor, drain it in batches, release it.**

```python
# Open the cursor — COUNT 1000 = up to 1000 rows per batch, MAXIDLE in ms
FT.AGGREGATE idx:bicycle "*"
    GROUPBY 1 @brand
        REDUCE COUNT 0 AS bike_count
    SORTBY 2 @bike_count DESC
    WITHCURSOR COUNT 1000 MAXIDLE 30000
    DIALECT 2
# → reply: { rows..., cursor_id: 12345 }   (cursor_id = 0 means exhausted)

# Pull the next batch
FT.CURSOR READ idx:bicycle 12345 COUNT 1000
# → reply: { rows..., cursor_id: 12345 or 0 }

# Release explicitly when you stop early — don't wait for MAXIDLE
FT.CURSOR DEL idx:bicycle 12345
```

**Cursor lifecycle:**

- `COUNT n` — max rows per response (the server may return fewer).

- `MAXIDLE ms` — server discards the cursor after this idle time. Default is server-config-dependent (typically 30s).

- A returned `cursor_id` of `0` means the result set is fully drained.

- Cursors are scoped to a specific index; the read/del calls take both `<index>` and `<cursor_id>`.

**Incorrect: Leaking cursors or trying to paginate aggregates with `LIMIT offset n` for large `n`.**

```python
# Bad: LIMIT 1000000 5000 — server must compute and skip the first million rows
FT.AGGREGATE idx:bicycle "*" GROUPBY 1 @brand REDUCE COUNT 0 AS n
    SORTBY 2 @n DESC
    LIMIT 1000000 5000
    DIALECT 2

# Bad: Open WITHCURSOR, take first batch, never call FT.CURSOR DEL.
# Cursor leaks until MAXIDLE; long-running ETL jobs accumulate them.
```

**When to use cursors:**

- Aggregations expected to return > ~10k rows.

- Streaming results into an ETL/export pipeline.

- Background analytics where you want bounded memory at both ends.

**When NOT needed:**

- Top-N analytics (`SORTBY ... LIMIT 0 100`) — the result fits in one response.

- Real-time dashboard queries where you only show the top page.

**Client mirrors:**

```java
// Jedis — STEP_START aggregate_cursor
import redis.clients.jedis.UnifiedJedis;
import redis.clients.jedis.search.aggr.AggregationBuilder;
import redis.clients.jedis.search.aggr.AggregationResult;
import redis.clients.jedis.search.aggr.Reducers;

try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    AggregationBuilder agg = new AggregationBuilder("*")
        .groupBy("@brand", Reducers.count().as("bike_count"))
        .cursor(1000, 30000)
        .dialect(2);
    AggregationResult res = jedis.ftAggregate("idx:bicycle", agg);
    long cursorId = res.getCursorId();
    while (cursorId != 0) {
        res = jedis.ftCursorRead("idx:bicycle", cursorId, 1000);
        cursorId = res.getCursorId();
    }
    // jedis.ftCursorDel("idx:bicycle", cursorId) if exiting early
}
// STEP_END
```

**Client mirrors — read exactly one:**

- For raw redis-py targets, read `references/clients/python-redis-py.md`.

- For Jedis (Java) targets, read `references/clients/java-jedis.md`.

- For RedisVL targets, read `references/clients/python-redisvl.md`.

- Do not read more than one client reference.

Upstream sources: No direct upstream example — authored from official Redis Search command documentation (https://redis.io/commands/ft.aggregate/, https://redis.io/commands/ft.cursor-read/, https://redis.io/commands/ft.cursor-del/).

Reference: [https://redis.io/docs/latest/commands/ft.aggregate/](https://redis.io/docs/latest/commands/ft.aggregate/), [https://redis.io/docs/latest/commands/ft.cursor-read/](https://redis.io/docs/latest/commands/ft.cursor-read/), [https://redis.io/docs/latest/commands/ft.cursor-del/](https://redis.io/docs/latest/commands/ft.cursor-del/)

### 5.11 Run KNN, Range, and Pre-Filtered Vector Queries

**Impact: HIGH (Correct vector-query syntax avoids full-scan fallbacks and lets pre-filters cut search space 10–100x)**

Vector queries live inside `FT.SEARCH` as a `=>[KNN ...]` or `[VECTOR_RANGE ...]` clause. The query *expression* on the left side acts as a pre-filter; the vector clause then runs over the surviving candidate set, not the entire index. Forgetting to pre-filter is the most common cause of slow or low-recall vector queries.

`DIALECT 2` is required for the `=>[KNN ...]` attribute form. The vector blob is bound through `PARAMS` rather than inlined.

**Correct: KNN, range, and hybrid pre-filter forms against the canonical Bicycle dataset (vector field `description_embeddings`, dim 1536).**

```python
# Pure KNN — 10 nearest neighbours, no pre-filter
FT.SEARCH idx:bicycle "*=>[KNN 10 @description_embeddings $vec AS score]"
    SORTBY score
    PARAMS 2 vec "<vector_blob>"
    DIALECT 2

# Pre-filtered KNN — narrow by TAG + NUMERIC first, then KNN over survivors
FT.SEARCH idx:bicycle "(@type:{mountain} @price:[100 500])=>[KNN 10 @description_embeddings $vec AS score]"
    SORTBY score
    PARAMS 2 vec "<vector_blob>"
    RETURN 4 model brand price score
    DIALECT 2

# Range query — every doc within radius 0.5 (COSINE distance)
FT.SEARCH idx:bicycle "@description_embeddings:[VECTOR_RANGE 0.5 $vec]=>{$yield_distance_as: dist}"
    SORTBY dist
    PARAMS 2 vec "<vector_blob>"
    DIALECT 2

# Tune recall vs latency per query — HNSW only
FT.SEARCH idx:bicycle "*=>[KNN 10 @description_embeddings $vec EF_RUNTIME 200 AS score]"
    SORTBY score
    PARAMS 2 vec "<vector_blob>"
    DIALECT 2
```

**Why this matters:**

- `AS score` aliases the distance so you can `SORTBY` and `RETURN` it.

- `PARAMS` binds the binary vector blob — never inline it in the query string.

- The pre-filter prefix `(@type:{mountain} @price:[100 500])` is applied *before* the vector search, slashing the work for HNSW.

- `EF_RUNTIME` raises HNSW search effort per-query; the index-time `EF_CONSTRUCTION` is independent.

**Incorrect: Inlining the vector, omitting `DIALECT 2`, or running a wide-open KNN when you could pre-filter.**

```python
# Bad: no PARAMS — vector blob does not survive RESP encoding cleanly
FT.SEARCH idx:bicycle "*=>[KNN 10 @description_embeddings <raw-bytes>]" DIALECT 2

# Bad: forgot DIALECT 2 — older default rejects the attribute form
FT.SEARCH idx:bicycle "*=>[KNN 10 @description_embeddings $vec AS score]" PARAMS 2 vec "..."

# Bad: KNN over the whole index when a TAG pre-filter would cut 99% of candidates
FT.SEARCH idx:bicycle "*=>[KNN 10 @description_embeddings $vec AS score]"
    PARAMS 2 vec "..." DIALECT 2
```

**Hybrid lexical + vector ranking with explicit fusion (Redis ≥ 8.4.0): Use `FT.HYBRID` — see `search-command-selection.md`. The pre-filter pattern above is still the right tool for *filter-narrowed* vector search; `FT.HYBRID` is for *blended ranking* with RRF or LINEAR fusion.**

**Client mirrors:**

```java
// Jedis — STEP_START vector_query
// Mirrors VectorSearchExample.java
import redis.clients.jedis.UnifiedJedis;
import redis.clients.jedis.search.Query;
import java.nio.ByteBuffer;
import java.nio.ByteOrder;

byte[] vecBlob = floatArrayToBytes(queryEmbedding);  // little-endian FLOAT32
try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    Query q = new Query(
        "(@type:{mountain} @price:[100 500])=>[KNN 10 @description_embeddings $vec AS score]")
        .setSortBy("score", true)
        .returnFields("model", "brand", "price", "score")
        .addParam("vec", vecBlob)
        .dialect(2)
        .limit(0, 10);
    jedis.ftSearch("idx:bicycle", q);
}
// STEP_END
```

**Client mirrors — read exactly one:**

- For raw redis-py targets, read `references/clients/python-redis-py.md`.

- For Jedis (Java) targets, read `references/clients/java-jedis.md`.

- For RedisVL targets, read `references/clients/python-redisvl.md`.

- Do not read more than one client reference.

Upstream sources:

- redis-py: [`doctests/search_vss.py`](https://github.com/redis/redis-py/blob/master/doctests/search_vss.py), [`query_combined.py`](https://github.com/redis/redis-py/blob/master/doctests/query_combined.py)

- Jedis: [`VectorSearchExample.java`](https://github.com/redis/jedis/blob/master/src/test/java/io/redis/examples/VectorSearchExample.java)

Reference: [https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/vectors/](https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/vectors/), [https://redis.io/docs/latest/develop/interact/search-and-query/query/vector-search/](https://redis.io/docs/latest/develop/interact/search-and-query/query/vector-search/)

### 5.12 Shape Search Results with RETURN, SORTBY, HIGHLIGHT, SUMMARIZE

**Impact: MEDIUM (Returning only needed fields and using SORTABLE cuts payload + latency by 2–10x)**

By default `FT.SEARCH` returns full documents — expensive when you only need a few fields, or a count, or a UI-ready snippet. The result-shaping clauses (`RETURN`, `NOCONTENT`, `LIMIT`, `SORTBY`, `HIGHLIGHT`, `SUMMARIZE`) trim the response server-side and pre-format text for display.

**Correct: Shape the response to exactly what the caller needs.**

```python
# Count only — no documents returned
FT.SEARCH idx:bicycle "@type:{mountain}" LIMIT 0 0 DIALECT 2

# IDs only — NOCONTENT skips the field payload
FT.SEARCH idx:bicycle "@type:{mountain}" NOCONTENT LIMIT 0 20 DIALECT 2

# Specific fields only — RETURN n field1 field2 ...
FT.SEARCH idx:bicycle "@type:{mountain}"
    RETURN 3 model brand price
    LIMIT 0 20
    DIALECT 2

# Sort by an indexed field — requires SORTABLE on the field at FT.CREATE time
FT.SEARCH idx:bicycle "@type:{mountain}"
    SORTBY price ASC
    LIMIT 0 10
    RETURN 3 model brand price
    DIALECT 2

# Highlight matched terms with HTML tags
FT.SEARCH idx:bicycle "wireless"
    HIGHLIGHT FIELDS 1 description TAGS "<b>" "</b>"
    DIALECT 2

# Summarize: extract up to 3 fragments of 20 tokens each from @description
FT.SEARCH idx:bicycle "wireless"
    SUMMARIZE FIELDS 1 description FRAGS 3 LEN 20 SEPARATOR " ... "
    DIALECT 2
```

**Why these matter:**

- `RETURN n` is the single biggest perf win for wide schemas — typical 50% latency cut when you stop sending unused fields.

- `SORTBY` on a non-`SORTABLE` field falls back to a row-by-row sort over the result page; on a `SORTABLE NUMERIC` field it's near-free.

- `NOCONTENT` is what `FT.SEARCH` wants when you only need the matching keys (e.g., to pipeline a follow-up `MGET`).

- `LIMIT 0 0` is the canonical count idiom — total appears in position 0 of the reply.

- `HIGHLIGHT` and `SUMMARIZE` only operate on TEXT fields and assume the field was indexed without `NOOFFSETS`.

**Incorrect: Pagination with deep offsets, sorting non-SORTABLE fields at high LIMIT, fetching full docs to throw away most fields.**

```python
# Bad: deep pagination — server must scan + sort offset+page rows
FT.SEARCH idx:bicycle "*" LIMIT 100000 20

# Bad: SORTBY a TEXT field that wasn't marked SORTABLE — falls back to in-page sort
FT.SEARCH idx:bicycle "*" SORTBY description ASC LIMIT 0 1000

# Bad: fetching the entire doc when only 3 fields are used in the UI
FT.SEARCH idx:bicycle "*" LIMIT 0 50
```

**Pagination patterns:**

- Up to a few thousand rows: `LIMIT offset n` is fine.

- Beyond that, switch to **search-after** patterns (sort by a stable cursor like `@id` or `@created_at`, then `FILTER @id > $last` on the next page).

- For `FT.AGGREGATE` over very large result sets, use `WITHCURSOR` (see `search-aggregate-cursors.md`).

**Client mirrors:**

```java
// Jedis — STEP_START result_shaping
import redis.clients.jedis.UnifiedJedis;
import redis.clients.jedis.search.Query;

try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    Query q = new Query("@type:{mountain}")
        .returnFields("model", "brand", "price")
        .setSortBy("price", true)
        .limit(0, 20)
        .dialect(2);
    jedis.ftSearch("idx:bicycle", q);

    Query countOnly = new Query("@type:{mountain}").limit(0, 0).dialect(2);
    long total = jedis.ftSearch("idx:bicycle", countOnly).getTotalResults();
}
// STEP_END
```

**Client mirrors — read exactly one:**

- For raw redis-py targets, read `references/clients/python-redis-py.md`.

- For Jedis (Java) targets, read `references/clients/java-jedis.md`.

- For RedisVL targets, read `references/clients/python-redisvl.md`.

- Do not read more than one client reference.

Upstream sources:

- redis-py: covered across the upstream `doctests/query_*.py` set, including [`doctests/query_ft.py`](https://github.com/redis/redis-py/blob/master/doctests/query_ft.py)

- Jedis: covered across the upstream `Query*Example.java` set, including [`QueryFtExample.java`](https://github.com/redis/jedis/blob/master/src/test/java/io/redis/examples/QueryFtExample.java)

Reference: [https://redis.io/docs/latest/commands/ft.search/](https://redis.io/docs/latest/commands/ft.search/), [https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/highlight/](https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/highlight/)

### 5.13 Tune FT.CREATE Options for Memory and Indexing Cost

**Impact: LOW (Disabling unused index features can cut memory 20–50% on large indexes)**

`FT.CREATE` ships sensible defaults that pay for features most apps want — offsets for highlighting, frequencies for scoring, per-document field map for `FT.AGGREGATE LOAD`. On a very large index, those costs add up. Several flags let you opt out where you don't need them, and a few flags change the *behavior* of index creation itself (`SKIPINITIALSCAN`, `TEMPORARY`).

**Correct: Pick the flags whose trade-offs match your workload.**

```python
# A lean index — no highlight, no field-frequency scoring, no field map
FT.CREATE idx:logs ON HASH PREFIX 1 log:
    NOOFFSETS                       # don't store term offsets → no HIGHLIGHT/SUMMARIZE/phrase queries
    NOHL                            # disable highlight payload (subset of NOOFFSETS savings)
    NOFREQS                         # don't store term frequencies → lighter scoring
    NOFIELDS                        # don't store per-doc field bitmap → no @field-scoped queries
    SCHEMA
        message TEXT

# Only index new documents (skip the initial scan over existing keys)
FT.CREATE idx:events ON HASH PREFIX 1 event:
    SKIPINITIALSCAN
    SCHEMA
        topic TAG
        ts NUMERIC SORTABLE

# Pre-allocate room for FT.ALTER (cannot grow beyond MAXTEXTFIELDS slots later)
FT.CREATE idx:bicycle ON HASH PREFIX 1 bicycle:
    MAXTEXTFIELDS                   # reserves capacity for adding TEXT fields later
    SCHEMA
        model TEXT

# Custom stopword list (or disable entirely with STOPWORDS 0)
FT.CREATE idx:books ON HASH PREFIX 1 book:
    STOPWORDS 0                     # disable stopword filtering altogether
    SCHEMA
        title TEXT
        description TEXT

# Auto-expire the index if idle (in seconds) — useful for transient indexes
FT.CREATE idx:session_search ON HASH PREFIX 1 sess:
    TEMPORARY 3600
    SCHEMA
        user_id TAG
        last_query TEXT
```

**Trade-off table:**

| Flag | Saves | Costs |
|------|-------|-------|
| `NOOFFSETS` | Term offsets — can be 30–50% of TEXT-heavy index size. | Disables `HIGHLIGHT`, `SUMMARIZE`, and phrase queries with `$slop`/`$inorder`. |
| `NOHL` | Highlight payload only. | Disables `HIGHLIGHT` (offsets still kept for phrase queries). |
| `NOFREQS` | Per-term frequency counters. | Scoring quality degrades; BM25 / TFIDF can't differentiate doc relevance well. |
| `NOFIELDS` | Per-document field bitmap. | Disables `@field:` scoping on queries — every term searches all TEXT fields. |
| `SKIPINITIALSCAN` | Time + IO of scanning existing keys. | Existing matching documents are not in the index — only new HSET/JSON.SET. |
| `MAXTEXTFIELDS` | n/a (reserves capacity). | Slightly larger empty-index footprint. Use only if you'll add fields via `FT.ALTER`. |
| `STOPWORDS 0` | Stopword filtering. | Common words (the, and, of) are now searchable and inflate the inverted index. |
| `TEMPORARY <sec>` | n/a (sets a TTL on the index). | Index is reaped after `<sec>` of idleness — must be re-created. |

- Creating an index for a new feature where existing documents are irrelevant.

- Setting up an index ahead of a data load that will fully populate it.

- The dataset is too large for initial scan latency to be acceptable.

- Event-driven architectures that only care about new events going forward.

- You need historical documents to appear in search immediately.

- Migrating an existing dataset to a new schema (the new index must include all existing docs).

- Most general-purpose search use cases.

**Incorrect: Disabling features you actually use, or combining mutually destructive flags.**

```python
# Bad: NOOFFSETS on an index that highlights snippets in the UI.
FT.CREATE idx:blog ON HASH PREFIX 1 post:
    NOOFFSETS
    SCHEMA title TEXT body TEXT
# Later — fails or returns no highlights:
FT.SEARCH idx:blog "redis" HIGHLIGHT FIELDS 1 body

# Bad: NOFIELDS with field-scoped queries — every @-prefixed term becomes a global term
FT.CREATE idx:logs ON HASH PREFIX 1 log: NOFIELDS SCHEMA service TAG message TEXT
FT.SEARCH idx:logs "@service:{api}"     # no longer effective

# Bad: SKIPINITIALSCAN when migrating data into a new index
FT.CREATE idx:v2 ON HASH PREFIX 1 product: SKIPINITIALSCAN SCHEMA name TEXT
# Existing product:* keys are never indexed; queries return only new docs.
```

**Client mirrors:**

```java
// Jedis — STEP_START ft_create_options
import redis.clients.jedis.UnifiedJedis;
import redis.clients.jedis.search.FTCreateParams;
import redis.clients.jedis.search.IndexDataType;
import redis.clients.jedis.search.schemafields.*;

try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    jedis.ftCreate("idx:events",
        FTCreateParams.createParams().on(IndexDataType.HASH).prefix("event:").skipInitialScan(),
        TagField.of("topic"), NumericField.of("ts").sortable());

    jedis.ftCreate("idx:logs",
        FTCreateParams.createParams().on(IndexDataType.HASH).prefix("log:")
            .noOffsets().noFields().noFrequencies(),
        TextField.of("message"));
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

Reference: [https://redis.io/docs/latest/commands/ft.create/](https://redis.io/docs/latest/commands/ft.create/)

### 5.14 Use DIALECT 2 for Query Syntax

**Impact: MEDIUM (Ensures consistent query behavior and access to modern features)**

Pass `DIALECT 2` on every `FT.SEARCH` / `FT.AGGREGATE` / `FT.HYBRID` call. From Redis 8 onward, **DIALECT 2 is the only supported value** — dialects 1, 3, and 4 are deprecated and removed in current Redis Open Source. Vector query attributes (the `=>[KNN ...]` form) require DIALECT 2 to parse.

**Correct: Specify DIALECT 2 explicitly, or rely on modern client defaults.**

```python
# In raw commands, specify DIALECT 2 at the end
FT.SEARCH idx:bicycle "@model:hyperion" DIALECT 2

FT.AGGREGATE idx:bicycle "@type:{mountain}"
    GROUPBY 1 @brand
    REDUCE COUNT 0 AS bike_count
    DIALECT 2
```

**Note on Redis 8 and DIALECT: Redis 8 (built-in Redis Search) accepts only DIALECT 2. The `DEFAULT_DIALECT` `FT.CONFIG` knob no longer accepts other values. Older Redis 7.x / RediSearch-module deployments still respect dialect 1; if you target both, set `DIALECT 2` explicitly so behavior is identical across versions.**

**Why DIALECT 2:**

- Required for vector search (`=>[KNN ...]` attribute syntax).

- Required for `PARAMS` placeholder binding.

- Predictable handling of special characters and NULL-like missing fields.

- The only dialect that will be supported going forward.

**Incorrect: Relying on the server-side default with a client library that pins an older dialect.**

```python
# Bad: omitting DIALECT in a vector query with a legacy redis-py — falls back to DIALECT 1 and rejects =>[KNN ...]
FT.SEARCH idx:bicycle "*=>[KNN 10 @embedding $vec AS score]" PARAMS 2 vec "..."
```

**Client mirrors:**

```java
// Jedis — STEP_START dialect
// Mirrors SearchQuickstartExample.java
import redis.clients.jedis.UnifiedJedis;
import redis.clients.jedis.search.FTSearchParams;
import redis.clients.jedis.search.SearchResult;

try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    SearchResult res = jedis.ftSearch("idx:bicycle",
        "@model:hyperion",
        FTSearchParams.searchParams().dialect(2));
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

Reference: [https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/dialects/](https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/dialects/)

### 5.15 Write Performant Queries

**Impact: HIGH (Pre-filters, SORTABLE fields, and tight RETURN cut query latency by orders of magnitude)**

This rule is performance-focused — syntax details live in `search-query-syntax.md`, vector queries in `search-vector-query.md`, aggregate pipelines in `search-aggregate-pipeline.md`. The lever is the same in every case: narrow the candidate set as early as possible, return as little as possible, and use indexed sort paths.

**Correct: Pre-filter, sort on `SORTABLE` fields, return only what you use.**

```python
# Specific filters drop the candidate set before any scoring
FT.SEARCH idx:bicycle "@type:{mountain} @price:[100 500]"
    SORTBY price ASC                       # price is SORTABLE NUMERIC → near-free
    LIMIT 0 20
    RETURN 3 model brand price
    DIALECT 2

# Pre-filtered vector query — TAG + NUMERIC cut 99% of vectors before KNN
FT.SEARCH idx:bicycle "(@type:{mountain} @price:[100 500])=>[KNN 10 @description_embeddings $vec AS score]"
    SORTBY score
    PARAMS 2 vec "<vector_blob>"
    RETURN 4 model brand price score
    DIALECT 2
```

**The performance levers — in priority order:**

```python
# Diagnose a slow query
FT.PROFILE idx:bicycle SEARCH QUERY "@type:{mountain}" LIMIT 0 20

# See whether stemming/expansion is bloating the term list
FT.EXPLAIN idx:bicycle "running shoes"
```

1. **Narrow with TAG / NUMERIC predicates first.** They're cheaper than TEXT scoring and cut candidate counts dramatically. See `search-query-syntax.md` for syntax.

2. **`SORTBY` on `SORTABLE` fields.** Non-sortable sorting falls back to a row-by-row sort over the page. Mark `NUMERIC SORTABLE` and `TAG SORTABLE` on any field you'll order by.

3. **`LIMIT 0 n` aggressively.** Default page size returns 10; raising to 1000 is fine, raising to 100000 will hurt.

4. **`RETURN n f1 f2 ...`.** Stops Redis from materializing fields you'll throw away. Combine with `NOCONTENT` when you only need keys.

5. **`NOSTEM` and `TAG` over `TEXT` for identifiers.** Tokenization is expensive and easy to misconfigure (see `search-text-tokenization.md`).

6. **Profile, don't guess.** `FT.PROFILE` reports per-stage timing; `FT.EXPLAIN` shows how the parser interpreted the query (see `search-debugging.md`).

**Incorrect: Wildcard scans, deep pagination, sorting non-SORTABLE fields, dumping the full doc.**

```python
# Bad: wildcard scan over the whole index
FT.SEARCH idx:bicycle "*" LIMIT 0 10000

# Bad: deep offset pagination — server scans+sorts offset+page rows
FT.SEARCH idx:bicycle "*" LIMIT 100000 20

# Bad: SORTBY on a non-SORTABLE TEXT field at high LIMIT
FT.SEARCH idx:bicycle "*" SORTBY description ASC LIMIT 0 1000

# Bad: returning every field when only 3 are used downstream
FT.AGGREGATE idx:bicycle "*" LOAD *
```

**Client mirrors:**

```java
// Jedis — STEP_START query_perf
import redis.clients.jedis.UnifiedJedis;
import redis.clients.jedis.search.Query;
try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    Query q = new Query("@type:{mountain} @price:[100 500]")
        .setSortBy("price", true)
        .returnFields("model", "brand", "price")
        .limit(0, 20)
        .dialect(2);
    jedis.ftSearch("idx:bicycle", q);
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

Reference: [https://redis.io/docs/latest/develop/interact/search-and-query/query/](https://redis.io/docs/latest/develop/interact/search-and-query/query/), [https://redis.io/docs/latest/commands/ft.profile/](https://redis.io/docs/latest/commands/ft.profile/)

---

## 6. Vector Search & RedisVL

**Impact: HIGH**

Vector indexes, HNSW vs FLAT, hybrid search, and RAG patterns with RedisVL.

### 6.1 Choose HNSW vs FLAT Based on Requirements

**Impact: HIGH (HNSW gives ~95%+ recall at sub-millisecond latency; FLAT gives exact results but scales linearly)**

`HNSW` (Hierarchical Navigable Small World) is the production default: approximate nearest neighbour with tunable recall, sub-millisecond queries even on millions of vectors. `FLAT` is exact brute-force: 100% recall but linear scan cost — fine for thousands of vectors, not for millions.

| Algorithm | Speed | Accuracy | Memory | Best for |
|-----------|-------|----------|--------|----------|
| HNSW | Fast (approximate) | ~95%+ recall, tunable | Higher | Large datasets (> 10k vectors) |
| FLAT | Slow (exact) | 100% (exact) | Lower | Small datasets, accuracy-critical |

**Correct: HNSW** — use for large-scale production workloads.**

```python
# HNSW with tunable M and EF_CONSTRUCTION
FT.CREATE idx:bicycle ON HASH PREFIX 1 bicycle:
    SCHEMA
        description_embeddings VECTOR HNSW 10
            TYPE FLOAT32
            DIM 1536
            DISTANCE_METRIC COSINE
            M 16
            EF_CONSTRUCTION 200
```

**Correct: FLAT** — use when exact results are required and the dataset is small.**

```python
# FLAT — exact brute-force search, guaranteed accuracy
FT.CREATE idx:bicycle_small ON HASH PREFIX 1 bicycle_small:
    SCHEMA
        description_embeddings VECTOR FLAT 6
            TYPE FLOAT32
            DIM 1536
            DISTANCE_METRIC COSINE
```

**Tuning HNSW recall vs latency:**

- `M` (default 16) — graph connections per node. Higher = better recall, more memory. Practical range 8–64.

- `EF_CONSTRUCTION` (default 200) — build-time exploration depth. Higher = better graph quality, slower index build.

- `EF_RUNTIME` — per-query exploration depth. Set on the query itself (`...=>[KNN 10 @vec $vec EF_RUNTIME 200 AS score]`), not at index time. Higher = better recall, slower query.

**When to use FLAT:**

- Dataset under ~10k vectors and won't grow much.

- Recall must be exactly 100% (e.g., regulatory or evaluation/baseline use cases).

- You need predictable, deterministic results regardless of insert order.

**When NOT needed: use HNSW**

- Production semantic search, RAG retrieval, recommendation.

- Datasets above ~10k vectors where linear scan becomes expensive.

- Any case where 95%+ recall is acceptable.

**Incorrect: FLAT on a million-vector index, or under-tuning HNSW and then blaming recall.**

```python
# Bad: FLAT on 1M vectors — every query becomes a 1M-vector linear scan
FT.CREATE idx:big_vectors ON HASH PREFIX 1 doc:
    SCHEMA embedding VECTOR FLAT 6 TYPE FLOAT32 DIM 1536 DISTANCE_METRIC COSINE

# Bad: HNSW with default M=16 and EF_CONSTRUCTION=200 on a recall-critical workload —
# then logging poor recall instead of raising EF_RUNTIME at query time.
```

**Client mirrors:**

```java
// Jedis — STEP_START vector_algorithm
import redis.clients.jedis.search.schemafields.VectorField;
import java.util.Map;
VectorField hnsw = VectorField.builder()
    .fieldName("description_embeddings")
    .algorithm(VectorField.VectorAlgorithm.HNSW)
    .attributes(Map.of("TYPE", "FLOAT32", "DIM", 1536,
                       "DISTANCE_METRIC", "COSINE",
                       "M", 16, "EF_CONSTRUCTION", 200))
    .build();
VectorField flat = VectorField.builder()
    .fieldName("description_embeddings")
    .algorithm(VectorField.VectorAlgorithm.FLAT)
    .attributes(Map.of("TYPE", "FLOAT32", "DIM", 1536, "DISTANCE_METRIC", "COSINE"))
    .build();
// STEP_END
```

**Client mirrors — read exactly one:**

- For raw redis-py targets, read `references/clients/python-redis-py.md`.

- For Jedis (Java) targets, read `references/clients/java-jedis.md`.

- For RedisVL targets, read `references/clients/python-redisvl.md`.

- Do not read more than one client reference.

**RedisVL coverage: schema-dict examples for HNSW and FLAT live in `references/clients/python-redisvl.md` (forthcoming, spec 0004). Inline RedisVL is intentionally omitted here.**

Reference: [https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/vectors/](https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/vectors/)

### 6.2 Combine Lexical and Vector Search Correctly

**Impact: MEDIUM (Pre-filter + KNN works on every Redis 8.x; FT.HYBRID adds explicit rank fusion on Redis ≥ 8.4.0)**

Two patterns address two different needs:

- **Filter-narrowed vector search** (works on every Redis with vector support): write a normal `FT.SEARCH` with a TAG/NUMERIC pre-filter on the left side of the `=>[KNN ...]` clause. The pre-filter shrinks the candidate set; KNN then runs only over survivors.

- **Blended lexical + vector ranking with explicit fusion** (Redis ≥ 8.4.0): use `FT.HYBRID`, which runs a `SEARCH` leg and a `VSIM` leg in parallel and fuses their rankings via Reciprocal Rank Fusion (`COMBINE RRF`) or a weighted score blend (`COMBINE LINEAR`).

**Correct: pre-filtered KNN** (works on all Redis 8.x and the RediSearch module).**

```python
# Filter to mountain bikes under $500, then KNN over the survivors
FT.SEARCH idx:bicycle "(@type:{mountain} @price:[100 500])=>[KNN 10 @description_embeddings $vec AS score]"
    SORTBY score
    PARAMS 2 vec "<vector_blob>"
    RETURN 4 model brand price score
    DIALECT 2
```

**Correct: FT.HYBRID** — requires Redis ≥ 8.4.0.**

```python
# Blend lexical ("mountain bicycle") + vector similarity with RRF fusion
FT.HYBRID idx:bicycle
    SEARCH "mountain bicycle"
    VSIM @description_embeddings $vec
    KNN 2 K 10
    COMBINE RRF 10                         # RRF <count> — number of fused results to keep
    PARAMS 2 vec "<vector_blob>"
    LIMIT 0 10
    DIALECT 2

# Weighted (LINEAR) — α weights the SEARCH score, β the VSIM score
FT.HYBRID idx:bicycle
    SEARCH "mountain bicycle" YIELD_SCORE_AS lex_score
    VSIM @description_embeddings $vec YIELD_SCORE_AS vec_score
    KNN 2 K 20
    COMBINE LINEAR 4 ALPHA 0.4 BETA 0.6
    PARAMS 2 vec "<vector_blob>"
    DIALECT 2
```

**When to use which:**

| Goal | Use |
|------|-----|
| "Find vectors near $vec, but only within category X and price < $500." | Pre-filtered KNN inside `FT.SEARCH` (works everywhere). |
| "Rank documents by a blend of lexical relevance and semantic similarity." | `FT.HYBRID` (Redis ≥ 8.4.0). |
| "Same goal but on Redis < 8.4.0." | Run two separate queries client-side and fuse the rankings yourself (rough fallback; loses cross-leg score calibration). |

**Incorrect: Running an unfiltered KNN and then filtering client-side, or assuming `FT.HYBRID` exists on older Redis.**

```python
# Bad (client mirror): same anti-pattern in Python — fetch 1000, filter in memory.
results = r.ft("idx:bicycle").search(
    Query("*=>[KNN 1000 @description_embeddings $vec AS score]")
    .sort_by("score").dialect(2),
    query_params={"vec": vec_blob})
mountain = [r for r in results.docs if r.type == "mountain" and 100 <= int(r.price) <= 500]
```

**Performance notes:**

- Pre-filter with `TAG` and `NUMERIC` fields — these are cheap and dramatically cut the KNN candidate set.

- For `FT.HYBRID`, the `KNN <count> K <k>` clause inside `VSIM` controls how many vector neighbours feed the fusion stage; the outer `LIMIT` controls how many results you return.

- `COMBINE RRF` needs no tuning; `COMBINE LINEAR` needs calibrated α/β — start at 0.5/0.5 and adjust based on relevance evals.

**Client mirrors:**

```java
// Jedis — STEP_START hybrid_search
import redis.clients.jedis.UnifiedJedis;
import redis.clients.jedis.search.Query;

try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    Query q = new Query(
        "(@type:{mountain} @price:[100 500])=>[KNN 10 @description_embeddings $vec AS score]")
        .setSortBy("score", true)
        .returnFields("model", "brand", "price", "score")
        .addParam("vec", vecBlob)
        .dialect(2)
        .limit(0, 10);
    jedis.ftSearch("idx:bicycle", q);
}
// STEP_END
```

**Client mirrors — read exactly one:**

- For raw redis-py targets, read `references/clients/python-redis-py.md`.

- For Jedis (Java) targets, read `references/clients/java-jedis.md`.

- For RedisVL targets, read `references/clients/python-redisvl.md`.

- Do not read more than one client reference.

**RedisVL coverage: `VectorQuery` with filter expressions and the `HybridQuery` wrapper for FT.HYBRID live in `references/clients/python-redisvl.md` (forthcoming, spec 0004).**

Upstream sources:

- redis-py: [`doctests/query_combined.py`](https://github.com/redis/redis-py/blob/master/doctests/query_combined.py)

- Jedis: [`VectorSearchExample.java`](https://github.com/redis/jedis/blob/master/src/test/java/io/redis/examples/VectorSearchExample.java)

Reference: [https://redis.io/docs/latest/develop/interact/search-and-query/query/combined/](https://redis.io/docs/latest/develop/interact/search-and-query/query/combined/), [https://redis.io/docs/latest/commands/ft.hybrid/](https://redis.io/docs/latest/commands/ft.hybrid/)

### 6.3 Configure Vector Indexes Properly

**Impact: HIGH (Correct dimensions, algorithm, and distance metric are required for vector search to work at all)**

A vector field needs three things stated correctly at index time: `TYPE` (almost always `FLOAT32`), `DIM` (must equal your embedding model's output size), and `DISTANCE_METRIC` (`COSINE`, `L2`, or `IP`). Mismatching any of these silently produces wrong results or refuses inserts — there is no runtime warning.

For the algorithm choice (HNSW vs FLAT), see `vector-algorithm-choice.md`.

**Correct: Canonical CLI form against the Bicycle dataset — 1536-dim OpenAI-style embeddings on a HASH index.**

```python
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

**For JSON documents** the vector field is a JSONPath plus `AS alias` (see `search-json-indexing.md`):**

```python
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

**Incorrect: Dim mismatch, wrong metric for normalized embeddings, or inlining the vector blob at query time (use PARAMS — see `search-vector-query.md`).**

```python
# Bad: DIM mismatch — inserts silently truncated/padded, queries return junk
FT.CREATE idx:bicycle ON HASH PREFIX 1 bicycle:
    SCHEMA description_embeddings VECTOR HNSW 6 TYPE FLOAT32 DIM 768 DISTANCE_METRIC COSINE
# ... but the embeddings inserted are 1536 floats

# Bad: L2 on normalized embeddings — works but obscures interpretability (use COSINE)
```

**Verifying the index after creation:**

```python
FT.INFO idx:bicycle
# Look for "attributes" — confirm vector field shows correct DIM/TYPE/DISTANCE_METRIC
```

**Client mirrors:**

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

**RedisVL coverage: higher-level schema-from-dict and `SearchIndex` usage are covered in `references/clients/python-redisvl.md` (forthcoming, spec 0004). RedisVL examples are intentionally omitted from this rule — read the RedisVL reference when targeting that SDK.**

Upstream sources:

- redis-py: [`doctests/search_vss.py`](https://github.com/redis/redis-py/blob/master/doctests/search_vss.py)

- Jedis: [`VectorSearchExample.java`](https://github.com/redis/jedis/blob/master/src/test/java/io/redis/examples/VectorSearchExample.java)

Reference: [https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/vectors/](https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/vectors/)

### 6.4 Implement RAG Retrieval Against Redis Correctly

**Impact: HIGH (Proper retrieval shape (pre-filter, score alias, RETURN) directly determines LLM answer quality)**

A RAG pipeline against Redis is three steps: (1) store documents + embeddings in a HASH or JSON index, (2) embed the user's question with the same model, (3) run a KNN query that returns the top-k passages and their distance. Step 3 is where most quality bugs live — see `search-vector-query.md` for the canonical query form.

**Correct: minimal end-to-end pipeline.** The retrieval step is CLI-form first; the embedding/LLM steps are deliberately client-side.**

```python
# 1. Index, built once
FT.CREATE idx:bicycle ON HASH PREFIX 1 bicycle:
    SCHEMA
        description TEXT
        type TAG
        price NUMERIC SORTABLE
        description_embeddings VECTOR HNSW 6 TYPE FLOAT32 DIM 1536 DISTANCE_METRIC COSINE

# 2. Documents inserted with HSET (or JSON.SET for JSON indexes).
#    The vector field holds the raw FLOAT32 little-endian blob.

# 3. Retrieval — pre-filtered KNN, score aliased, only the fields the LLM needs returned
FT.SEARCH idx:bicycle "(@type:{mountain})=>[KNN 5 @description_embeddings $query_vec AS score]"
    SORTBY score
    PARAMS 2 query_vec "<query_vector_blob>"
    RETURN 3 description type score
    DIALECT 2
```

**End-to-end pattern (redis-py):**

```python
# redis-py — STEP_START rag_pipeline
# Distilled from doctests/search_vss.py
import numpy as np
from redis import Redis
from redis.commands.search.query import Query

r = Redis()

def embed(text: str) -> bytes:
    # Replace with your model — must produce the SAME dim as the index (1536 here)
    return np.array(embed_model.encode(text), dtype=np.float32).tobytes()

def retrieve(question: str, k: int = 5, type_filter: str = "mountain"):
    q = (Query(f"(@type:{{{type_filter}}})=>[KNN {k} @description_embeddings $vec AS score]")
         .sort_by("score").return_fields("description", "type", "score")
         .dialect(2).paging(0, k))
    return r.ft("idx:bicycle").search(q, query_params={"vec": embed(question)})

passages = retrieve("lightweight mountain bicycle for trails")
context = "\n\n".join(d.description for d in passages.docs)
# Pass `context` + question to your LLM of choice.
# STEP_END
```

**End-to-end pattern (Jedis):**

```java
// Jedis — STEP_START rag_pipeline
import redis.clients.jedis.UnifiedJedis;
import redis.clients.jedis.search.Query;

try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    byte[] vec = embed("lightweight mountain bicycle for trails"); // FLOAT32 little-endian
    Query q = new Query("(@type:{mountain})=>[KNN 5 @description_embeddings $vec AS score]")
        .setSortBy("score", true)
        .returnFields("description", "type", "score")
        .addParam("vec", vec)
        .dialect(2)
        .limit(0, 5);
    var result = jedis.ftSearch("idx:bicycle", q);
    // Build the prompt from result.getDocuments() and call your LLM.
}
// STEP_END
```

**Retrieval-quality checklist:**

- Normalize embeddings if the model isn't already producing unit vectors and you use `COSINE`.

- Use a pre-filter (TAG/NUMERIC) before `=>[KNN ...]` when the user supplies categorical or range constraints — see `search-vector-query.md`.

- Return only the fields the LLM consumes (the score alias + the passage text). Returning the embedding wastes bandwidth.

- Chunk long documents to a size near the embedding model's effective context (e.g., 200–500 tokens) before indexing — retrieval quality drops sharply on chunks too large for the embedding model.

- Re-embedding the corpus after a model change is mandatory — you cannot mix embeddings from different models in the same index.

**Incorrect: Returning everything and filtering client-side, mismatched embedding models, or skipping the pre-filter.**

```python
# Bad: client-side filter wastes vector work
results = r.ft("idx:bicycle").search(
    Query("*=>[KNN 1000 @description_embeddings $vec AS score]")
    .sort_by("score").dialect(2),
    query_params={"vec": vec_blob})
mountain = [d for d in results.docs if d.type == "mountain"][:5]

# Bad: question embedded with model A, corpus embedded with model B — distances meaningless
```

**Client mirrors — read exactly one:**

- For raw redis-py targets, read `references/clients/python-redis-py.md`.

- For Jedis (Java) targets, read `references/clients/java-jedis.md`.

- For RedisVL targets, read `references/clients/python-redisvl.md`.

- Do not read more than one client reference.

**RedisVL coverage: `SearchIndex.load()` for bulk doc + embedding insertion and `VectorQuery` end-to-end pipelines live in `references/clients/python-redisvl.md` (forthcoming, spec 0004). RedisVL pipeline examples are intentionally omitted here.**

Cross-links: `search-vector-query.md` (KNN syntax in depth), `vector-index-creation.md`, `vector-hybrid-search.md`.

Reference: [https://redis.io/docs/latest/develop/get-started/rag/](https://redis.io/docs/latest/develop/get-started/rag/), [https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/vectors/](https://redis.io/docs/latest/develop/interact/search-and-query/advanced-concepts/vectors/)

---

## 7. Semantic Caching

**Impact: MEDIUM**

LangCache for LLM response caching, distance thresholds, and cache strategies.

### 7.1 Configure Semantic Cache Properly

**Impact: MEDIUM (Correct threshold tuning balances hit rate vs accuracy)**

> **Note:** LangCache is currently in preview on Redis Cloud. Features and behavior may change.

Tune similarity threshold and cache separation for optimal LangCache results.

**Correct: Tune similarity threshold for your use case.**

```python
from langcache import LangCache

lang_cache = LangCache(
    server_url=f"https://{os.getenv('HOST')}",
    cache_id=os.getenv("CACHE_ID"),
    api_key=os.getenv("API_KEY")
)

# Stricter matching - fewer false positives (0.95 = very similar)
result = lang_cache.search(
    prompt="What is Redis?",
    similarity_threshold=0.95
)

# Looser matching - higher hit rate (0.8 = somewhat similar)
result = lang_cache.search(
    prompt="What is Redis?",
    similarity_threshold=0.8
)
```

**Correct: Use separate caches for different use cases.**

```python
# Create different cache IDs in Redis Cloud for different LLM tasks
support_cache = LangCache(
    server_url=server_url,
    cache_id="support-cache-id",
    api_key=api_key
)

code_cache = LangCache(
    server_url=server_url,
    cache_id="code-cache-id",
    api_key=api_key
)
```

**Incorrect: Using a single cache for all LLM tasks.**

```python
# All tasks share one cache - responses may not be relevant
result = lang_cache.search(prompt="How do I reset my password?")
# Could return a code snippet if someone asked a similar coding question
```

**Best practices:**

- Start with threshold 0.9, adjust based on your use case

- Use custom attributes to filter results within a single cache

- Monitor cache hit rates to evaluate effectiveness

- Use separate cache IDs for fundamentally different LLM tasks

Reference: [https://redis.io/docs/latest/develop/ai/langcache/](https://redis.io/docs/latest/develop/ai/langcache/)

### 7.2 Use LangCache for LLM Response Caching

**Impact: HIGH (Reduces LLM API costs by 50-90% for similar queries)**

> **Note:** LangCache is currently in preview on Redis Cloud. Features and behavior may change.

LangCache is a fully-managed semantic caching service on Redis Cloud that reduces LLM costs and latency.

**How it works:**

1. Your app sends a prompt to LangCache via `POST /v1/caches/{cacheId}/entries/search`

2. LangCache generates an embedding and searches for similar cached responses

3. If found (cache hit), returns the cached response instantly

4. If not found (cache miss), your app calls the LLM and stores the response

**Correct: Use the LangCache Python SDK.**

```python
from langcache import LangCache
import os

lang_cache = LangCache(
    server_url=f"https://{os.getenv('HOST')}",
    cache_id=os.getenv("CACHE_ID"),
    api_key=os.getenv("API_KEY")
)

# Search for cached response
result = lang_cache.search(
    prompt="What is Redis?",
    similarity_threshold=0.9
)

if result:
    response = result[0]["response"]
else:
    response = llm.generate("What is Redis?")
    # Store for future queries
    lang_cache.set(
        prompt="What is Redis?",
        response=response
    )
```

**LangCache REST API:**

```bash
# Search cache
curl -X POST "https://$HOST/v1/caches/$CACHE_ID/entries/search" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "What is Redis?"}'

# Store a response
curl -X POST "https://$HOST/v1/caches/$CACHE_ID/entries" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "What is Redis?", "response": "Redis is an in-memory database..."}'
```

**With custom attributes for filtering:**

```python
# Store with attributes
lang_cache.set(
    prompt="What is Redis?",
    response="Redis is an in-memory database...",
    attributes={"category": "database", "version": "v1"}
)

# Search with attribute filter
result = lang_cache.search(
    prompt="Tell me about Redis",
    attributes={"category": "database"},
    similarity_threshold=0.9
)
```

Reference: [https://redis.io/docs/latest/develop/ai/langcache/](https://redis.io/docs/latest/develop/ai/langcache/)

---

## 8. Streams & Pub/Sub

**Impact: MEDIUM**

Choosing between Streams and Pub/Sub for messaging patterns.

### 8.1 Choose Streams vs Pub/Sub Appropriately

**Impact: MEDIUM (Wrong choice leads to lost messages or unnecessary complexity)**

Redis supports two messaging approaches for different use cases.

**Incorrect: Using Pub/Sub when messages must not be lost.**

```python
# Pub/Sub - messages lost if no subscribers connected
r.publish("orders", json.dumps(order))  # Fire and forget!
```

**Correct: Use Streams when message durability matters.**

```python
# Streams - messages persist and can be replayed
r.xadd("orders:stream", {"order": json.dumps(order)})

# Consumer group for reliable processing
r.xreadgroup("workers", "worker-1", {"orders:stream": ">"}, count=10)
r.xack("orders:stream", "workers", message_id)
```

| Requirement | Use |
|-------------|-----|
| Real-time notifications, OK to miss messages | Pub/Sub |
| Messages must not be lost | Streams |
| Need to replay/reprocess messages | Streams |
| Multiple workers processing same queue | Streams (consumer groups) |
| Simple broadcast to connected clients | Pub/Sub |
| Event sourcing or audit trail | Streams |

Reference: [https://redis.io/docs/latest/develop/data-types/streams/](https://redis.io/docs/latest/develop/data-types/streams/)

---

## 9. Clustering & Replication

**Impact: MEDIUM**

Hash tags for key colocation, read replicas, and cluster-aware patterns.

### 9.1 Use Hash Tags for Multi-Key Operations

**Impact: HIGH (Enables multi-key operations in Redis Cluster)**

In Redis Cluster, keys are distributed across slots based on their hash. Use hash tags to ensure keys that must be used together in [multi-key operations](https://redis.io/docs/latest/operate/rs/databases/durability-ha/clustering/#multikey-operations) are on the same slot.

**Correct: Use hash tags for keys used in multi-key operations.**

**Python** (redis-py):**

```python
# These keys go to the same slot because {user:1001} is the hash tag
redis.set("{user:1001}:profile", "...")
redis.set("{user:1001}:settings", "...")
redis.set("{user:1001}:cart", "...")

# Now you can use transactions and pipelines
pipe = redis.pipeline()
pipe.get("{user:1001}:profile")
pipe.get("{user:1001}:settings")
pipe.execute()

# Multi-key commands also work
redis.lmove("{user:1001}:pending", "{user:1001}:processed", "LEFT", "RIGHT")
```

**Java** (Jedis):**

```java
import redis.clients.jedis.UnifiedJedis;
import java.util.Set;

try (UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379")) {
    // Hash tags ensure keys go to the same slot
    jedis.sadd("{bikes:racing}:france", "bike:1", "bike:2", "bike:3");
    jedis.sadd("{bikes:racing}:usa", "bike:1", "bike:4");

    // Multi-key operation works because of matching hash tags
    Set<String> result = jedis.sdiff("{bikes:racing}:france", "{bikes:racing}:usa");
}
```

**Incorrect: Keys without hash tags that need multi-key operations.**

**Python** (redis-py):**

```python
# Bad: These may be on different slots
redis.set("user:1001:profile", "...")  # No hash tag
redis.set("user:1001:settings", "...")

# This will fail in cluster mode
pipe = redis.pipeline()
pipe.get("user:1001:profile")
pipe.get("user:1001:settings")
pipe.execute()  # CROSSSLOT error
```

**Java** (Jedis):**

```java
// Bad: No hash tags - keys may be on different slots
jedis.sadd("bikes:racing:france", "bike:1", "bike:2", "bike:3");
jedis.sadd("bikes:racing:usa", "bike:1", "bike:4");

// This will fail in cluster mode with CROSSSLOT error
Set<String> result = jedis.sdiff("bikes:racing:france", "bikes:racing:usa");
```

**Hash tag rules:**

- Only the part between `{` and `}` is hashed for slot assignment

- Use meaningful identifiers like `{user:1001}` not just `{1001}` to avoid unrelated keys (e.g., `purchase:{1001}`, `employee:{1001}`) saturating the same slot

- Use hash tags only where multi-key operations are needed, not as a general habit

Reference: [https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/#hash-tags](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/#hash-tags)

### 9.2 Use Read Replicas for Read-Heavy Workloads

**Impact: MEDIUM (Scales read throughput without adding primary nodes)**

For read-heavy workloads, distribute reads across replicas to reduce load on primaries.

**Correct: Configure replica reads in Redis Cluster.**

```python
from redis.cluster import RedisCluster

rc = RedisCluster(
    host='localhost',
    port=6379,
    read_from_replicas=True  # Distribute reads to replicas
)

# Writes go to primary
rc.set("key", "value")

# Reads can be served by replicas (eventually consistent)
value = rc.get("key")
```

**Correct: Use replica reads in standalone replication setup.**

```python
from redis import Redis

# Connect to primary for writes
primary = Redis(host='primary-host', port=6379)

# Connect to replica for reads
replica = Redis(host='replica-host', port=6379)

# Write to primary
primary.set("key", "value")

# Read from replica (eventually consistent)
value = replica.get("key")
```

**Considerations:**

- Replica reads are eventually consistent

- Don't read from replicas for data that was just written

- Use for read-heavy, slightly-stale-OK workloads (caches, analytics, dashboards)

Reference: [https://redis.io/docs/latest/operate/oss_and_stack/management/replication/](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/)

---

## 10. Security

**Impact: HIGH**

Authentication, ACLs, TLS, and network security.

### 10.1 Always Use Authentication in Production

**Impact: HIGH (Prevents unauthorized access to your data)**

Never run Redis without authentication in production environments.

**Correct: Use password and TLS.**

**Python** (redis-py):**

```python
r = redis.Redis(
    host='localhost',
    port=6379,
    password='your-strong-password',
    ssl=True,
    ssl_cert_reqs='required'
)
```

**Java** (Jedis):**

```java
import redis.clients.jedis.*;
import javax.net.ssl.*;
import java.security.KeyStore;

// Create SSL context with trust store and key store
KeyStore trustStore = KeyStore.getInstance("jks");
trustStore.load(new FileInputStream("./truststore.jks"), "password".toCharArray());

TrustManagerFactory tmf = TrustManagerFactory.getInstance("X509");
tmf.init(trustStore);

SSLContext sslContext = SSLContext.getInstance("TLS");
sslContext.init(null, tmf.getTrustManagers(), null);

JedisClientConfig config = DefaultJedisClientConfig.builder()
    .ssl(true)
    .sslSocketFactory(sslContext.getSocketFactory())
    .user("redisUser")
    .password("redisPassword")
    .build();

JedisPooled jedis = new JedisPooled(new HostAndPort("redis-host", 6379), config);
```

**Incorrect: Connecting without authentication.**

**Python** (redis-py):**

```python
# Bad: No authentication
r = redis.Redis(host='localhost', port=6379)
```

**Java** (Jedis):**

```java
// Bad: No authentication or TLS
UnifiedJedis jedis = new UnifiedJedis("redis://localhost:6379");
```

**Configuration:**

```python
# redis.conf
requirepass your-strong-password
tls-port 6380
tls-cert-file /path/to/redis.crt
tls-key-file /path/to/redis.key
```

Reference: [https://redis.io/docs/latest/operate/oss_and_stack/management/security/](https://redis.io/docs/latest/operate/oss_and_stack/management/security/)

### 10.2 Secure Network Access

**Impact: HIGH (Reduces attack surface and prevents unauthorized access)**

Restrict network access to Redis to only trusted sources.

**Correct: Bind to specific interfaces.**

```python
# redis.conf
bind 127.0.0.1 192.168.1.100
protected-mode yes
```

**Correct: Use firewall rules.**

```bash
# Allow only application servers
iptables -A INPUT -p tcp --dport 6379 -s 192.168.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 6379 -j DROP
```

**Incorrect: Exposing Redis to the internet.**

```python
# Bad: Binds to all interfaces
bind 0.0.0.0
protected-mode no
```

**Security checklist:**

```python
# Disable dangerous commands
rename-command FLUSHALL ""
rename-command DEBUG ""
rename-command CONFIG ""
```

- Use TLS for connections

- Bind to specific interfaces, not `0.0.0.0`

- Use firewall rules to restrict access

- Disable dangerous commands in production

Reference: [https://redis.io/docs/latest/operate/oss_and_stack/management/security/](https://redis.io/docs/latest/operate/oss_and_stack/management/security/)

### 10.3 Use ACLs for Fine-Grained Access Control

**Impact: HIGH (Limits blast radius if credentials are compromised)**

Create users with only the permissions they need (principle of least privilege).

**Correct: Create specific users with limited permissions.**

```python
# Read-only user for cache access
ACL SETUSER app_readonly on >password ~cache:* +get +mget +scan

# Writer that can't run dangerous commands
ACL SETUSER app_writer on >password ~* +@all -@dangerous

# Admin user (use sparingly)
ACL SETUSER admin on >strong-password ~* +@all
```

**Incorrect: Using the default user for everything.**

```python
# Bad: Single password for all access
requirepass shared-password
```

**ACL categories:**

- `@read` - Read commands

- `@write` - Write commands

- `@dangerous` - Commands like FLUSHALL, DEBUG

- `@admin` - Administrative commands

Reference: [https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/](https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/)

---

## 11. Observability

**Impact: MEDIUM**

SLOWLOG, INFO, MEMORY commands, monitoring metrics, and Redis Insight.

### 11.1 Monitor Key Redis Metrics

**Impact: MEDIUM (Early detection of performance and capacity issues)**

Track these metrics to catch issues before they impact users.

| Metric | What It Tells You | Alert When |
|--------|-------------------|------------|
| `used_memory` | Current memory usage | > 80% of maxmemory |
| `connected_clients` | Number of connections | Sudden spikes or drops |
| `blocked_clients` | Clients waiting on blocking ops | > 0 sustained |
| `instantaneous_ops_per_sec` | Current throughput | Significant drops |
| `keyspace_hits/misses` | Cache hit ratio | Hit ratio < 80% |
| `rejected_connections` | Connection limit issues | > 0 |
| `rdb_last_save_time` | Last persistence snapshot | Too old |

**Correct: Export metrics to your monitoring system.**

```python
# Get key metrics
info = redis.info()
print(f"Memory: {info['used_memory_human']}")
print(f"Connections: {info['connected_clients']}")
print(f"Ops/sec: {info['instantaneous_ops_per_sec']}")
print(f"Hit ratio: {info['keyspace_hits'] / (info['keyspace_hits'] + info['keyspace_misses']) * 100:.1f}%")
```

**Redis Insight:**

Use Redis Insight for visual monitoring, query profiling, and debugging. It includes Redis Copilot for natural language queries.

Reference: [https://redis.io/insight/](https://redis.io/insight/)

### 11.2 Use Observability Commands for Debugging

**Impact: MEDIUM (Enables quick diagnosis of performance issues)**

Redis provides built-in commands for monitoring and debugging.

**Key commands:**

```python
# Slow query log - find slow commands
SLOWLOG GET 10
SLOWLOG LEN
SLOWLOG RESET

# Server info - comprehensive stats
INFO all
INFO memory
INFO stats
INFO replication
INFO clients

# Memory analysis
MEMORY DOCTOR
MEMORY STATS
MEMORY USAGE mykey

# Client connections
CLIENT LIST
CLIENT INFO

# Index info (Search)
FT.INFO idx:products
FT.PROFILE idx:products SEARCH QUERY "@name:laptop"
```

**Correct: Check SLOWLOG regularly.**

```python
# Get recent slow queries
slow_queries = redis.slowlog_get(10)
for query in slow_queries:
    print(f"Duration: {query['duration']}μs, Command: {query['command']}")
```

Reference: [https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/)

---

## References

1. [https://redis.io/docs/](https://redis.io/docs/)
2. [https://redis.io/docs/latest/develop/interact/search-and-query/](https://redis.io/docs/latest/develop/interact/search-and-query/)
3. [https://redis.io/docs/latest/develop/clients/redisvl/](https://redis.io/docs/latest/develop/clients/redisvl/)
4. [https://redis.io/docs/latest/develop/ai/langcache/](https://redis.io/docs/latest/develop/ai/langcache/)
5. [https://redis.io/commands/](https://redis.io/commands/)

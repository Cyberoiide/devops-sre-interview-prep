# Q220 — Solution and walkthrough

## The one-paragraph answer

Both are in-memory key-value stores, but they differ on three axes that drive
design decisions. **Availability/durability**: Redis has persistence (RDB
snapshots and/or AOF) and replication and is often treated as a store you expect
to survive restarts; Memcached is a pure ephemeral cache where data loss on
restart is accepted by design. **Data model**: Memcached is flat string
key→value only; Redis has rich server-side types (lists, sets, sorted sets,
hashes, streams, bitmaps, HyperLogLog, geo) with atomic operations. **Threading**:
Memcached is multi-threaded and scales command processing across all cores on a
box; classic Redis executes commands on a single thread (Redis 6+ added threaded
*I/O*, but command execution stayed single-threaded). Default to Redis for
almost everything; **choose Memcached when** you want a simple, huge, pure
look-aside cache of opaque blobs at very high concurrency on a big multi-core
box, and you need none of Redis's data types, persistence, or replication — its
multi-threaded simplicity is then an asset, not a limitation.

---

## The three differences that actually matter

### 1. Availability and durability expectation

- **Memcached** holds everything in volatile memory with **no persistence**. A
  restart, crash, or eviction means the data is gone, and that's fine — it's a
  cache in front of a source of truth, and you re-warm from that source. The
  operational model is "it can disappear at any time".
- **Redis** offers durability *as a choice*: **RDB** point-in-time snapshots,
  **AOF** (append-only log, `appendfsync everysec` being the common setting), or
  both. Combined with **replication** (primary + replicas) and **Sentinel** or
  **Cluster** for failover, Redis is routinely run as a store you expect to
  survive restarts and node loss. That's why teams often hold Redis to a higher
  availability bar and use it for more than caching (queues, leaderboards,
  session stores, rate limiters, locks).

The design consequence: if losing the data on restart is unacceptable, that's a
point for Redis (or at least means Memcached needs a durable backing store it
can always rebuild from).

### 2. Data model

- **Memcached**: one type — an opaque byte string keyed by a string. Any
  structure lives in *your application*: to keep a list or object you serialize
  a blob (JSON/protobuf) and, to change it, you read the whole blob, modify it,
  and write it back — extra round trips, extra bandwidth, and a lost-update race
  unless you use **CAS** (compare-and-swap). It does have atomic `incr`/`decr`
  on numeric strings.
- **Redis**: rich types executed **server-side and atomically** — strings,
  lists, sets, sorted sets (the leaderboard question's ZSET), hashes, streams,
  bitmaps, HyperLogLog, geospatial. Plus pub/sub, transactions (`MULTI`/`EXEC`),
  Lua scripting, and keyspace notifications.

The design consequence: the moment your workload wants a structure — a
leaderboard, a queue, set-membership, a counter you increment concurrently, a
capped list — Redis does it in one atomic server-side op while Memcached forces
blob rewrites. This axis alone decides most non-trivial cases in Redis's favor.

### 3. Threading / concurrency model

- **Memcached** is **multi-threaded**: it uses a thread pool (configurable with
  `-t`) and can process many concurrent requests across all CPU cores on the
  box. For a very high-parallelism, simple KV read/write load, one big Memcached
  instance can saturate a multi-core machine.
- **Classic Redis** executes **commands on a single thread**. This is by design
  — it makes every command effectively atomic and avoids lock complexity, and
  because Redis operations are so cheap the single thread rarely bottlenecks.
  **Redis 6+ added threaded I/O** (reading/parsing requests and writing replies
  can use multiple threads), but **command execution itself remains
  single-threaded**. To use more cores with Redis you run **multiple instances /
  Redis Cluster** and shard across them.

The design consequence: on a single large multi-core box with a simple, hugely
parallel pure-KV workload, Memcached can post higher aggregate throughput per
instance. With Redis you'd scale that by sharding across instances/cluster. In
practice the raw-QPS gap is often smaller than folklore claims and is
workload-specific — but the architectural reason is real.

---

## When to pick Memcached (the interesting question)

Choose **Memcached** when *all* of these hold:

- The workload is a **simple, pure look-aside cache** of opaque values (rendered
  fragments, serialized objects, query results) — no need for structures.
- You need **none** of: persistence, replication, pub/sub, scripting, data
  types, or exact rank/queue semantics.
- You want **maximum simple-KV throughput on a single big multi-core instance**,
  where Memcached's multi-threading is an advantage and its slab allocator gives
  predictable memory behavior.
- **Data loss on restart is fully acceptable** because you always re-warm from
  the source of truth.

Canonical example: a large read-heavy web app fronting a database with a big,
flat cache of serialized objects/HTML fragments, running on beefy multi-core
cache nodes, where every value is an opaque blob and the cache is genuinely
disposable. This is the environment Memcached was born in (LiveJournal,
Facebook's early cache tier) and where its simplicity and multi-core throughput
shine.

**Choose Redis** (the default) when you want anything more: data structures,
durability, replication/HA, pub/sub, atomic server-side operations, Lua,
streams, or when the same store doubles as a queue / leaderboard / session store
/ rate limiter / lock manager. For most modern systems that "anything more" is
true, which is why Redis is the default.

---

## Secondary differences worth mentioning

- **Eviction**: Redis offers explicit policies (`noeviction`, `allkeys-lru`,
  `allkeys-lfu`, `volatile-lru`, `volatile-ttl`, `allkeys-random`, ...) via
  `maxmemory-policy`. Memcached does automatic LRU within **slab classes**.
- **Memory allocation**: Memcached uses a **slab allocator** (fixed-size chunk
  classes) — no external fragmentation, but possible internal waste and "slab
  calcification" if item sizes shift over time. Redis uses a general allocator
  (jemalloc by default) — more flexible, can fragment.
- **Max value size**: Memcached defaults to a **1 MB** item limit (tunable);
  Redis strings go up to 512 MB and its structures can be large.
- **Clustering**: Memcached has no built-in clustering — sharding is
  **client-side** (consistent hashing across a static list of nodes). Redis has
  **Redis Cluster** with automatic sharding (hash slots), resharding, and
  failover built in.
- **Multi-threaded Redis forks**: KeyDB and Dragonfly are Redis-protocol-
  compatible servers that are multi-threaded; worth a mention if the interviewer
  pushes on Redis's single-threaded limitation, but they're separate projects.

---

## Decision table

| Need                                        | Pick        |
|---------------------------------------------|-------------|
| Data structures (lists/sets/ZSet/hash/...)  | **Redis**   |
| Persistence / survive restart               | **Redis**   |
| Replication, failover, HA                   | **Redis**   |
| Pub/sub, streams, Lua, atomic ops           | **Redis**   |
| Built-in clustering & resharding            | **Redis**   |
| Simple opaque-blob cache, max QPS on one big multi-core box | **Memcached** |
| Data loss on restart is acceptable, no structures needed    | **Memcached** |

The headline for the interview: **default Redis; reach for Memcached only for a
simple, huge, disposable, multi-core pure-KV cache** where its multi-threaded
simplicity is a genuine advantage and you need nothing Redis uniquely provides.

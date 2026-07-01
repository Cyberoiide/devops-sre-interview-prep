# Q220 — Interviewer follow-ups

**Q: "Redis is single-threaded" — is that still true in Redis 6+?**
Command *execution* is still single-threaded, and that's deliberate: it makes
each command atomic and avoids locking. What Redis 6 added is threaded **I/O** —
reading/parsing incoming requests and writing replies can be spread across
threads, which relieves the network syscall bottleneck at high connection
counts. But the actual command logic still runs on one thread. To use more cores
for command processing you run multiple instances / Redis Cluster and shard. If
someone wants genuinely multi-threaded command execution with the Redis
protocol, that's KeyDB or Dragonfly, which are separate projects.

**Q: If Memcached is multi-threaded and Redis isn't, why is Redis usually
faster/preferred?**
Because Redis operations are so cheap that a single core rarely saturates before
the network does, and Redis's richer types mean you often replace *several*
Memcached round trips (read blob, modify, write blob) with *one* atomic
server-side op — fewer round trips beats more threads for most workloads.
Memcached's threading advantage only materializes for simple KV at very high
concurrency on a big box, and even then the gap is workload-specific and often
modest. Redis is "preferred" for its capabilities, not raw single-op speed.

**Q: Give a concrete workload where Memcached is the better choice.**
A large read-heavy web tier caching opaque serialized objects or rendered HTML
fragments in front of a database, running on big multi-core cache nodes, where
every value is a disposable blob, you need no structures/persistence/replication,
and you want to squeeze maximum simple-KV QPS out of each instance. That's
Memcached's home turf (its LiveJournal/early-Facebook origin). If you'd benefit
from *any* structure, durability, or HA, switch to Redis.

**Q: How does each handle running out of memory?**
Redis: you set `maxmemory` and a `maxmemory-policy` — `noeviction` (reject
writes), `allkeys-lru`, `allkeys-lfu`, `volatile-lru`, `volatile-ttl`,
`allkeys-random`, etc. You choose the behavior. Memcached: automatic LRU
eviction within slab classes once memory is full; you don't pick a policy, it
just evicts the least-recently-used item in the appropriate slab. Redis's
`noeviction` option matters when you're using it as a durable store, not a cache
— you don't want silent eviction of real data.

**Q: What's a slab allocator and what's "slab calcification"?**
Memcached pre-divides memory into slabs of fixed chunk sizes (slab classes,
e.g. 96 B, 120 B, ...). An item goes into the smallest class that fits it. This
eliminates external fragmentation and makes allocation O(1), but wastes the
slack between item size and chunk size (internal fragmentation). "Calcification"
is when memory got assigned to slab classes for an old size distribution and
your item sizes have since shifted — memory is stuck in the wrong classes and
can't be easily reassigned, so you get evictions in one class while another has
free chunks. Restart or automatic slab rebalancing addresses it.

**Q: How do you scale each horizontally?**
Memcached has no built-in clustering: the *client* shards across a static list
of nodes, almost always with **consistent hashing** so adding/removing a node
only remaps a fraction of keys. Redis has **Redis Cluster** — 16,384 hash slots
distributed across primaries, with automatic resharding and failover built in,
plus replicas per shard. So Memcached pushes sharding to the client; Redis
builds it into the server.

**Q: Persistence — what are RDB and AOF, and which would you use?**
RDB is a periodic point-in-time snapshot of the dataset — compact, fast to load,
but you can lose everything since the last snapshot on a crash. AOF logs every
write; with `appendfsync everysec` you lose at most ~1 second on a crash, at the
cost of larger files (mitigated by rewrite/compaction). Common production choice
is **both**: RDB for fast restarts and backups, AOF for point-of-crash
durability. Memcached has neither — it's ephemeral.

**Q: Both offer atomic increment — so why isn't Memcached enough for a counter?**
For a single flat counter it's fine (`incr`/`decr` are atomic). It falls short
when the counter is part of a structure (a hash field, a member's score in a
sorted set), needs conditional/multi-key atomicity (Lua/`MULTI`), or you want it
to persist. Anything beyond "one atomic number keyed by a string" pushes you to
Redis.

**Q: Value size limits?**
Memcached defaults to a 1 MB item cap (raisable with `-I`, but large items fight
the slab model). Redis strings go to 512 MB and its data structures can hold
very large collections. If you're caching big blobs, Redis is more forgiving,
though caching multi-MB values anywhere is usually a smell.

**Q: When would you run BOTH in the same architecture?**
Occasionally: Memcached as a huge, cheap, disposable blob cache for the
highest-volume simple lookups, and Redis for everything that needs
structures/durability/atomic ops (sessions, leaderboards, rate limiters, queues,
locks). More often teams consolidate on Redis for operational simplicity —
running two cache technologies is only worth it if the Memcached tier's
scale/cost advantage is large and measured.

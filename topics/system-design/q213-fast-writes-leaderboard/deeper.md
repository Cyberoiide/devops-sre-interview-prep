# Q213 — Interviewer follow-ups

**Q: Why a sorted set and not just an indexed SQL column?**
Because the expensive query is *rank*, and SQL computes it as `COUNT(*) WHERE
score > X` — an `O(N)` scan of everyone above the player, run for every rank
lookup. A sorted set (skip list + hash) maintains order at write time, so rank
is `O(log N)`. SQL can serve top-N acceptably with an index, but per-player rank
at high query volume is where it falls apart.

**Q: How is a Redis sorted set implemented, and why does that matter?**
A skip list keyed by score (ordered traversal, `O(log N)` insert that keeps
ordering) plus a hash table mapping member → score (`O(1)` score lookup). The
skip list is what lets rank and range queries avoid scanning the whole set. For
very small sets Redis uses a compact listpack encoding, but at leaderboard scale
you're always on the skip-list encoding.

**Q: Two players get the same score. What's the order, and how do you make ties
deterministic in a meaningful way?**
By default equal scores tie-break lexicographically by member name — stable but
arbitrary. To make ties meaningful (earlier-achiever wins), fold the tiebreaker
into the score: e.g. `points * 1e13 + (MAX_TS - reached_at)`. Mind that Redis
scores are 64-bit doubles (~15–17 significant digits), so keep the composite
within precision or you'll get silent rounding.

**Q: Redis is in-memory — what about durability?**
Enable AOF (`appendfsync everysec` is the usual sweet spot) and/or RDB
snapshots. For a leaderboard, losing a couple of seconds of the newest updates
on a crash is typically fine. Better still, if the board is derived from an
authoritative durable source (event log or primary DB), treat Redis as a
rebuildable cache and replay to reconstruct after a total loss.

**Q: How do you make it highly available?**
Primary + replica(s) with Sentinel for automatic failover, or Redis Cluster.
Serve heavy reads (`ZREVRANGE`, `ZREVRANK`) from replicas to offload the
primary; accept slight replication lag on reads. Writes go to the primary.

**Q: `ZADD` vs `ZINCRBY` for a score update — which and why?**
`ZINCRBY` for "player earned N points" because it's atomic: concurrent
increments both count. `ZADD` sets an absolute value, which is right for "score
is now X" but wrong for deltas — a `ZSCORE`+`ZADD` read-modify-write would drop
updates under concurrency.

**Q: The global board no longer fits on one node. Now what?**
Global exact rank is inherently a cross-shard aggregate, which is the hard case.
Practical options: (1) shard by score range — a player's global rank = sum of
counts in higher-score shards + local rank within their shard; (2) keep exact
ranks only for the top-K that people actually scroll, and give approximate rank
("top 3%") for the long tail, which nobody inspects precisely. Most large
products do some version of (2). If your boards are naturally partitioned
(per-region, per-guild, per-mode), shard on that and each board stays on one
node — far simpler.

**Q: How do you do daily / weekly / all-time boards together?**
Key-per-window with TTL for daily (`lb:daily:<date>`, expire after N days),
plus an all-time key with no TTL. Compute weekly/monthly on demand with
`ZUNIONSTORE` over the relevant daily keys — it sums each member's scores across
the windows, which is exactly rolling-total semantics.

**Q: A single hot leaderboard key concentrates all traffic on one shard/node —
is that a problem?**
It can be. A ZSET lives on one shard (its key hashes to one slot), so a very hot
global board is a single-node bottleneck and Redis Cluster can't split one key.
Mitigate by serving reads from replicas, caching the top-N page (it changes less
often than you'd think and is read far more than it's written), and rate-limiting
or coalescing rank lookups. If writes alone exceed one node, you must partition
the board itself (see the sharding answer).

**Q: How do you show "players around me" (e.g. ranks 4,210–4,220)?**
Get the player's rank with `ZREVRANK`, then `ZREVRANGE key rank-5 rank+5
WITHSCORES` for the surrounding window — both `O(log N + window)`. This is the
standard "your neighborhood" view and stays cheap because the window is small.

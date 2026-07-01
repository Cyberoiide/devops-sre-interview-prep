# Q213 — Solution and walkthrough

## The one-paragraph answer

Use a **Redis Sorted Set (ZSET)**. It stores `(member, score)` pairs and keeps
them ordered by score *at all times*, so the expensive part of a leaderboard —
ranking — is maintained incrementally at write time instead of computed at read
time. `ZADD`/`ZINCRBY` update a score in `O(log N)`, `ZREVRANK` returns a
player's rank in `O(log N)`, and `ZREVRANGE 0 N` returns the top N in
`O(log N + N)`. This is in-memory, single-node capable of hundreds of thousands
of ops/sec, and it turns the two hard queries ("my rank" and "top N") into cheap
reads. It beats the SQL approach because `SELECT COUNT(*) WHERE score > X` is an
`O(N)` scan per rank lookup, which collapses under a real leaderboard's load.

---

## Why the sorted set is the right structure

A Redis sorted set is implemented as a **skip list** (for ordered traversal by
score/rank) plus a **hash table** (member → score, for `O(1)` score lookup).
The skip list is what gives you `O(log N)` insertion that *keeps the set
ordered*. This is the crux: the ordering work is amortized across every write,
so no read ever has to sort or scan the full population.

Contrast with SQL `players(id, score)`:

| Query          | SQL                                    | Cost           | ZSET        | Cost         |
|----------------|----------------------------------------|----------------|-------------|--------------|
| Update score   | `UPDATE ... SET score`                 | index maint.   | `ZADD`      | `O(log N)`   |
| Player's rank  | `COUNT(*) WHERE score > X`             | **`O(N)` scan**| `ZREVRANK`  | `O(log N)`   |
| Top N          | `ORDER BY score DESC LIMIT N`          | index scan     | `ZREVRANGE` | `O(log N+N)` |
| Player's score | `SELECT score WHERE id = X`            | `O(1)` (PK)    | `ZSCORE`    | `O(1)`       |

The killer is the rank query. In SQL, computing "how many players are above me"
means counting everyone above you — linear in the population, run for every
lookup. In a sorted set the rank is a property the skip list already tracks.

---

## The core operations

```
ZADD    game:lb <score> <member>     # set/replace a player's score          O(log N)
ZINCRBY game:lb <delta> <member>     # atomically add points                 O(log N)
ZREVRANK game:lb <member>            # rank, 0-based, top score = rank 0      O(log N)
ZREVRANGE game:lb 0 99 WITHSCORES    # top 100 with scores                    O(log N + 100)
ZSCORE  game:lb <member>             # current score                         O(1)
ZCARD   game:lb                      # total players                         O(1)
ZCOUNT  game:lb <min> <max>          # players in a score band               O(log N)
```

Use **`ZINCRBY` for "player scored N points"** — it's atomic, so two concurrent
increments both land. A `ZSCORE` + `ZADD` read-modify-write would lose updates
under concurrency.

Ranks are **0-based**; add 1 for human display ("#1"). Use the `REV` variants
because leaderboards want highest-score-first; the non-`REV` versions order
ascending.

---

## Complexity summary

- Writes: `O(log N)`. At a million members, `log2(1e6) ≈ 20` — trivial. A
  single Redis node handles ~100k+ writes/sec; pipelining pushes bulk loads far
  higher (you saw this in the lab).
- Rank: `O(log N)`. Instant regardless of population.
- Top N: `O(log N + N)` where N is the page size (e.g. 100), *not* the
  population. This is why "top 100 of 8 million" is cheap.

---

## Scaling and production concerns

### Persistence and durability

Redis is in-memory but not necessarily ephemeral here. Enable **AOF**
(append-only file, ideally `appendfsync everysec`) and/or **RDB** snapshots so
the board survives a restart. For a leaderboard, losing a few seconds of the
most recent score updates on a crash is usually acceptable, so `everysec` AOF is
a good balance. If the leaderboard is *derived* from an authoritative store
(e.g. a durable event log or the game's primary DB), you can treat Redis as a
rebuildable cache and replay to reconstruct it.

### Replication and availability

Run a **primary + replica(s)** with Redis Sentinel (or Redis Cluster) for
failover. Reads (`ZREVRANGE`, `ZREVRANK`) can be served from replicas to spread
read load; writes go to the primary. Note replica reads can be slightly stale.

### Memory

Each member costs roughly the size of its name plus skip-list/hash overhead
(tens of bytes). Millions of members fit comfortably in a few hundred MB to a
few GB. Keep member identifiers short (numeric IDs, not long usernames — map
names elsewhere). Redis has a small-zset listpack optimization for tiny sets,
irrelevant at this scale.

### Sharding

One board on one node is fine for millions of members. You shard when either
(a) memory exceeds one node, or (b) write throughput exceeds one node. Options:

- **Global leaderboard, too big for one node**: this is genuinely hard because
  global rank is inherently a cross-shard aggregate. Approaches: bucket by
  score range across nodes (rank = sum of counts in higher buckets +
  within-bucket rank), or maintain approximate ranks and only compute exact
  rank for the top slice people actually look at. Many products only guarantee
  exact ranks for the top-K and give approximate rank ("top 5%") below that.
- **Naturally partitioned boards** (per-region, per-guild, per-game-mode):
  shard by that key — each board lives entirely on one node. Much simpler and
  the common real-world case.

### Time-windowed leaderboards

Key-per-window with a TTL: `lb:daily:2026-07-01`, `EXPIRE` it after the window.
Build weekly/monthly on demand with `ZUNIONSTORE` over the daily keys (it sums
scores). Old boards clean themselves up via TTL.

---

## Ties

By default, members with equal scores are ordered **lexicographically by
member name** — deterministic but arbitrary. If ties need a real tiebreaker
(e.g. "who reached the score first"), encode it into the score. A common trick:
make the score a composite float `score.timestampComplement`, or use a large
integer score `points * 1e13 + (MAX_TS - reached_at)` so higher points win and,
within equal points, earlier achievers rank higher. Just watch float precision —
Redis scores are IEEE-754 doubles with ~15–17 significant digits, so keep the
composite within that budget or use integer scoring carefully.

---

## End-to-end architecture

```
game servers --(score events)--> write path: ZINCRBY game:lb <delta> <player>
                                            + ZINCRBY lb:daily:<date> <delta> <player>
players open leaderboard -------> read path: ZREVRANGE game:lb 0 99 (top 100)
                                            + ZREVRANK game:lb <player> (my rank)
                                            + ZSCORE   game:lb <player> (my score)
durability: AOF everysec (+ optional rebuild from authoritative store)
availability: primary + replica via Sentinel; reads may hit replicas
windows: key-per-day + TTL; weekly via ZUNIONSTORE
```

The takeaway: the leaderboard's difficulty is *ranking under high write rate*,
and the sorted set solves exactly that by maintaining order incrementally so
both "top N" and "my rank" are `O(log N)` reads.

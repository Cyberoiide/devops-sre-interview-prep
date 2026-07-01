# Q213 — Hands-on lab: build a leaderboard on Redis Sorted Sets

You will build a working leaderboard, load ~1,000,000 players into it, and
query top-N and individual ranks — observing that both stay fast even at a
million members.

---

## 0. Prerequisites

- Docker.
- Python 3.9+ (`pip install redis==5.0.8`).

---

## 1. Start Redis

```bash
docker run -d --name lb-redis -p 6379:6379 redis:7-alpine
docker exec -it lb-redis redis-cli PING   # -> PONG
```

---

## 2. Learn the four commands by hand

Open a CLI and try the core sorted-set operations:

```bash
docker exec -it lb-redis redis-cli
```

```redis
# ZADD key score member  -> add/update a player's score. O(log N).
ZADD game:leaderboard 1500 alice
ZADD game:leaderboard 2300 bob
ZADD game:leaderboard 1800 carol
ZADD game:leaderboard 2300 dave      # tie with bob

# ZREVRANGE key start stop [WITHSCORES]  -> top N, highest first. O(log N + M).
ZREVRANGE game:leaderboard 0 2 WITHSCORES     # top 3

# ZREVRANK key member  -> a player's rank, 0-based, highest score = rank 0.
ZREVRANK game:leaderboard alice               # alice's rank

# ZSCORE key member  -> a player's current score.
ZSCORE game:leaderboard bob

# ZINCRBY key delta member  -> atomically add points (great for "player scored").
ZINCRBY game:leaderboard 50 alice             # alice gains 50 points

# ZCARD -> total players. ZCOUNT -> how many in a score range.
ZCARD game:leaderboard
```

Note the four operations that matter:

| Need                        | Command      | Complexity        |
|-----------------------------|--------------|-------------------|
| Update a score              | `ZADD`/`ZINCRBY` | `O(log N)`    |
| A player's rank             | `ZREVRANK`   | `O(log N)`        |
| Top N players               | `ZREVRANGE`  | `O(log N + M)`    |
| A player's score            | `ZSCORE`     | `O(1)`            |

`M` is the number of results returned (e.g. 100 for top-100), not the total
size — that's why top-N is cheap even with millions of members.

Type `exit` to leave the CLI.

---

## 3. Load 1,000,000 players fast (pipelining)

Doing a million round-trips one at a time is slow because of network latency,
not Redis. Batch them with a pipeline. Create `load.py`:

```python
# load.py
import random
import time
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)
KEY = "game:leaderboard"
TOTAL = 1_000_000
BATCH = 10_000

r.delete(KEY)
start = time.time()
pipe = r.pipeline(transaction=False)   # no MULTI/EXEC needed, just batching
for i in range(TOTAL):
    # score in [0, 1_000_000); member = player id
    pipe.zadd(KEY, {f"player:{i}": random.randint(0, 1_000_000)})
    if (i + 1) % BATCH == 0:
        pipe.execute()
        pipe = r.pipeline(transaction=False)
pipe.execute()
elapsed = time.time() - start
print(f"loaded {r.zcard(KEY):,} players in {elapsed:.1f}s "
      f"({TOTAL/elapsed:,.0f} writes/s)")
```

```bash
python load.py
```

You should load a million entries in a few seconds and see a high writes/sec
number — pipelining is the trick that turns "slow" into "fast" for bulk loads.

---

## 4. Query ranks and top-N at scale

Create `query.py`:

```python
# query.py
import time
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)
KEY = "game:leaderboard"

def timed(label, fn):
    t = time.perf_counter()
    out = fn()
    ms = (time.perf_counter() - t) * 1000
    print(f"{label:<32} {ms:7.3f} ms")
    return out

print(f"total players: {r.zcard(KEY):,}\n")

# Top 10 (highest scores first)
top = timed("ZREVRANGE top 10", lambda: r.zrevrange(KEY, 0, 9, withscores=True))
for rank, (member, score) in enumerate(top):
    print(f"  #{rank+1:>2}  {member:<14} {int(score):>8}")

print()

# One player's rank + score (0-based rank -> +1 for human display)
p = "player:500000"
rank = timed(f"ZREVRANK {p}", lambda: r.zrevrank(KEY, p))
score = timed(f"ZSCORE {p}",  lambda: r.zscore(KEY, p))
print(f"  {p}: rank #{rank+1:,} of {r.zcard(KEY):,}, score {int(score)}")

# A "page" around a player (like showing rank 4210-4220)
around = timed("ZREVRANGE window", lambda: r.zrevrange(KEY, 4210, 4220,
                                                        withscores=True))
```

```bash
python query.py
```

You should see every query complete in well under a millisecond of Redis time
even with a million members — because sorted sets keep the ordering maintained
at write time, so reads never scan the whole set.

---

## 5. Model a live match: many updates per second

Create `updates.py` to simulate score updates flooding in, using `ZINCRBY`
(atomic add — no read-modify-write race):

```python
# updates.py
import random
import time
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)
KEY = "game:leaderboard"

N = 100_000
start = time.time()
pipe = r.pipeline(transaction=False)
for i in range(N):
    player = f"player:{random.randint(0, 999_999)}"
    pipe.zincrby(KEY, random.randint(1, 100), player)  # atomic increment
    if (i + 1) % 5_000 == 0:
        pipe.execute()
        pipe = r.pipeline(transaction=False)
pipe.execute()
elapsed = time.time() - start
print(f"{N:,} score updates in {elapsed:.2f}s ({N/elapsed:,.0f} updates/s)")
```

```bash
python updates.py
```

`ZINCRBY` is the right primitive for "player just scored N points" because it's
atomic: two concurrent increments both count, with no lost-update race that a
`GET`-then-`SET` would suffer.

---

## 6. Time-windowed leaderboard (daily / weekly)

Real games want "today's top players", not all-time. Use a **key per window**
and let it expire:

```bash
docker exec -it lb-redis redis-cli
```

```redis
# key includes the date; set a TTL so old boards clean themselves up
ZINCRBY lb:daily:2026-07-01 50 alice
EXPIRE  lb:daily:2026-07-01 172800          # keep 2 days, then auto-delete

# A weekly board = union of 7 daily boards, computed on demand:
ZUNIONSTORE lb:weekly:2026-W27 7 \
  lb:daily:2026-06-29 lb:daily:2026-06-30 lb:daily:2026-07-01 \
  lb:daily:2026-07-02 lb:daily:2026-07-03 lb:daily:2026-07-04 lb:daily:2026-07-05
ZREVRANGE lb:weekly:2026-W27 0 9 WITHSCORES
```

`ZUNIONSTORE` sums scores across the member's presence in each daily set —
exactly the semantics you want for a rolling weekly total.

---

## 7. Cleanup

```bash
docker rm -f lb-redis
```

---

## What you proved

- Writes (`ZADD`/`ZINCRBY`) are `O(log N)` and pipeline to hundreds of
  thousands per second on a single node.
- Rank (`ZREVRANK`) and top-N (`ZREVRANGE`) are cheap reads because order is
  maintained at write time — no scan-the-world like SQL `COUNT(*) WHERE score >
  X`.
- Time-windowed boards fall out naturally from key-per-window + TTL +
  `ZUNIONSTORE`.

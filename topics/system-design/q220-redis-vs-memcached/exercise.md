# Q220 — Hands-on lab: run Redis and Memcached side by side

You will run both stores in Docker, feel the protocol/data-model difference
directly, and benchmark them — including a concurrent load that shows Memcached
spreading work across multiple cores while classic Redis executes commands on
one.

---

## 0. Prerequisites

- Docker.
- Python 3.9+ (`pip install redis==5.0.8 pymemcache==4.0.0`).
- Optional: `redis-benchmark` (ships in the redis image) and `memtier_benchmark`.

---

## 1. Start both

```bash
docker run -d --name cmp-redis     -p 6379:6379 redis:7-alpine
# give memcached 4 threads and 256 MB so we can show multi-core scaling
docker run -d --name cmp-memcached -p 11211:11211 memcached:1.6-alpine \
  memcached -m 256 -t 4

docker exec -it cmp-redis redis-cli PING          # -> PONG
printf 'stats\r\nquit\r\n' | nc localhost 11211 | head -5   # memcached stats
```

---

## 2. Feel the data-model difference

Memcached only does opaque string key→value. Redis has real data structures.
This is the difference you *can't* benchmark away.

Redis — rich types:

```bash
docker exec -it cmp-redis redis-cli
```

```redis
LPUSH mylist a b c            # a list
LRANGE mylist 0 -1
ZADD board 100 alice 200 bob  # a sorted set (see the leaderboard question!)
ZREVRANGE board 0 -1 WITHSCORES
HSET user:1 name alice age 30 # a hash
HGETALL user:1
INCR counter                  # atomic server-side increment
exit
```

Memcached — everything is a string; the "structure" lives in your app:

```python
# mc_datamodel.py
from pymemcache.client.base import Client
import json

mc = Client(("localhost", 11211))

# No lists/sets/hashes. To store structured data you serialize it yourself:
mc.set("user:1", json.dumps({"name": "alice", "age": 30}))
print("user:1 ->", json.loads(mc.get("user:1")))

# Want to append to a "list"? You must read-modify-write the whole blob:
mc.set("mylist", json.dumps(["a", "b"]))
lst = json.loads(mc.get("mylist"))
lst.append("c")                     # in your app, not the server
mc.set("mylist", json.dumps(lst))   # round trip + full rewrite
print("mylist ->", json.loads(mc.get("mylist")))

# Memcached DOES have atomic incr/decr on numeric strings, and CAS:
mc.set("counter", "0")
mc.incr("counter", 1)
print("counter ->", mc.get("counter"))
```

```bash
python mc_datamodel.py
```

**Takeaway:** with Memcached, any structure (a leaderboard, a list, a set
membership test) means serialize-the-whole-blob + read-modify-write, which is
more round trips, more bandwidth, and a lost-update race unless you use CAS.
Redis does these operations *server-side and atomically*. If your workload needs
structures, that alone decides it: Redis.

---

## 3. Persistence difference (the availability point)

Redis can persist and survive a restart. Memcached cannot — restart = empty.

```bash
# Redis with AOF on: write, restart, data survives
docker exec -it cmp-redis redis-cli CONFIG SET appendonly yes
docker exec -it cmp-redis redis-cli SET survive me
docker restart cmp-redis
sleep 1
docker exec -it cmp-redis redis-cli GET survive      # -> "me"  (survived)

# Memcached: write, restart, data is gone (by design)
python - <<'PY'
from pymemcache.client.base import Client
Client(("localhost", 11211)).set("survive", "me")
PY
docker restart cmp-memcached
sleep 1
python - <<'PY'
from pymemcache.client.base import Client
print("memcached after restart:", Client(("localhost", 11211)).get("survive"))  # -> None
PY
```

**Takeaway:** Memcached treats data loss on restart as normal — it's a pure
cache; you re-warm from the source of truth. Redis lets you *choose* durability
(RDB snapshots and/or AOF), which is why it's often held to a higher
availability bar and used as more than a cache.

---

## 4. Benchmark: single-value GET/SET throughput

Create `bench_compare.py`:

```python
# bench_compare.py
import time
import redis
from pymemcache.client.base import Client

r = redis.Redis(host="localhost", port=6379)
mc = Client(("localhost", 11211))

VAL = b"x" * 100
N = 50_000

def bench(label, setfn, getfn):
    t = time.perf_counter()
    for i in range(N):
        setfn(f"k{i}", VAL)
    set_s = time.perf_counter() - t
    t = time.perf_counter()
    for i in range(N):
        getfn(f"k{i}")
    get_s = time.perf_counter() - t
    print(f"{label:<12} SET {N/set_s:9,.0f} ops/s   GET {N/get_s:9,.0f} ops/s")

bench("redis", lambda k, v: r.set(k, v), lambda k: r.get(k))
bench("memcached", lambda k, v: mc.set(k, v), lambda k: mc.get(k))
```

```bash
python bench_compare.py
```

Single-client, single-connection numbers will be *similar* — both are fast, and
you're mostly measuring the round trip (recall the latency question). The
interesting difference shows up under **concurrency**, next.

---

## 5. Benchmark under concurrency (the threading difference)

Memcached is multi-threaded (we gave it `-t 4`), so many concurrent clients can
be served on multiple cores in parallel. Classic Redis executes commands on a
single thread, so command work serializes (very fast, but one core). Push both
with many parallel connections and compare.

Use the bundled `redis-benchmark` and `memtier_benchmark` (or the Python
threaded loader below).

```bash
# Redis: 50 parallel clients, pipeline 1, GET/SET
docker exec -it cmp-redis redis-benchmark -h 127.0.0.1 -p 6379 \
  -c 50 -n 200000 -t set,get -q

# Memcached: use memtier (run it in a throwaway container on the host network)
docker run --rm --network host redislabs/memtier_benchmark:latest \
  memtier_benchmark -s 127.0.0.1 -p 11211 -P memcache_text \
  -c 50 -t 4 -n 20000 --ratio=1:1 --hide-histogram
```

Or a self-contained Python version, `bench_concurrent.py`:

```python
# bench_concurrent.py
import threading
import time
import redis
from pymemcache.client.base import Client

VAL = b"x" * 100
PER_THREAD = 20_000
THREADS = 8

def run_redis():
    r = redis.Redis(host="localhost", port=6379)   # own connection per thread
    for i in range(PER_THREAD):
        r.set(f"rk{threading.get_ident()}:{i}", VAL)
        r.get(f"rk{threading.get_ident()}:{i}")

def run_mc():
    mc = Client(("localhost", 11211))
    for i in range(PER_THREAD):
        mc.set(f"mk{threading.get_ident()}:{i}", VAL)
        mc.get(f"mk{threading.get_ident()}:{i}")

def drive(label, target):
    ts = [threading.Thread(target=target) for _ in range(THREADS)]
    t = time.perf_counter()
    for x in ts:
        x.start()
    for x in ts:
        x.join()
    ops = THREADS * PER_THREAD * 2
    print(f"{label:<12} {ops/(time.perf_counter()-t):,.0f} ops/s "
          f"({THREADS} threads)")

drive("redis", run_redis)
drive("memcached", run_mc)
```

```bash
python bench_concurrent.py
```

**What to observe:** on a multi-core box under high concurrency with tiny,
simple values, Memcached can post higher aggregate throughput because it uses
multiple cores for command processing, while a single Redis instance drives
command execution on one core. (Note: Python's GIL limits the *client* side, so
prefer `redis-benchmark`/`memtier` for a clean server-side comparison; the
Python version still illustrates the setup.) The gap is workload-specific and
often smaller than folklore suggests — but the *architectural* reason is real
and worth stating.

---

## 6. Eviction behavior (both are caches — know how they drop data)

```bash
# Redis: set a memory cap and an LRU-style eviction policy
docker exec -it cmp-redis redis-cli CONFIG SET maxmemory 10mb
docker exec -it cmp-redis redis-cli CONFIG SET maxmemory-policy allkeys-lru
# now writing >10mb of keys evicts least-recently-used ones

# Memcached: uses a slab allocator + LRU per slab class automatically.
printf 'stats\r\nquit\r\n' | nc localhost 11211 | grep -E 'evictions|curr_items'
```

**Takeaway:** Redis gives you *policies* (`noeviction`, `allkeys-lru`,
`allkeys-lfu`, `volatile-ttl`, etc.). Memcached auto-evicts LRU within slab
classes; its slab allocator avoids fragmentation but can waste memory if item
sizes don't fit slab classes well ("slab calcification").

---

## 7. Cleanup

```bash
docker rm -f cmp-redis cmp-memcached
```

---

## What you proved

- **Data model**: Redis does structures server-side and atomically; Memcached
  makes you serialize blobs and read-modify-write — decisive if you need
  structures.
- **Persistence/availability**: Redis survives restart (RDB/AOF); Memcached is
  ephemeral by design.
- **Threading**: Memcached is multi-threaded and can win aggregate QPS on a
  multi-core box for simple KV; classic Redis executes commands single-threaded.
- **Eviction**: Redis gives explicit policies; Memcached does slab-based LRU
  automatically.

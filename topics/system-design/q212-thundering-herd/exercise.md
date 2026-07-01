# Q212 — Hands-on lab: reproduce and cure a thundering herd

You will:

1. Stand up Redis + a deliberately slow, fragile "backend" in Docker.
2. Fire a burst of concurrent clients that all miss the cache at once and watch
   the backend get hammered (the stampede).
3. Fix it two ways and watch the spike flatten:
   - **Backoff + jitter** on retries.
   - **Singleflight / request collapsing** so N identical concurrent reads
     become **one** backend call.

Everything runs locally with Docker and a couple of small Python scripts. No
cloud account needed.

---

## 0. Prerequisites

- Docker (or Podman with a `docker` alias).
- Python 3.9+ with `pip`.

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install redis==5.0.8
```

---

## 1. Start Redis

```bash
docker run -d --name herd-redis -p 6379:6379 redis:7-alpine
# sanity check
docker exec -it herd-redis redis-cli PING   # -> PONG
```

---

## 2. The shared "expensive backend"

Create `backend.py`. This models a slow origin (a DB query, an upstream API).
It counts every call and records the timestamp so we can plot the spike. It is
also fragile: if too many calls arrive within one 100 ms window it starts
failing, exactly like a real overwhelmed backend.

```python
# backend.py
import time
import threading

_lock = threading.Lock()
_calls = []              # timestamps of every backend hit
_OVERLOAD_PER_100MS = 20 # more than this in one window => backend "trips"

def expensive_query(key: str) -> str:
    """Simulate a slow origin. Records the hit; may fail under overload."""
    now = time.monotonic()
    with _lock:
        _calls.append(now)
        window = [t for t in _calls if now - t < 0.1]
        if len(window) > _OVERLOAD_PER_100MS:
            raise RuntimeError("backend overloaded (429/503)")
    time.sleep(0.15)         # the query is slow
    return f"value-for-{key}"

def stats():
    """Print a text histogram of backend hits per 100 ms bucket."""
    with _lock:
        if not _calls:
            print("no backend calls recorded")
            return
        t0 = _calls[0]
        buckets = {}
        for t in _calls:
            b = int((t - t0) * 10)   # 100 ms buckets
            buckets[b] = buckets.get(b, 0) + 1
        peak = max(buckets.values())
        print(f"total backend calls: {len(_calls)}   peak/100ms: {peak}")
        for b in range(max(buckets) + 1):
            n = buckets.get(b, 0)
            print(f"  t+{b*100:>4}ms | {'#' * n} {n}")
```

---

## 3. Reproduce the stampede (no protection)

Create `stampede.py`. It simulates a cache that just expired: 200 clients all
ask for the same hot key at the same instant, all miss, and all dog-pile the
backend. Naive retry: on failure, retry immediately with a fixed tiny sleep —
the classic mistake.

```python
# stampede.py
import sys
import threading
import backend

CLIENTS = 200
results = {"ok": 0, "fail": 0}
_rlock = threading.Lock()

def naive_client():
    # Naive retry: fixed short delay, no jitter. All clients stay in lockstep.
    for _attempt in range(5):
        try:
            backend.expensive_query("hot-key")
            with _rlock:
                results["ok"] += 1
            return
        except RuntimeError:
            import time
            time.sleep(0.1)   # fixed backoff, everyone retries together
    with _rlock:
        results["fail"] += 1

def main():
    threads = [threading.Thread(target=naive_client) for _ in range(CLIENTS)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    print(f"\n=== NAIVE (no protection) ===")
    print(f"clients ok={results['ok']} fail={results['fail']}")
    backend.stats()

if __name__ == "__main__":
    main()
```

Run it:

```bash
python stampede.py
```

**What you should see:** a huge spike in the first 100 ms bucket, `fail > 0`
because the backend tripped, and the fixed-backoff retries produce *secondary*
spikes (all clients retry at the same t+100ms, t+200ms...). The herd never
disperses — it just re-forms.

---

## 4. Fix #1 — Exponential backoff WITH JITTER

The single most important word here is **jitter**. Exponential backoff alone
(0.1s, 0.2s, 0.4s...) does *not* fix a thundering herd: every client fails at
the same moment, so they all back off by the same amount and retry in lockstep
again. Jitter breaks the synchronization by randomizing each client's delay.

Create `backoff.py`:

```python
# backoff.py
import random
import time
import threading
import backend

CLIENTS = 200
BASE = 0.1     # seconds
CAP = 2.0      # max backoff
results = {"ok": 0, "fail": 0}
_rlock = threading.Lock()

def sleep_full_jitter(attempt):
    # "Full jitter" (AWS Architecture Blog): sleep = random(0, min(cap, base*2^n))
    exp = min(CAP, BASE * (2 ** attempt))
    time.sleep(random.uniform(0, exp))

def sleep_decorrelated(prev):
    # "Decorrelated jitter": sleep = min(cap, random(base, prev*3)). Returns new prev.
    nxt = min(CAP, random.uniform(BASE, prev * 3))
    time.sleep(nxt)
    return nxt

def client():
    prev = BASE
    for attempt in range(8):
        try:
            backend.expensive_query("hot-key")
            with _rlock:
                results["ok"] += 1
            return
        except RuntimeError:
            sleep_full_jitter(attempt)          # <- try swapping for decorrelated
            # prev = sleep_decorrelated(prev)
    with _rlock:
        results["fail"] += 1

def main():
    threads = [threading.Thread(target=client) for _ in range(CLIENTS)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    print(f"\n=== BACKOFF + FULL JITTER ===")
    print(f"clients ok={results['ok']} fail={results['fail']}")
    backend.stats()

if __name__ == "__main__":
    main()
```

Run it:

```bash
python backoff.py
```

**What you should see:** the backend calls are now *spread across many 100 ms
buckets* instead of stacked in the first one. The peak/100ms drops, fewer (or
zero) failures, and the histogram looks like a low plateau instead of a spike.
Try switching to decorrelated jitter (uncomment the line) and compare — it
tends to disperse load even more aggressively.

### The three jitter strategies (know all three)

| Strategy         | Formula                                             | Notes |
|------------------|-----------------------------------------------------|-------|
| **Full jitter**  | `sleep = random(0, min(cap, base*2^n))`             | Best general choice. Maximum spread. |
| **Equal jitter** | `t = min(cap, base*2^n); sleep = t/2 + random(0, t/2)` | Keeps a minimum wait; less spread than full. |
| **Decorrelated** | `sleep = min(cap, random(base, prev*3))`            | Uses previous sleep; great throughput, good spread. |

Backoff spreads the herd out **in time**. But notice we are *still* making
~200 backend calls for the same key. That is wasteful. Fix #2 attacks the
duplication itself.

---

## 5. Fix #2 — Singleflight / request collapsing

For a **read-heavy** workload where many callers want the *same* value, we can
collapse N concurrent identical requests into **one** backend call and fan the
result out to all waiters. This is what Go's `singleflight` package does and
what a good cache layer does on a miss.

Create `singleflight.py`:

```python
# singleflight.py
import threading
import backend

CLIENTS = 200
results = {"ok": 0, "fail": 0}
_rlock = threading.Lock()

class SingleFlight:
    """Collapse concurrent calls for the same key into one execution."""
    def __init__(self):
        self._lock = threading.Lock()
        self._inflight = {}   # key -> Event
        self._values = {}     # key -> (value, error)

    def do(self, key, fn):
        with self._lock:
            if key in self._inflight:
                ev = self._inflight[key]              # someone else is fetching
                leader = False
            else:
                ev = threading.Event()
                self._inflight[key] = ev
                leader = True

        if leader:
            try:
                val, err = fn(), None
            except Exception as e:                    # noqa: BLE001
                val, err = None, e
            with self._lock:
                self._values[key] = (val, err)
                del self._inflight[key]
            ev.set()                                  # wake all waiters
        else:
            ev.wait()                                 # block until leader done

        val, err = self._values[key]
        if err:
            raise err
        return val

sf = SingleFlight()

def client():
    try:
        sf.do("hot-key", lambda: backend.expensive_query("hot-key"))
        with _rlock:
            results["ok"] += 1
    except RuntimeError:
        with _rlock:
            results["fail"] += 1

def main():
    threads = [threading.Thread(target=client) for _ in range(CLIENTS)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    print(f"\n=== SINGLEFLIGHT (request collapsing) ===")
    print(f"clients ok={results['ok']} fail={results['fail']}")
    backend.stats()

if __name__ == "__main__":
    main()
```

Run it:

```bash
python singleflight.py
```

**What you should see:** `total backend calls: 1` (or a small handful,
depending on exact timing) even though 200 clients asked. All 200 get their
answer, the backend is never overloaded, `fail=0`. This is the dramatic one —
200x load reduction for the price of one lock and one Event.

> Note: in-process singleflight only dedupes within a **single process**. Ten
> app servers each still make one call = 10 calls. To dedupe across the fleet
> you need a **distributed lock in Redis** (see the cross-process version
> below).

---

## 6. Distributed cache stampede prevention (Redis)

Real systems run many app instances. Use Redis to make sure only ONE instance
across the whole fleet recomputes a missing hot key. Create `cache_lock.py`:

```python
# cache_lock.py
import time
import uuid
import redis
import backend

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def get_with_lock(key: str, ttl: int = 30):
    """Read-through cache with a distributed lock to prevent stampede."""
    cached = r.get(key)
    if cached is not None:
        return cached

    lock_key = f"lock:{key}"
    token = str(uuid.uuid4())
    # SET NX: only one instance wins the right to recompute. EX: lock auto-expires
    # so a crashed holder can't wedge the key forever.
    got_lock = r.set(lock_key, token, nx=True, ex=10)

    if got_lock:
        try:
            value = backend.expensive_query(key)   # the ONE real call
            r.set(key, value, ex=ttl)
            return value
        finally:
            # release only if we still own it (compare-and-delete via Lua)
            r.eval(
                "if redis.call('get', KEYS[1]) == ARGV[1] then "
                "return redis.call('del', KEYS[1]) else return 0 end",
                1, lock_key, token,
            )
    else:
        # someone else is recomputing; poll briefly for the filled cache
        for _ in range(50):
            time.sleep(0.1)
            cached = r.get(key)
            if cached is not None:
                return cached
        # fallback: lock holder died; recompute ourselves rather than fail
        return backend.expensive_query(key)

if __name__ == "__main__":
    r.delete("hot-key", "lock:hot-key")
    import threading
    threads = [threading.Thread(target=lambda: get_with_lock("hot-key"))
               for _ in range(200)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    print("\n=== REDIS DISTRIBUTED LOCK (cross-process safe) ===")
    backend.stats()
```

```bash
python cache_lock.py
```

**What you should see:** one backend call; the other 199 wait for the cache to
fill. This is the pattern you actually ship to production because it works
across many app servers.

---

## 7. Bonus — probabilistic early expiration (XFetch)

Locks add latency for waiters. An elegant lock-free alternative: each reader,
*before* the key expires, rolls a dice that gets likelier as expiry
approaches, and if it "wins" it refreshes the value early — so the key is
almost never allowed to fully expire under load. This is the **XFetch**
algorithm. Sketch:

```python
import math, random, time

def should_early_recompute(delta, ttl_remaining, beta=1.0):
    # delta = how long the recompute takes (measured last time)
    # recompute early if now - delta*beta*ln(rand) >= expiry
    return ttl_remaining <= delta * beta * -math.log(random.random())
```

Store the value together with `delta` (measured recompute time) and the
absolute expiry. On each read, if `should_early_recompute(...)` is true, one
lucky reader refreshes ahead of time while everyone else still serves the
slightly-stale-but-present value. No lock, no stampede.

---

## 8. Cleanup

```bash
docker rm -f herd-redis
```

---

## What to take away from the numbers

Put the four runs side by side:

| Run                    | Total backend calls | Peak / 100 ms | Failures |
|------------------------|---------------------|---------------|----------|
| Naive (fixed retry)    | high, with echoes   | very high     | many     |
| Backoff + full jitter  | still ~N, but spread | low          | few/none |
| In-proc singleflight   | ~1                  | ~1            | 0        |
| Redis distributed lock | ~1 across fleet     | ~1            | 0        |

Backoff+jitter and request collapsing are **complementary**, not competing:
jitter protects the backend from the retry storm; collapsing/locking protects
it from duplicate work. Production systems use both, plus a circuit breaker to
stop hammering a backend that is clearly still down.

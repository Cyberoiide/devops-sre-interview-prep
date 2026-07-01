# Q214 — Hands-on lab: reproduce and fix a matchmaking race

You will:

1. Reproduce the double-match race with concurrent matchmaker workers hammering
   a shared Redis lobby.
2. Fix it with an atomic **`SETNX`** claim.
3. Harden it with an atomic **Lua** script that claims *and* records the match
   in one indivisible step.
4. Emit a metric that counts prevented double-claims (the observability signal).

---

## 0. Prerequisites

- Docker.
- Python 3.9+ (`pip install redis==5.0.8`).

---

## 1. Start Redis

```bash
docker run -d --name mm-redis -p 6379:6379 redis:7-alpine
docker exec -it mm-redis redis-cli PING   # -> PONG
```

---

## 2. Reproduce the race (the broken version)

This models the buggy check-then-act: each worker reads the player's status,
sees "waiting", and claims them — with a tiny sleep in the gap to make the race
fire reliably (in production the gap is real network/GC latency). Create
`race_broken.py`:

```python
# race_broken.py
import threading
import time
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

PLAYER = "player:P"
WORKERS = 50
double_matches = []          # matches that wrongly grabbed an already-claimed player
_lock = threading.Lock()

def reset():
    r.delete(f"{PLAYER}:status")
    r.set(f"{PLAYER}:status", "waiting")

def matchmaker(worker_id):
    # CHECK
    status = r.get(f"{PLAYER}:status")
    if status == "waiting":
        time.sleep(0.005)                     # the TOCTOU gap: real latency lives here
        # ACT
        r.set(f"{PLAYER}:status", f"matched-by-{worker_id}")
        with _lock:
            double_matches.append(worker_id)  # every worker that got here "won" P

def main():
    reset()
    threads = [threading.Thread(target=matchmaker, args=(i,))
               for i in range(WORKERS)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    print(f"=== BROKEN check-then-act ===")
    print(f"workers that claimed the SAME player P: {len(double_matches)}")
    print(f"  -> {double_matches[:10]}{' ...' if len(double_matches) > 10 else ''}")
    print(f"final status: {r.get(f'{PLAYER}:status')}")

if __name__ == "__main__":
    main()
```

```bash
python race_broken.py
```

**What you should see:** `workers that claimed the SAME player P:` is **greater
than 1** — often 5, 10, or more. Every one of those is a player being dropped
into multiple matches. The bug is reproduced.

---

## 3. Fix #1 — atomic claim with SETNX

The fix is to collapse check-and-act into one atomic operation. `SET key value
NX` ("set if not exists") sets the key **only if it doesn't already exist** and
tells you whether *you* were the one who set it. Exactly one worker can win.
Create `race_setnx.py`:

```python
# race_setnx.py
import threading
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

PLAYER = "player:P"
WORKERS = 50
winners = []
denied = 0                     # <-- the observability signal: prevented double-claims
_lock = threading.Lock()

def reset():
    r.delete(f"{PLAYER}:claim")

def matchmaker(worker_id):
    global denied
    # Atomic claim: only ONE worker's SETNX returns True. TTL guards against a
    # crashed worker wedging the claim forever.
    claimed = r.set(f"{PLAYER}:claim", worker_id, nx=True, ex=30)
    if claimed:
        with _lock:
            winners.append(worker_id)         # exactly one winner expected
    else:
        with _lock:
            denied += 1                        # metric: a race was prevented here

def main():
    reset()
    threads = [threading.Thread(target=matchmaker, args=(i,))
               for i in range(WORKERS)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    print(f"=== SETNX atomic claim ===")
    print(f"winners (should be exactly 1): {winners}")
    print(f"double-claims PREVENTED (metric): {denied}")

if __name__ == "__main__":
    main()
```

```bash
python race_setnx.py
```

**What you should see:** `winners (should be exactly 1): [<one id>]` and
`double-claims PREVENTED: 49`. The 49 denials are not errors — they're the
system *correctly* rejecting the racing workers. That count is precisely the
metric you graph in production.

---

## 4. Fix #2 — atomic Lua: claim AND record the match together

`SETNX` claims a player, but a real match needs multiple things to happen
atomically: claim *both* players, and write the match record — all or nothing.
A Lua script runs atomically inside Redis (single-threaded execution, no other
command interleaves), so you can do a multi-key check-and-act as one unit.
Create `race_lua.py`:

```python
# race_lua.py
import threading
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

# KEYS: player1 claim, player2 claim, match record
# ARGV: match_id
# Returns 1 if match formed, 0 if either player was already claimed.
CLAIM_MATCH = """
if redis.call('exists', KEYS[1]) == 1 or redis.call('exists', KEYS[2]) == 1 then
    return 0
end
redis.call('set', KEYS[1], ARGV[1], 'EX', 30)
redis.call('set', KEYS[2], ARGV[1], 'EX', 30)
redis.call('set', KEYS[3], ARGV[1] .. ':' .. KEYS[1] .. ':' .. KEYS[2])
return 1
"""

claim_match = r.register_script(CLAIM_MATCH)

WORKERS = 50
formed = []
denied = 0
_lock = threading.Lock()

def reset():
    r.delete("p:A:claim", "p:B:claim")
    for i in range(WORKERS):
        r.delete(f"match:{i}")

def matchmaker(worker_id):
    global denied
    # every worker tries to form the SAME pairing (A vs B) -> only one may win
    ok = claim_match(keys=["p:A:claim", "p:B:claim", f"match:{worker_id}"],
                     args=[f"m{worker_id}"])
    if ok == 1:
        with _lock:
            formed.append(worker_id)
    else:
        with _lock:
            denied += 1

def main():
    reset()
    threads = [threading.Thread(target=matchmaker, args=(i,))
               for i in range(WORKERS)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    print(f"=== atomic Lua claim-and-record ===")
    print(f"matches formed with players A,B (should be exactly 1): {formed}")
    print(f"double-matches PREVENTED (metric): {denied}")

if __name__ == "__main__":
    main()
```

```bash
python race_lua.py
```

**What you should see:** exactly one match formed; both players claimed and the
match record written as a single atomic step. 49 prevented. This is the
production-shaped fix: the *entire* "if both free, claim both and record"
transaction is indivisible.

---

## 5. The observability signal (what you'd wire to Grafana)

In production you don't just prevent the race — you *measure* that you're
preventing it. Emit a counter every time a claim is denied. Sketch with
`prometheus_client`:

```python
# metrics_sketch.py  (illustrative)
from prometheus_client import Counter

double_claim_prevented = Counter(
    "matchmaking_double_claim_prevented_total",
    "Times a matchmaker was denied a player already claimed (race prevented)",
    ["reason"],
)

# in the else-branch of the SETNX / Lua fix:
# double_claim_prevented.labels(reason="already_claimed").inc()
```

Then a Grafana panel: `rate(matchmaking_double_claim_prevented_total[1m])`.

Interpretation:
- **Zero under normal load, small spikes under traffic surges** = healthy. The
  guard is catching real contention and rejecting it cleanly.
- **The old broken metric** you'd want is "players found in >1 active match" —
  that must be **flat zero**. If the prevented-count is climbing but
  double-membership stays zero, your fix works. If double-membership is ever
  nonzero, the guard is not actually atomic.

Also alert on the *lock/claim TTL expiry while held* case (a worker crashed
mid-match) and on claim latency.

---

## 6. Cleanup

```bash
docker rm -f mm-redis
```

---

## What you proved

- Check-then-act on shared state races: multiple workers claim the same player.
- `SET ... NX` makes the claim atomic — exactly one winner, the rest denied.
- A Lua script extends atomicity to a *multi-key* transaction (claim both
  players + record the match, all-or-nothing).
- The count of denied claims is the production signal that the race is being
  prevented; "player in >1 match" must stay zero.

# Q214 — Solution and walkthrough

## The one-paragraph answer

The double-match bug is a **check-then-act race** (TOCTOU): two matchmaker
workers both read "player P is waiting" before either writes "P is matched", so
both claim P. The fix is to make "if available, claim" a **single atomic
operation** that only one worker can win. The ladder: an in-process **mutex**
works only for a single matchmaker instance; a **distributed lock (Redlock)**
extends mutual exclusion across many instances; but the cleaner answer is an
**atomic conditional claim** — `SET player:claim <worker> NX EX 30`, or an
atomic **Lua script** when the claim spans multiple keys (claim both players
*and* record the match, all-or-nothing). Prove it in production by emitting a
counter on every denied claim and graphing it: prevented-races spike under load
(healthy), while "player in more than one active match" must stay flat zero.

---

## Why the race happens

Any operation shaped as

```
value = read(state)      # CHECK
if value is acceptable:
    write(new_state)     # ACT
```

has a gap between CHECK and ACT. Two concurrent workers can both pass the CHECK
before either does the ACT, so both proceed. In matchmaking, both see P as
"waiting" and both grab P. The gap is filled by ordinary latency — network
round-trips, GC pauses, scheduler preemption — so it fires reliably under load
even though it "looks" instantaneous in code.

The only durable fix is to eliminate the gap: turn CHECK+ACT into **one atomic
step** the datastore executes indivisibly, so exactly one worker wins and the
losers are told "already claimed".

---

## The ladder of solutions

### Rung 1 — Application-level lock (single instance only)

A `threading.Lock` / `sync.Mutex` around the claim serializes workers *within
one process*. Correct and simple **if there is exactly one matchmaker process**.
The instant you run two matchmaker instances (for HA or throughput), the mutex
is worthless — each process has its own lock and they don't coordinate. So this
is the right answer only for a single-node matchmaker, and you should say so
explicitly.

### Rung 2 — Distributed lock (Redlock) across instances

To coordinate many matchmaker instances, use a lock that lives *outside* the
processes — in Redis. The pattern:

```
SET lock:player:P <unique-token> NX EX 30     # acquire, only one wins
... do the matching work ...
# release ONLY if we still own it (compare-and-delete via Lua):
if redis.call('get', KEYS[1]) == ARGV[1] then redis.call('del', KEYS[1]) end
```

**Redlock** is the algorithm for acquiring a lock across *multiple independent
Redis nodes* (acquire on a majority within a time bound) so a single Redis
failure doesn't drop mutual exclusion. Use it when the lock is protecting the
matching *logic* rather than a single key, or when the critical section spans
work that isn't a simple Redis write. Non-negotiable details:

- **Unique token per holder** + **compare-and-delete on release**, so you never
  delete someone else's lock.
- **TTL (`EX`)** so a crashed holder's lock auto-expires instead of wedging the
  player forever.
- Be aware of Redlock's **correctness limits** (see below and `deeper.md`).

### Rung 3 — Atomic conditional claim (usually the best answer)

Often you don't need a separate lock at all — make the *claim itself* the atomic
operation. `SET player:P:claim <worker> NX EX 30`:

- `NX` = set only if the key doesn't exist. Exactly one worker's SET succeeds.
- The return value tells that worker it won; every other worker gets "denied".
- `EX 30` = the claim self-heals if the winner crashes before finishing.

This is simpler and less error-prone than a lock: there's no separate acquire /
critical-section / release dance, and the "resource" (the player being claimed)
*is* the thing you're locking. For a claim that touches **multiple keys** (claim
player A **and** player B, then write the match record — all or nothing), wrap
it in a **Lua script**. Redis executes scripts atomically on its single command
thread, so no other command interleaves; the whole multi-key check-and-act is
indivisible. That's the `race_lua.py` version in the lab.

**Rule of thumb:** prefer an atomic claim (`SETNX`/Lua) over a general lock when
the critical section is "conditionally mutate some Redis keys". Reach for a
distributed lock (Redlock) when you must guard work that lives *outside* Redis or
spans multiple operations you can't express as one script.

---

## Correctness details you must mention

- **Atomicity**: `SET ... NX` and Lua scripts are atomic in Redis. A `GET` then
  a separate `SET` is *not* — that's the original bug. Never use
  read-modify-write across two commands for a claim.
- **TTL on every lock/claim**: without it, a worker that crashes mid-match holds
  the player forever. With it, the claim expires and the player re-enters
  matchmaking. Set the TTL comfortably longer than the worst-case matching time.
- **TTL is a trade-off**: too short and the claim expires while the winner is
  still legitimately working (now two workers can hold it → the race returns);
  too long and a crash leaves the player stuck for that duration. Tune to
  `p99(match_time) + margin`.
- **Fencing tokens**: if the protected action can be delayed (a paused worker
  wakes after its TTL expired and still tries to write "P is in match #1"), a
  TTL alone doesn't save you. Hand out a monotonically increasing **fencing
  token** with each claim and have the downstream reject writes carrying a stale
  token. This is the rigorous fix for the "expired-lock holder acts anyway"
  problem.
- **Compare-and-delete on release**: release must verify ownership (Lua
  `get==mytoken` then `del`) so a slow worker whose lock already expired doesn't
  delete the *next* holder's lock.

---

## Observability — proving the race is gone

Fixing the race silently isn't enough; you must be able to *demonstrate* it.

- **Emit a counter on every denied claim**:
  `matchmaking_double_claim_prevented_total`. Each denial is the system
  correctly rejecting a racing worker.
- **Grafana panel**: `rate(matchmaking_double_claim_prevented_total[1m])`.
  Expected shape: ~zero at rest, small spikes during traffic surges. Those
  spikes are *good* — they mean real contention is being caught and handled
  cleanly rather than corrupting state.
- **The invariant metric**: "players currently in more than one active match"
  must be **flat zero**. Compute it periodically (or assert it in a consistency
  check) and alert if it's ever nonzero — that would mean your claim isn't
  actually atomic.
- **Also watch**: distributed-lock/claim acquisition failures vs. successes,
  claim latency, and TTL-expiry-while-held events (a proxy for workers crashing
  mid-match).

The story to tell the interviewer: *prevented-races rising + double-membership
staying zero = the guard works.* If prevented-races is zero but you still get
bug reports, your "atomic" claim isn't atomic (probably a `GET`/`SET` in
disguise).

---

## Summary table

| Rung | Mechanism                     | Works across instances? | Use when |
|------|-------------------------------|-------------------------|----------|
| 1    | In-process mutex              | No                      | Single matchmaker only |
| 2    | Distributed lock / Redlock    | Yes                     | Guarding non-Redis work or multi-step logic |
| 3    | `SETNX` / atomic Lua claim    | Yes                     | Claim = conditional mutation of Redis keys (preferred) |

Always: unique token, TTL, compare-and-delete release, and fencing tokens if the
protected action can be delayed. Always: a metric proving denials happen and an
invariant proving double-membership never does.

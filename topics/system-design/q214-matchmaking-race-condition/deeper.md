# Q214 — Interviewer follow-ups

**Q: Precisely what kind of bug is this?**
A check-then-act race condition, a.k.a. TOCTOU (time-of-check to time-of-use).
Two workers both evaluate "is player P available?" as true before either commits
the claim, because the read and the write are separate operations with a gap
between them. The gap is filled by real latency (network, GC, scheduling), so it
fires under load.

**Q: Why is a `threading.Lock` / mutex not a real fix here?**
It only serializes threads inside one process. Matchmakers run as multiple
instances for HA and throughput; each process has its own mutex and they don't
coordinate, so two instances still double-claim. In-process locks are correct
only for a single-instance matchmaker.

**Q: `SETNX` vs a distributed lock (Redlock) — when do you use which?**
Use an atomic `SETNX`/Lua claim when the critical section is "conditionally
mutate some Redis keys" — the resource you're protecting *is* a Redis key, so
make the claim itself atomic; simpler and no separate release dance. Use a
distributed lock (Redlock) when you must guard work that lives outside Redis or
spans multiple steps you can't express as one script. For matchmaking, an atomic
claim is usually the cleaner answer.

**Q: Why does the Lua script give you atomicity that two commands don't?**
Redis executes commands (and each `EVAL` script) on a single thread, one at a
time, with no interleaving. So a script that reads several keys and conditionally
writes several keys runs as one indivisible unit — no other client's command can
slip into the middle. That lets you do a multi-key check-and-act (claim both
players + record the match) atomically, which `SETNX` on a single key can't.

**Q: You put a TTL on the lock. What if it expires while the worker is still
legitimately working?**
Then a second worker can acquire the "same" claim and you're back to two holders
— the race returns. And on release, the slow original could delete the second
holder's lock. Two mitigations: (1) compare-and-delete on release (only delete a
lock whose token is still yours), and (2) fencing tokens — hand out a
monotonically increasing number with each claim and have the downstream reject
any write carrying a stale token. Also size the TTL to `p99(match_time) + margin`
so legitimate work rarely outlives it.

**Q: What are fencing tokens and when are they strictly necessary?**
A fencing token is a monotonically increasing number issued with each lock grant.
The protected resource records the highest token it has seen and rejects any
operation with a lower one. They're necessary when a lock holder can be *paused*
(GC, VM freeze) past its TTL and then wake up and still attempt its write: TTL +
compare-and-delete don't stop that stale write, but a fencing check does. This is
the core of Martin Kleppmann's critique of using locks for correctness.

**Q: Is Redlock safe? I've heard it's controversial.**
Redlock provides good *efficiency* (avoid duplicate work most of the time) but is
debated for strict *correctness* under clock drift and process pauses. Kleppmann
argued that without fencing tokens no distributed lock — Redlock included —
guarantees mutual exclusion when a holder stalls past its TTL. Antirez (Redis's
author) pushed back on some assumptions. Practical stance: if a rare double-claim
is merely wasteful, Redlock/`SETNX` with TTL is fine; if a double-claim corrupts
state and must never happen, add fencing tokens or use a system with real
consensus/linearizable transactions.

**Q: Could you avoid distributed coordination entirely?**
Yes — make one worker authoritative per partition. Route all players in a given
region/mode/skill-bucket to a single matchmaker shard (consistent hashing or a
partitioned queue), so no two workers ever consider the same player concurrently.
This trades the lock for a partitioning scheme and removes the race by design.
Redis Streams consumer groups or a Kafka partition per bucket give you exactly
one consumer per partition.

**Q: How do you prove in production that the race is fixed?**
Emit `matchmaking_double_claim_prevented_total` on every denied claim and graph
`rate(...[1m])` in Grafana — expect ~zero at rest with small spikes under load
(healthy: contention caught and rejected). Separately track the invariant
"players in more than one active match", which must be flat zero and alert if
ever nonzero. Rising prevented-count + zero double-membership = the guard works.
Zero prevented-count while bugs persist = your claim isn't actually atomic.

**Q: What if the claim succeeds but the worker crashes before forming the match
— isn't the player stuck?**
The TTL handles it: the claim expires and the player re-enters matchmaking. To
re-match faster than the TTL, add a reaper that detects dead workers (missed
heartbeats) and releases their claims early, or use Redis Streams' pending-entry
list so unacked work is reassigned to another consumer.

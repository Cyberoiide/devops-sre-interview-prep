# Q216 — Solution and walkthrough

## The one-paragraph answer

Keep the *orders of magnitude* in your head, not exact figures: RAM ~100 ns,
local SSD ~10–100 µs, a same-datacenter/AZ round trip ~0.5 ms, cross-AZ within a
region ~1–2 ms, an in-memory DB (Redis) round trip ~1–5 ms, a cross-region round
trip ~50–150 ms, and a serverless cold start ~200–500 ms. The skill that matters
is **budget reasoning**: users expect ~0.5–1 s in a SaaS, your response time is
the *sum of the serial critical path*, and in distributed systems that path is
dominated by **network round trips and boundary crossings**, not CPU. So you
optimize by removing round trips (cache, batch/pipeline, colocate, parallelize
independent calls) rather than shaving compute — one avoidable cross-region hop
can cost more than thousands of RAM accesses.

---

## The numbers, grouped by scale

Think in tiers separated by ~1000x:

- **Nanoseconds — on-CPU / RAM.** L1 ~1 ns, L2 ~4 ns, branch mispredict ~3 ns,
  mutex ~17 ns, main memory ~100 ns. Anything touching only registers, cache,
  and RAM is effectively free compared to I/O.
- **Microseconds — local device / same-host.** Reading 1 MB from RAM ~3 µs,
  fast compression of 1 KB ~2 µs, NVMe SSD random 4 KB read ~10–20 µs, SSD 1 MB
  sequential ~50 µs, loopback network round trip ~5–30 µs.
- **Milliseconds — the network and spinning disks.** Same-DC/AZ round trip
  ~0.5 ms, older SATA SSD 1 MB read ~1 ms, cross-AZ same region ~1–2 ms, Redis
  round trip ~1–5 ms, HDD seek ~10 ms.
- **Tens-to-hundreds of milliseconds — long distance and cold starts.**
  Cross-region round trip ~50–150 ms (floor set by speed of light in fiber),
  serverless cold start ~200–500 ms, cross-continent ~150–300 ms.

The "multiply by 1e9" trick makes the ratios human: RAM = a minute, a same-DC
round trip = a week, a cross-region hop = years. If RAM is a minute and a
cross-region call is years, you can *feel* why you never make a chatty
cross-region protocol.

Why cross-region has a hard floor: light in fiber travels ~200,000 km/s (~5
µs/km), so a 6,000 km path (e.g. Amsterdam↔N. Virginia) is ~30 ms one-way *at
best*, ~60 ms round trip before any routing or processing. No amount of
engineering beats physics; you beat it by *not making the round trip* (edge
caching, regional replicas, async replication).

---

## Budget reasoning: the actual skill

Interviewers don't care that you memorized 100 ns for RAM. They care that you
can do this:

> Budget = user expectation − fixed overhead. Does the critical path fit?

### Worked example 1 — a SaaS API request, target 500 ms

```
TLS/HTTP ingress + LB                ~5 ms
Auth check (cache hit, Redis)        ~2 ms
Primary DB query (same AZ, indexed)  ~5 ms
Downstream service call (same region)~10 ms
Serialization + app logic            ~5 ms
Response egress                      ~5 ms
------------------------------------------
Serial total                         ~32 ms   → fits 500 ms with huge margin
```

Now break it:

```
DB query hits disk / unindexed scan  ~200 ms
Downstream call goes cross-region    ~120 ms
Cold Lambda in the path              ~300 ms
------------------------------------------
Serial total                         ~620 ms  → blows the budget
```

The fix isn't faster code — it's an index (avoid the disk scan), colocating the
downstream service in-region (avoid the cross-region hop), and keeping the
function warm or precomputing (avoid the cold start). **Removing round trips and
boundary crossings, not optimizing CPU.**

### Worked example 2 — fan-out and the tail

If a request makes 5 downstream calls:

- **Serial**: latencies add → 5 × 10 ms = 50 ms.
- **Parallel** (independent calls): latency ≈ the *slowest* one, but you're now
  exposed to **tail latency** — with 5 calls, your p99 response is closer to the
  p99 of the *slowest of 5*, which is worse than any single call's p99. This is
  why fan-out amplifies tail latency and why techniques like hedged requests,
  timeouts, and backup requests exist.

Rule: **parallelize independent calls, serialize only true dependencies, and
budget for the tail, not the median.**

---

## What dominates, and therefore what to optimize

In almost every distributed system the critical path is dominated by:

1. **Network round trips** — each one is µs (same host) to ms (same region) to
   tens-of-ms (cross region). Chatty protocols (N sequential calls) are the
   classic killer.
2. **Boundary crossings** — user space ↔ kernel (syscalls), process ↔ process,
   host ↔ host, region ↔ region. Each step up is ~10–100x.
3. **Serialization** and, at large payloads, **bandwidth** (moving 1 MB isn't
   free even in-DC).

CPU work (the actual computation) is usually a rounding error next to these,
*unless* you're doing something genuinely compute-heavy. So the high-leverage
moves are:

- **Cache** to turn a ms-scale DB/network read into a µs-scale memory read.
- **Batch / pipeline** to amortize the round trip over many operations (the lab
  shows Redis pipelining winning 10–100x per op).
- **Colocate** chatty services in the same AZ/region.
- **Parallelize** independent calls; **fail fast** with tight timeouts.
- **Precompute / async** the expensive parts off the critical path.
- **Avoid cold starts** (provisioned concurrency / warm pools) when they'd land
  on the user's path.

---

## What NOT to do

- Don't quote numbers to false precision — say "~100 ns", "~1 ms", "~100 ms".
  Exact values depend on hardware generation and year; the ratios are stable.
- Don't optimize CPU when the bottleneck is a round trip.
- Don't design a synchronous chain of cross-region calls and hope.
- Don't forget the tail: median budgets that "fit" can still miss at p99, which
  is what users actually feel.

The headline: **know the tiers, reason with budgets, and remember that in
distributed systems latency is mostly about round trips and boundaries, not
compute.**

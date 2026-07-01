# Q216 — Interviewer follow-ups

**Q: Why should I trust a table of latency numbers that's clearly hardware- and
year-dependent?**
You shouldn't trust the exact digits — you should internalize the *ratios* and
*tiers*, which are stable across generations. RAM has been ~100x slower than L1
and ~1000x faster than a network round trip for a long time, and the speed of
light isn't changing. Reason in orders of magnitude ("ns / µs / ms / tens-of-ms")
and you'll be right about what dominates even as absolute numbers drift.

**Q: What's the single most important latency to respect and why?**
The cross-region round trip (~50–150 ms), because it's bounded by physics — light
in fiber is ~5 µs/km, so a few thousand km is tens of ms one-way no matter what
you build. You can't optimize it away; you can only avoid making the trip (edge
caches, regional replicas, async replication). It's also the one people
accidentally put in the synchronous path and then wonder why p99 is terrible.

**Q: RAM is ~100 ns and an SSD read is ~10–100 µs — so why do we bother with
SSDs at all in the hot path?**
We mostly don't, for the hot path — that's exactly why caches and in-memory
stores exist. SSDs give durability and capacity at a cost that's still ~1000x
faster than a spinning disk seek. The design move is to serve hot data from RAM
(cache) and fall back to SSD for the cold tail, accepting the ~100x penalty only
when you must.

**Q: You measured an "SSD" read at a few µs — that's faster than the table says.
What happened?**
The OS page cache served the repeat reads from RAM, so you measured
cache-hit + syscall cost, not the device. True uncached SSD is ~10–100 µs. To
measure the device you'd drop caches (`echo 3 > /proc/sys/vm/drop_caches`) or use
`O_DIRECT`. The "surprise" is itself the lesson: caching is why real systems feel
fast, and benchmarks lie if you don't control the cache.

**Q: Pipelining made Redis ~50x faster per op. Why, and when can't you do it?**
Because a single `GET` is dominated by the network round trip, not Redis's work;
pipelining sends many commands before reading replies, amortizing one round trip
over hundreds of ops. You can't pipeline when each request depends on the
previous response (you need the result to decide the next call), or across
independent user requests that arrive one at a time. It shines for batch work and
independent bulk operations.

**Q: You have 5 independent downstream calls. Serial or parallel, and what's the
catch?**
Parallel — total latency becomes the max instead of the sum. The catch is **tail
amplification**: your response waits for the slowest of the 5, so your p99 is
governed by "the p99 of the worst of 5 calls", which is worse than any single
call's p99. Mitigate with tight timeouts, hedged/backup requests (fire a second
copy if the first is slow), and by not fanning out more than necessary.

**Q: A cold start is ~300 ms. When does that actually matter?**
Only when it lands on a *user-facing synchronous* path — the first request after
a scale-up or idle period. It's invisible for async/batch work. Fixes:
provisioned concurrency / warm pools, keeping instances warm with pings, smaller
deployment artifacts and faster runtimes, or moving latency-critical endpoints
off cold-start-prone platforms.

**Q: How do you use these numbers when someone hands you a latency SLO?**
Decompose the target into a budget across the serial critical path: subtract
fixed costs (ingress, TLS, egress), then check whether the remaining hops
(cache, DB, downstream) fit what's left, budgeting at p99 not median. If it
doesn't fit, the numbers tell you *which hop* to attack — usually a round trip to
cache away, colocate, batch, or parallelize — rather than guessing.

**Q: Is CPU ever the bottleneck?**
Yes — genuinely compute-heavy work (crypto, compression, ML inference, big
serialization, tight numerical loops) can dominate, and then you optimize the
code, use SIMD/GPU, or cache the result. But for typical CRUD/service traffic the
round trips and boundary crossings dwarf the compute, so profile before assuming
CPU is the problem.

**Q: Where does bandwidth (not latency) start to matter?**
When payloads get large. Latency is the fixed cost of a round trip; bandwidth is
the per-byte cost. Reading 1 MB sequentially from RAM is ~µs but from SSD or
across the network it's ms-scale, and a 100 MB response is bandwidth-bound
regardless of how low your round-trip latency is. For big payloads: compress,
paginate, stream, or move computation to the data instead of the data to the
computation.

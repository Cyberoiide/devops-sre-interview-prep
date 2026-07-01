# Q220 — Redis vs Memcached: what's the difference, and when do you pick Memcached?

## The scenario (as an interviewer would pose it)

> "Redis and Memcached are both in-memory key-value stores that people reach for
> as a cache. In practice most teams default to Redis. So tell me: what are the
> real, substantive differences between them? And then — because it's the more
> interesting question — when would you actually choose **Memcached** over
> Redis? Give me a concrete workload where Memcached is the *better* pick, not
> just an acceptable one."

## What the interviewer is really probing

1. Do you know the differences that *matter* in design, not trivia:
   - **Availability / durability expectation** — Redis has persistence
     (RDB/AOF) and replication and is often run as a store you expect to
     survive restarts; Memcached is pure ephemeral cache where data loss on
     restart is *accepted by design*.
   - **Data model** — Memcached is flat string key→value only; Redis has rich
     types (strings, lists, sets, sorted sets, hashes, streams, bitmaps,
     HyperLogLog, geo).
   - **Threading / concurrency model** — Memcached is **multi-threaded** and
     scales across all cores on a box; classic Redis executes commands on a
     **single thread** (Redis 6+ added threaded *I/O*, but command execution is
     still single-threaded).
2. Can you turn those into a *decision*: name the workload where Memcached wins.
3. Do you know the surrounding details: eviction policies, memory model
   (slab allocator vs. Redis allocator), clustering, and the operational
   trade-offs?

## The short version of the answer

- **Default to Redis** for almost everything — the rich data types, persistence,
  replication, pub/sub, scripting, and cluster mode make it a Swiss-army store,
  and its single command thread is rarely the bottleneck.
- **Pick Memcached when** you have a *simple, huge, pure cache* of opaque
  string/blob values, a very high concurrent read/write rate on a multi-core
  box, and you genuinely don't need any data structures, persistence, or
  replication — a big volatile look-aside cache in front of a database. Its
  multi-threaded design squeezes more raw QPS out of a single large instance for
  that specific shape, and its simplicity is a feature.

## Deliverables

- `exercise.md` — run both in Docker, exercise the protocol differences, and
  benchmark them side by side (including a multi-threaded load to show
  Memcached using multiple cores).
- `solution.md` — the full written comparison and the decision framework.
- `deeper.md` — interviewer follow-ups (eviction, threading nuance, persistence,
  clustering) with concise answers.

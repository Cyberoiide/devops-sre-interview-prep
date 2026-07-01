# Q212 — Solution and walkthrough

## The one-paragraph answer

A thundering herd happens because a shared failure **synchronizes** all your
clients: they all fail at the same instant, so any uniform reaction (retry now,
retry in exactly 1 s, all recompute the expired key) keeps them in lockstep and
they slam the recovering resource together. The cure has two independent axes.
**Spread load out in time** with exponential backoff *plus jitter* (jitter is
the essential part — backoff without jitter just reschedules the herd), backed
by circuit breakers and load shedding. **Remove duplicate load** with request
collapsing / singleflight and cache locking, so N concurrent identical reads
become one backend call. Real systems use both, because they solve different
problems: jitter defuses the retry storm, collapsing defuses the duplicate-work
storm.

---

## Why it happens: correlation is the root cause

The mental model that matters: **a retry policy is a load-generation policy.**
Under normal conditions clients arrive at random phases, so their load is
naturally smeared out. A failure destroys that randomness — it becomes a shared
clock. Every client:

- notices the failure at ~the same time,
- reacts with the same rule,
- and therefore acts again at the same time.

So the fix is fundamentally about **re-introducing randomness** (jitter) and
**not generating N times the necessary load** (collapsing). Everything below is
a variation on those two ideas.

---

## Axis 1 — Spread load out in time

### Exponential backoff, and why it is not enough alone

Backoff = wait longer after each failure: `base * 2^attempt`, capped. It caps
the *rate* a single client retries. But if 10,000 clients all fail at t=0, plain
exponential backoff has all 10,000 retry at t=base, then all at t=2·base, etc.
The herd is preserved; you have just moved it. This is the single most common
mistake, and interviewers love to catch it.

### Jitter — the essential ingredient

Jitter randomizes each client's delay so the herd disperses. Three standard
variants (from the AWS Architecture Blog, "Exponential Backoff and Jitter"):

- **Full jitter**: `sleep = random_between(0, min(cap, base·2^attempt))`.
  Best general-purpose choice; maximum dispersion. Simulations show it
  minimizes both total work and completion time under contention.
- **Equal jitter**: `temp = min(cap, base·2^attempt); sleep = temp/2 +
  random(0, temp/2)`. Guarantees a minimum wait; slightly less spread.
- **Decorrelated jitter**: `sleep = min(cap, random(base, prev·3))`. Uses the
  previous sleep as the ceiling; excellent throughput, self-adjusting.

The lab in `exercise.md` shows the histogram flattening the moment jitter is
introduced.

### Circuit breaker

Backoff still lets clients keep probing a backend that is genuinely down. A
**circuit breaker** wraps calls to a dependency and trips **open** after a
failure threshold, so subsequent calls **fail fast locally** (no network hop)
instead of piling onto the dead backend. After a cooldown it goes
**half-open** and lets a *few* trial requests through; if they succeed it
closes, if they fail it re-opens. Crucially, half-open must allow only a
*trickle* — otherwise re-closing the breaker itself becomes a thundering herd.

### Load shedding / rate limiting at the server

Defense in depth: the server should protect itself, not trust clients to
behave. Admission control (token bucket, concurrency limits, `503 Retry-After`)
lets a recovering backend accept load it can actually handle and reject the
rest cheaply. Combine with a **health-check warmup / slow-start** so a
just-recovered node isn't sent full traffic while its caches and pools are
cold.

---

## Axis 2 — Remove duplicate load

### Request collapsing / singleflight

For read-heavy traffic where many callers want the same key, only the **first**
caller should hit the backend; the rest wait and share the result. This is:

- Go's `golang.org/x/sync/singleflight`,
- what a well-built read-through cache does on a miss,
- sometimes called "request coalescing" (e.g. in Varnish/CDNs).

In the lab, 200 concurrent readers collapse into **one** backend call. Caveats:

- Only correct for **idempotent reads** of the *same* logical value. Never
  collapse writes or requests whose responses legitimately differ per caller.
- In-process singleflight dedupes within one process only. A fleet of N app
  servers still makes up to N calls — use a distributed lock for fleet-wide
  dedup.

### Cache stampede (dog-piling) and its cures

A cache stampede is the thundering herd's cache-shaped cousin: a hot key
expires, every concurrent request misses at once, and all recompute the same
expensive value simultaneously. Cures:

1. **Locking / mutex on miss** — first misser takes a lock (`SET key val NX
   EX`), recomputes, and fills the cache; others wait briefly and read the
   filled value. The lab's `cache_lock.py` shows the production-grade version,
   including a **compare-and-delete via Lua** so you only release *your* lock,
   and a **TTL on the lock** so a crashed holder can't wedge the key forever.
2. **Probabilistic early expiration (XFetch)** — readers refresh the value
   *before* it expires, with a probability that rises as expiry nears. The key
   effectively never fully expires under load, so misses never bunch up. Fully
   lock-free. Store the measured recompute time (`delta`) alongside the value.
3. **Stale-while-revalidate** — serve the slightly stale value immediately and
   refresh it in the background. Users never wait; the backend sees one refresh
   instead of a herd. (HTTP has this as a `Cache-Control` directive.)

---

## How the two axes fit together

| Problem                                   | Fix                          |
|-------------------------------------------|------------------------------|
| Clients retry in lockstep after recovery  | Backoff **+ jitter**         |
| Clients keep probing a dead backend       | Circuit breaker              |
| Backend accepts more than it can handle   | Load shedding / slow-start   |
| N identical concurrent reads on a miss    | Singleflight / request collapse |
| Hot key expiry causes a miss storm        | Cache lock / XFetch / SWR    |

A production read path typically layers: CDN/edge cache → local in-process
cache with singleflight → distributed cache (Redis) with a lock or XFetch on
miss → origin protected by a circuit breaker and admission control, with all
retries using capped exponential backoff **with full jitter**.

---

## What the lab demonstrated, in one table

| Run                    | Backend calls | Peak/100ms | Failures | Lesson |
|------------------------|---------------|------------|----------|--------|
| Naive fixed retry      | high + echoes | very high  | many     | fixed retries re-form the herd |
| Backoff + full jitter  | ~N, spread    | low        | few/none | jitter is what disperses load |
| Singleflight           | ~1            | ~1         | 0        | collapse duplicate reads |
| Redis distributed lock | ~1 (fleet)    | ~1         | 0        | dedup across many app servers |

The headline: **backoff without jitter is theater**, and **for read-heavy hot
keys, collapsing is a 100–200x win** that backoff alone can never achieve
because backoff still makes N calls.

# Q212 — The Thundering Herd

## The scenario (as an interviewer would pose it)

> "You run a read-heavy service. Your API servers depend on a backend — say a
> primary database or a cache node. One afternoon that backend goes down for
> about 30 seconds. During the outage, thousands of client requests pile up:
> some are queued inside your app servers, some are being retried by clients,
> some are waiting on connection pools.
>
> Then the backend comes back. The instant it does, **every** queued and
> retrying request slams into it at the same millisecond. The freshly-recovered
> backend — which has cold caches and empty connection pools — is instantly
> overwhelmed again and falls over. Your clients see a wave of timeouts and
> 500s that is *worse* than the original outage.
>
> This is a **thundering herd** (also called a retry storm, or when it hits a
> cache, a **cache stampede**). Walk me through what is happening, why naive
> retries make it worse, and how you would design the system so that recovery
> is smooth instead of self-destructive."

## What the interviewer is really probing

1. Do you understand that **retries are load**? Every retry policy is
   implicitly a load-generation policy.
2. Do you know that **synchronized clients are the enemy**? The failure
   correlates every client's clock, so they all act in lockstep.
3. Can you name the two families of fixes and explain *why* each works:
   - **Spreading load out in time** — exponential backoff **with jitter**,
     circuit breakers, rate limiting / load shedding.
   - **Reducing duplicate load** — request collapsing / singleflight, cache
     locking, probabilistic early expiration.
4. Can you reason about the trade-offs (latency vs. protection, correctness of
   deduplication, TTL tuning)?

## Key terms

- **Thundering herd**: many actors wake up / retry simultaneously and
  overwhelm a shared resource.
- **Retry storm**: the client-side amplification where failed requests
  multiply into many more requests.
- **Cache stampede** (a.k.a. dog-piling): a hot cache key expires and N
  concurrent requests all miss simultaneously, all recomputing the same
  expensive value against the backend at once.

## Deliverables for this question

- `exercise.md` — a runnable Docker + Redis lab that reproduces the stampede,
  then flattens it with backoff+jitter and with singleflight/request
  collapsing. You will see the spike in a graph of "requests hitting the
  backend per 100ms".
- `solution.md` — the full written answer and walkthrough.
- `deeper.md` — the follow-up questions an interviewer asks after your first
  answer, with concise, correct responses.

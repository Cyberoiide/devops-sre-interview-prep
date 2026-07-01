# Q212 — Interviewer follow-ups

**Q: Why isn't exponential backoff enough on its own?**
Because the failure synchronizes every client's clock. Plain exponential
backoff makes every client wait the *same* amount, so they all retry together
at t=base, 2·base, 4·base... The herd is rescheduled, not dispersed. Jitter
randomizes each client's delay, which is what actually breaks the
synchronization. Backoff caps a single client's rate; jitter decorrelates the
fleet.

**Q: Full vs. equal vs. decorrelated jitter — which do you pick?**
Default to **full jitter** (`random(0, min(cap, base·2^n))`): AWS's simulations
show it minimizes both total work and time-to-completion under contention.
**Equal jitter** if you want to guarantee some minimum wait between attempts.
**Decorrelated jitter** (`min(cap, random(base, prev·3))`) if you want maximum
throughput and are comfortable with occasional short retries; it self-adjusts
and is what the AWS SDKs and Redis clients often use.

**Q: Singleflight collapses concurrent reads — what breaks if you apply it to
writes?**
Correctness. Collapsing assumes all callers want the *same* result. Writes have
side effects and per-caller semantics; collapsing two "increment counter" calls
into one loses an increment. Only collapse idempotent reads of the same logical
key. Also watch error sharing: if the single flight fails, all waiters get the
same error — usually fine, but it means one transient blip fails everyone in
that batch. Some implementations "forget" the failed result immediately so the
next caller retries fresh.

**Q: Your Redis lock has a TTL. What happens if the lock holder is slow and the
TTL fires while it's still working?**
Another instance acquires the lock and you now have two recomputes — and worse,
if the release uses a plain `DEL`, the slow original could delete the *new*
holder's lock. That's why release must be a **compare-and-delete** (Lua:
`if get==mytoken then del`), so you only release a lock you still own. For
strict correctness under this scenario you need **fencing tokens**: the lock
hands out a monotonically increasing number and the protected resource rejects
writes carrying a stale token. This is Martin Kleppmann's well-known critique of
using locks (including Redlock) for correctness-critical mutual exclusion.

**Q: When is a distributed lock the wrong tool for stampede prevention?**
When the added waiter latency matters more than the duplicate work you're
avoiding, or when the value is cheap to compute. Locks serialize the miss path
and add a poll/wait for every non-leader. **Probabilistic early expiration
(XFetch)** or **stale-while-revalidate** are lock-free and lower-latency: no one
waits, because the value is refreshed before it disappears or served stale
during refresh. Use locks when the recompute is genuinely expensive and must not
run more than once.

**Q: A just-recovered node has cold caches and empty connection pools. Even
with jitter, sending it full traffic knocks it over. What do you do?**
Warm-up / slow-start: the load balancer ramps traffic to a freshly-healthy
node gradually rather than all at once (many LBs and service meshes have a
`slow_start_window`). Combine with server-side admission control so the node
sheds what it can't handle, and don't mark it healthy until caches/pools are
primed (a readiness probe that does real work, not just returns 200).

**Q: Circuit breaker half-open state — what's the trap?**
If half-open lets *many* trial requests through and they succeed, the breaker
snaps fully closed and the whole fleet resumes at once — a thundering herd
triggered by recovery. Half-open must admit only a **trickle** (often a single
probe or a small rate), and closing should ramp rather than switch, mirroring
LB slow-start.

**Q: How do you know your mitigations are working in production?**
Instrument it. Track retries-per-request (a retry storm shows as this ratio
spiking), backend request rate vs. client request rate (collapsing should make
backend << client on hot keys), cache-lock acquisitions and lock-wait time,
circuit-breaker state transitions, and the classic golden signals on the
backend. A dashboard panel of "backend QPS during/after an incident" is the
direct evidence: a flat plateau on recovery means your jitter+collapsing works;
a spike means it doesn't.

**Q: Does adding jitter hurt latency in the happy path?**
No — jitter only applies to *retries*, i.e. after a failure. Successful
first-attempt requests never sleep. The trade-off is only felt during
contention, where accepting a small randomized delay is vastly cheaper than a
retry storm that fails everyone.

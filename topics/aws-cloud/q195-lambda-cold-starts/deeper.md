# Deeper — follow-up interview questions

### 1. Define a cold start precisely — and what are its phases?

A cold start is the latency of **creating and initializing a new execution
environment** before Lambda can run the handler for the first time. Phases:
(1) **download** the deployment package/layers; (2) **start the runtime** — boot
the microVM sandbox and initialize the language runtime; (3) **run your init
code** — everything in the global scope outside the handler (imports, SDK/DB
client setup, config); (4) **invoke the handler**. A **warm** invocation skips
1–3 and goes straight to 4, which is why it's milliseconds. Cold starts hit
**tail latency**, not the median.

### 2. Init code vs handler code — why does the distinction matter?

Code in the **global/module scope** runs **once per environment** (on the cold
start) and its results **persist across warm invocations**; code **inside the
handler** runs **on every invocation**. So the lever is placement: build clients,
open connections, and load config **in the global scope** so warm requests reuse
them, and keep the handler lean. Putting connection setup inside the handler is a
classic bug — it makes *every* request slow, not just cold ones. Conversely,
doing unnecessary work in the global scope inflates every cold start.

### 3. What triggers cold starts, given Lambda's concurrency model?

Each concurrent request needs its **own execution environment**; Lambda reuses a
free warm one if available, otherwise it **builds a new one** (= a cold start).
So they're triggered by: a **new code version/deploy** (all environments are
new), a **traffic burst** that needs more concurrent environments than are
warm, and **idle reclamation** (Lambda tears down environments that sit unused,
so the next request scales from zero). Steady, moderate traffic sees very few;
spiky or bursty traffic sees many.

### 4. Provisioned Concurrency vs SnapStart — pick between them.

**Provisioned Concurrency** keeps **N environments pre-initialized and warm**;
those invocations have zero cold start, but **you pay for the warm capacity
continuously** (mitigate with scheduled scaling before peaks). It works for any
runtime. **SnapStart** takes a **snapshot of the initialized environment** and
**restores from it** on cold start instead of re-running init — big win for
slow-init runtimes (originally Java, now expanded) and **no charge for idle warm
capacity**. Rule of thumb: predictable high-traffic path where you want a
guaranteed warm floor → Provisioned Concurrency; slow-initializing runtime where
you want cheap fast cold starts → SnapStart. They can be combined.

### 5. What are the SnapStart gotchas?

Because many environments **restore from the same snapshot**, any state captured
during init is **shared/duplicated**: random seeds (weak crypto/uniqueness),
pre-generated unique IDs, cached timestamps, and **open network/DB connections**
(which are stale after restore). Fix with the **runtime hooks** — do such work in
an `afterRestore` hook so each restored environment re-seeds randomness and
re-opens connections. Also, the snapshot is per **published version**, so it
interacts with your versioning/alias flow.

### 6. How does RDS Proxy help with cold starts specifically?

It's less about cold-start *latency* and more about the **connection storm** that
cold starts cause. Each new execution environment opens its own DB connections;
a burst (or a deploy) spins up many environments at once and can **exhaust the
database's connection limit** or thrash it with connect/disconnect churn. **RDS
Proxy pools and multiplexes** connections in front of the database, so a swarm of
fresh Lambda environments shares a managed pool instead of each hammering the DB
directly. It also speeds reconnection and improves failover behaviour.

### 7. Does increasing memory help cold starts?

Often, yes — indirectly. Lambda allocates **CPU proportionally to memory**, so a
higher memory setting gives the init phase (and the handler) **more CPU**, which
can shorten CPU-bound initialization (runtime startup, JIT, decompression). It
doesn't reduce the *frequency* of cold starts, and it raises per-ms cost, so it's
a tuning knob: sometimes more memory finishes fast enough to be **cheaper
overall** despite the higher rate. Measure with Power Tuning rather than guessing.

### 8. Do keep-warm pings actually solve the problem?

Only partially, and they're the legacy hack. A scheduled ping keeps **one**
environment from being reclaimed, so it helps the "first request after idle"
case — but it does **nothing for bursts** (a spike still needs many new
environments) or **deploys** (new version, new environments). Provisioned
Concurrency and SnapStart supersede it. Mention pings to show you know the
history, but don't propose them as the fix for tail-latency-on-bursts.

### 9. How would you measure cold-start impact before optimizing?

Look at **CloudWatch**: the `Init Duration` field in Lambda logs (present only on
cold starts) tells you the init cost directly, and you can query it via **Logs
Insights** to see how often and how long. Track the **`ProvisionedConcurrency*`
metrics** (spillover/utilization) once you enable it, watch **P99 Duration** vs
median, and correlate cold-start spikes with **deploys and traffic bursts**.
X-Ray traces break a request into init vs handler. Optimize against those
numbers rather than assuming — many workloads with steady traffic barely feel
cold starts and don't need Provisioned Concurrency at all.

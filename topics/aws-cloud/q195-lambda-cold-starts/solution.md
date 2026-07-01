# Solution — Working around Lambda cold starts

## The core insight

A cold start is the **one-time cost of creating a new execution environment**
before Lambda can run your handler. It's a property of the concurrency model:
each concurrent request needs its own environment, and any request that can't be
served by an existing warm environment forces Lambda to build a fresh one. So
the two families of fixes are (A) **make that construction cheaper/faster**, and
(B) **arrange for a warm environment to already exist** so the hot path never
pays for construction at all. Conflating "made cold starts rarer" with "made
them not matter" is the mistake the interviewer is probing for.

## What actually happens on a cold start (the phases)

When a request arrives with no warm environment free, Lambda does, in order:

1. **Download** your deployment package and layers.
2. **Start the runtime / execution environment** — boot the Firecracker microVM
   sandbox and initialize the language runtime (JVM/CLR/Node/Python/…). Bigger,
   heavier runtimes cost more here (a cold JVM is the classic offender).
3. **Run your initialization code** — everything in the **global scope, outside
   the handler**: imports, building SDK clients, opening DB connections, reading
   config/secrets. **This is the phase you most directly control.**
4. **Invoke the handler** for the first time.

Phases 1–3 are the cold start. On a **warm** invocation the environment already
exists, so Lambda jumps straight to phase 4 — which is why warm invocations are
milliseconds and cold ones can be hundreds of ms to seconds.

**When they cluster** (matches the scenario): right after a **deploy** (a new
code version means every environment is new), during **scale-up bursts** (many
concurrent requests each need their own environment), and after **idle**
(Lambda reclaims environments that sit unused, so the next request scales from
zero). They hit **tail latency (P99)**, not the median.

## Bucket A — Make the cold start cheaper

### 1. Do less before the handler (optimize init)

The global-scope init runs on **every** cold start, so trim it:

- **Lazy-load heavy dependencies** — import a big library or build an expensive
  client only inside the code path that needs it, not unconditionally at module
  load. A function that only sometimes calls, say, an image library shouldn't pay
  to import it on every cold start.
- **Minimize work before the handler** — defer config parsing, warm-up loops,
  and anything not needed for the first request.

### 2. Reuse across invocations (init once, use warm)

Anything created in the **global scope persists across warm invocations of the
same environment**, so create it **once**:

- Build **SDK clients and DB connections at init**, store them in module-level
  variables, and reuse them in the handler. Re-creating a client per request
  wastes time on *every* invocation (not just cold ones).
- For relational databases, cold starts + bursts can **exhaust connection
  limits** (each new environment opens its own connections). Put **RDS Proxy** in
  front to pool and share connections, so a spike of new environments doesn't
  overwhelm the database.

### 3. Shrink the deployment package

Smaller package/layers = faster **download** (phase 1) and often faster init.
Trim unused dependencies, tree-shake/bundle, split rarely-used code into layers
loaded lazily, and avoid one giant monolith. Container-image functions have
their own image-pull cost — keep images lean too.

### 4. Choose a faster runtime

Interpreted/compiled-lightweight runtimes (**Node.js, Python, Go, Rust**) start
far faster than a **cold JVM or .NET**. If a function is latency-critical and
cold starts hurt, the runtime choice is a real lever — or use SnapStart (below)
to blunt a slow runtime's init.

## Bucket B — Take the cold start off the hot path

### 5. Provisioned Concurrency

Tell Lambda to keep **N execution environments pre-initialized and warm** on a
**version/alias**. Requests served by that pool have **no cold start** — init is
already done. Trade-off: **you pay for the provisioned capacity** whether or not
it's used. So:

- Set it on a **published version/alias** (not `$LATEST`).
- **Right-size N** to your steady concurrency, and use **Application Auto Scaling
  scheduled scaling** to **raise it before a known peak** and lower it after,
  rather than paying for peak capacity 24/7.
- Traffic above N still cold-starts normally — it's a floor of warm capacity,
  not a cap.

### 6. SnapStart (Java, and expanded runtimes)

Lambda takes a **snapshot of the fully initialized environment** and, on a cold
start, **restores from that snapshot** instead of re-running init. This slashes
cold starts for slow-to-init runtimes (dramatic for Java) **with no charge for
idle warm capacity** — the key cost difference vs Provisioned Concurrency.
Caveat: because many environments restore from the *same* snapshot, don't bake
per-instance state (random seeds, unique IDs, live connections) into init —
re-establish those in the **runtime hooks** (`afterRestore`) so each restored
env is unique/valid.

### 7. Keep-warm pings (mention, don't lead)

A scheduled (EventBridge) invocation every few minutes keeps at least one
environment from being reclaimed. It's crude: it doesn't help **bursts** (only
keeps a single env warm) or **deploys**, and it's largely superseded by
Provisioned Concurrency / SnapStart. Worth naming as the old hack, not the
recommendation.

## "Rarer" vs "doesn't matter"

- **Bucket A** makes each cold start **cheaper** and can make them **less
  frequent** (smaller package, faster runtime) — but a cold start still happens
  and still hits the tail.
- **Bucket B** makes the cold start **not matter on the hot path**: Provisioned
  Concurrency/SnapStart mean the served request doesn't pay construction cost.

The best answer does **both**: optimize init so that whatever cold starts remain
are cheap, and use Provisioned Concurrency (with scheduled scaling) or SnapStart
so your latency-critical paths don't eat them.

## The interview summary

> "A cold start is the one-time cost of Lambda spinning up a **new execution
> environment** — downloading the code, starting the runtime, and running my
> **init code outside the handler** — before the first invocation. It clusters
> after deploys, during scale-up bursts, and after idle scale-to-zero, and it
> hits P99. I attack it two ways. First, make init cheap: lazy-load heavy deps,
> build SDK/DB clients **once in the global scope** and reuse them (RDS Proxy for
> DB connection limits), shrink the package, and pick a fast runtime. Second,
> take it off the hot path: **Provisioned Concurrency** keeps N environments
> pre-warmed — with **scheduled scaling** before peaks so I'm not paying around
> the clock — and **SnapStart** restores Java (and other) functions from a
> post-init snapshot for fast cold starts with no idle cost. Keep-warm pings are
> the old hack, but they don't help bursts or deploys."

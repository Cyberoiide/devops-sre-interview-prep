# q210 — Deeper: Follow-up interview questions & answers

Concise, correct answers to the follow-ups an interviewer is likely to fire
after the main scenario.

---

### 1. Why not just put the DB check in the *liveness* probe?

Because liveness failure **restarts the container**. If the DB has a brief
outage, a DB-dependent liveness probe fails on every replica at once, kubelet
restarts them all, and on restart they still can't reach the DB — so they never
come up and you get a **cluster-wide `CrashLoopBackOff`**. You've turned a
transient dependency blip into a self-inflicted mass-restart outage, and
restarting a pod does nothing to fix an external DB. Liveness must only answer
"is *this process* wedged/deadlocked?" using local signals. Dependency awareness
belongs in readiness (traffic gating), never in liveness (restart).

---

### 2. What's the difference between a pod failing readiness and `CrashLoopBackOff`?

- **Readiness failing:** the container keeps running normally; kubelet just marks
  the pod NotReady and the endpoints controller **removes it from the Service
  endpoints** so it stops receiving traffic. No restart. When readiness passes
  again, it's added back. It's a *traffic-routing* state.
- **`CrashLoopBackOff`:** the container process actually **exited/crashed** (or
  failed liveness and was killed), and kubelet is **restarting it with
  exponential backoff**. It's a *lifecycle* state indicating the container can't
  stay up.

In our scenario the pods were Ready with **zero restarts** — that immediately
tells you the process is fine and it's a routing/config problem, not a crash.

---

### 3. How do you prevent cascading unreadiness from a deep readiness probe?

Several layers, used together:

- **Don't gate readiness on a *shared hard* dependency if you can avoid it.** If
  every replica shares the same DB, pulling them all out of rotation on a DB blip
  gives clients nowhere to go — it makes the outage total instead of partial.
- **Make the check lightweight, cached, and tolerant:** reuse a pooled connection
  health flag refreshed on an interval instead of a fresh heavy query per probe;
  set a sensible `failureThreshold`, `periodSeconds`, and `timeoutSeconds` so one
  slow response doesn't instantly flap the pod out.
- **Prefer gating the *rollout* on a real smoke test** (deploy-time) over gating
  *steady-state* readiness on the shared DB. That catches the config bug at
  deploy without arming a fleet-wide NotReady trigger during normal operation.
- **Serve degraded** where possible: stay Ready and serve DB-independent routes,
  return explicit errors only for the routes that truly need the DB.
- **Keep liveness dependency-free** so unreadiness never escalates into restarts.

---

### 4. How does `maxUnavailable` / `maxSurge` interact with a deep readiness probe during a bad rollout?

A pod only counts as **available** once it's **Ready**. With a deep readiness
probe, new pods that can't reach the DB never become Ready, so they never count
as available. `maxUnavailable` bounds how many old pods Kubernetes may take down,
and since the new pods aren't becoming available, Kubernetes **won't tear down
more old pods** — the rollout stalls with the old, working pods still Ready and
serving. `maxSurge` lets the new (broken) pods come up *alongside* the old ones
rather than replacing them. Net effect: the deep probe + `maxUnavailable`
together guarantee you keep serving capacity during a broken rollout, and
`kubectl rollout status` returns non-zero so automation can roll back. Set
`maxUnavailable: 0` for zero capacity loss during rollouts (needs surge headroom).

---

### 5. When would you add a `startupProbe`, and how does it relate to the others?

Use a startup probe for apps with a **slow or variable boot** (JVM warmup, large
cache load, migrations on start). While the startup probe is still failing,
**liveness and readiness are disabled**, so a slow boot won't trip liveness and
kill the container before it's up, and readiness won't flap during init. Once the
startup probe passes, liveness and readiness take over. This lets you keep
**liveness `periodSeconds` short** (fast detection of a wedged process in steady
state) without having to pad `initialDelaySeconds` to cover worst-case boot time.
Give the startup probe a generous `failureThreshold × periodSeconds` budget
covering the slowest legitimate boot.

---

### 6. How would a service mesh or a client-side circuit breaker change your answer?

A mesh (Istio, Linkerd) or resilient client adds an **application-traffic**
resilience layer that Kubernetes probes don't provide:

- **Outlier detection / circuit breaking** ejects an individual endpoint that
  starts returning errors, without needing kubelet to flip readiness — faster and
  more granular than the readiness/endpoints loop.
- **Retries, timeouts, and connection pooling** absorb brief DB blips so a
  hiccup doesn't surface as user-visible 500s, reducing the pressure to encode
  the dependency into readiness at all.
- **mesh-level health and metrics** give you the real error-rate signal to gate
  canary promotion and drive automatic rollback (Flagger integrates with mesh
  metrics for exactly this).

It doesn't remove the need for correct probes, but it shifts dependency-failure
handling toward the request path (retry/eject/degrade) and away from
coarse-grained, all-or-nothing readiness — which also mitigates cascading
unreadiness.

---

### 7. How do you detect and prevent secret-rotation drift causing this?

- **Single source of truth for secrets:** use External Secrets Operator, Vault,
  or a cloud secret manager so the DB's real credentials and the app's Secret are
  synced from one place and can't silently diverge.
- **Trigger a rollout on rotation:** when a secret rotates, re-sync the Kubernetes
  Secret and restart/roll the dependent workloads (e.g. Reloader, or a checksum
  annotation on the Pod template so a Secret change forces a new rollout).
- **Validate credentials in CI / at deploy:** a pre-deploy or post-deploy smoke
  test that actually authenticates to the DB with the shipped credentials catches
  a wrong/old password before or immediately after rollout.
- **Alert on auth failures:** monitor DB authentication-error rate; a spike right
  after a rotation or deploy is the drift signature.
- **Overlap old and new credentials during rotation** (dual-valid window) so
  there's no instant where neither the old nor new password works.

---

### 8. Your smoke test hits `/` and gets 200, but users still see errors. What might the shallow test miss, and how do you make deploy verification trustworthy?

A weak smoke test can pass while the service is broken if it:

- **Hits a cached or DB-independent route** (like `/healthz`) instead of a route
  that truly exercises the DB and downstream dependencies.
- **Runs against one pod / port-forward** instead of through the Service and
  load balancer, so it misses pods that are broken or misrouted.
- **Only checks status code**, not correctness (a 200 that returns stale/empty
  data), or doesn't cover **writes** (read path fine, write path failing).
- **Runs too early**, before the rollout actually shifts traffic to new pods.

Make it trustworthy: exercise the **real user path end-to-end through the
Service** (read *and* write), assert on **response content** not just status,
run it **after** `rollout status` succeeds and against the promoted version,
and **fail the pipeline + auto-rollback** on any failure. Pair it with real
**SLO/error-rate metrics** over a short bake window rather than a single request.

---

### 9. Should readiness ever check *transitive* dependencies (the DB's own upstreams, third-party APIs)?

Generally **no** — checking transitive/downstream dependencies in readiness is
where cascading unreadiness gets dangerous and hard to reason about. A flaky
third-party API you don't control would repeatedly flip your whole fleet
NotReady. Prefer to check only what this pod **directly and hard-requires** to
serve, keep it lightweight, and handle everything else on the **request path**
with timeouts, retries, circuit breakers, and graceful degradation (serve what
you can, return explicit errors for the rest). Readiness answers "should I get
traffic," not "is the entire dependency graph healthy" — the latter is the job
of observability and resilient client behavior, not a binary in/out-of-rotation
switch.

# q210 — Solution & Walkthrough

## 1. Read the timing: this is a config bug, not infrastructure

The single most important clue is that **the failure coincides exactly with the
deploy**. The previous version was serving fine; the database itself did not
change; nothing else in the environment moved. When a service breaks at the
precise moment a new version rolls out, the probability heavily favors a
**config change that the new version shipped**, not a random infrastructure
event.

For "pods are up but can't reach the DB", the config culprits are a short list:

- **A broken or rotated Secret** — a wrong DB password or API key. If secret
  rotation and the app config drifted (the DB got a new password but the app's
  Secret still has the old one, or vice versa), auth fails at connection time.
- **A wrong DB endpoint in a ConfigMap or env var** — wrong host, wrong port,
  wrong database name, wrong service name, or a bad value templated in by the
  CI/CD pipeline.
- **A network policy / RBAC / service-account change** shipped alongside the app
  that blocks egress to the DB — less common, same "coincides with deploy"
  signature.

You do *not* start by suspecting the database, the node, DNS at large, or a
cloud outage — those don't correlate with your deploy. Correlation with the
deploy is the whole diagnosis.

## 2. The real question: why did Kubernetes say everything was fine?

Because **"Ready" only means what the readiness probe checks**, and the probe
was **shallow**. A typical readiness probe is an HTTP `GET /healthz` that
returns 200 as long as the web server process is up, or a `TCP` check on the
app's own listening port. Neither of those touches the database.

So the chain of events is:

1. New pods start, the HTTP server binds its port, `/healthz` returns 200.
2. The readiness probe passes → Kubernetes marks each pod **Ready**.
3. The endpoints controller adds each Ready pod to the Service's `Endpoints`
   (really the `EndpointSlice`), so `kube-proxy` starts load-balancing live
   traffic to them.
4. The Deployment controller sees new pods becoming Ready, so the rolling update
   proceeds and eventually **completes**; `kubectl rollout status` reports
   success.
5. Meanwhile every request that needs the DB returns 500, because the pods
   cannot connect. Kubernetes never checked that path, so it never noticed.

**Ready ≠ can-actually-serve.** The probe measured a shallow proxy for health
(the process answers) instead of true serving capability (the request path
works end to end). That gap is the root cause of the *silent* nature of the
outage. The DB unreachability is the trigger; the shallow probe is why it
shipped silently.

## 3. The three probes — precise semantics

You must be crisp about what each probe means and what Kubernetes *does* when it
fails, because the whole fix hinges on it.

| Probe | Question it answers | Action on failure | May depend on external deps? |
|-------|--------------------|--------------------|------------------------------|
| **liveness** | "Is the process wedged / deadlocked? Should I restart it?" | **kubelet restarts the container** (per `restartPolicy`, with backoff → `CrashLoopBackOff`). | **No.** Never gate liveness on a dependency. |
| **readiness** | "Should this pod receive traffic *right now*?" | **kubelet marks the pod NotReady → the endpoints controller removes it from the Service endpoints.** The container is **not** restarted. | Cautiously — this is the double-edged sword. |
| **startup** | "Has the app finished booting?" | While failing, **liveness and readiness are disabled**; if it exceeds its threshold the container is restarted. | No; it's about boot time, for slow starters. |

Key mechanics that come up in follow-ups:

- **Readiness failing does not restart anything.** It only pulls the pod out of
  rotation. The pod keeps running; when readiness passes again it is re-added to
  endpoints. This is exactly the behavior we want to exploit for a safe rollout.
- **Liveness failing restarts the container.** This is why a dependency check in
  liveness is dangerous: if the DB is briefly down, liveness fails, Kubernetes
  restarts your pods, they still can't reach the DB on boot, and you get a
  **cluster-wide crash loop** — you've converted a dependency blip into a
  self-inflicted mass restart.
- **Startup probe** exists so slow-booting apps don't get killed by liveness
  before they finish initializing, and so readiness doesn't flap during boot. It
  runs first; once it passes, liveness and readiness take over.

## 4. Triage commands (how you'd confirm it live)

```bash
# 1. Is the rollout "green" while the service is down? (confirms the paradox)
kubectl rollout status deploy/web
kubectl get pods -l app=web            # all 1/1 Ready, no restarts, no CrashLoop

# 2. Are the (broken) pods actually receiving traffic?
kubectl get endpoints web              # broken pods listed -> they get traffic
kubectl get endpointslices -l kubernetes.io/service-name=web

# 3. What does the app actually say? (the real error)
kubectl logs -l app=web --tail=50      # "cannot connect to <host>:<port>"

# 4. What config did the new version ship? (find the bad value)
kubectl get deploy web -o yaml | grep -A20 env:
kubectl set env deploy/web --list      # inspect DB_HOST / DB_PORT / DB_NAME
kubectl get configmap app-config -o yaml
kubectl get secret db-credentials -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d

# 5. Compare to the last-good revision — the diff IS the root cause
kubectl rollout history deploy/web
kubectl rollout history deploy/web --revision=<current>
kubectl rollout history deploy/web --revision=<previous>

# 6. Prove it from inside a pod: can it even reach the DB?
kubectl exec deploy/web -- sh -c 'nc -zv $DB_HOST $DB_PORT'
kubectl exec deploy/web -- nslookup $DB_HOST

# 7. Fastest mitigation once confirmed: roll back
kubectl rollout undo deploy/web
```

The `rollout history` diff between the current and previous revision is usually
where you *see* the bad Secret/ConfigMap/env change with your own eyes.

## 5. The fix: make readiness reflect true serving capability

Change readiness from a shallow `/healthz` to a **deep `/readyz`** that verifies
the pod can actually do its job — including reaching the required backends
(e.g. a DB ping / `SELECT 1`). Keep **liveness shallow** on `/healthz`.

```yaml
livenessProbe:                 # shallow ON PURPOSE — "is the process wedged?"
  httpGet: { path: /healthz, port: 8080 }
  periodSeconds: 10
readinessProbe:                # deep — "can I actually serve a request?"
  httpGet: { path: /readyz, port: 8080 }
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3          # tolerate a brief blip before flipping out
startupProbe:                  # for slow boots — gates the other two
  httpGet: { path: /healthz, port: 8080 }
  periodSeconds: 3
  failureThreshold: 20
```

Why this fixes the *silent rollout* specifically:

- With a deep `/readyz`, a pod that can't reach the DB **never becomes Ready**.
- A not-Ready pod is **never added to the Service endpoints**, so it **never
  receives production traffic**.
- The Deployment's `RollingUpdate` strategy will not proceed past a batch of
  pods that won't become Ready. With `maxUnavailable` keeping old pods up and
  `maxSurge` bringing new ones up alongside them, the **old, working pods keep
  serving** while the new bad pods sit at `0/1 Ready`.
- The rollout **stalls** and `kubectl rollout status` **returns non-zero** —
  which is precisely the machine-readable signal your CI/CD pipeline should gate
  on to trigger an automatic rollback.

The bad deploy is now **self-arresting** instead of a silent outage.

### How `maxUnavailable` / `maxSurge` interact with this

`RollingUpdate` defaults are `maxUnavailable: 25%`, `maxSurge: 25%`.
`maxUnavailable` bounds how many pods can be down at once; because a pod only
counts as "available" once it's Ready, new pods that never become Ready are
never "available", so Kubernetes refuses to tear down more old pods. That's the
built-in safety: the deep readiness probe plus `maxUnavailable` together
guarantee you always have Ready old pods serving while a broken new version
fails to come up. Set `maxUnavailable: 0` if you want *zero* capacity loss
during rollouts (surge-only), at the cost of needing spare scheduling headroom.

## 6. The critical caveat: deep dependency checks are a double-edged sword

This is the senior-level nuance and interviewers look for it.

If **every** pod's readiness depends on a **shared hard dependency** (the DB),
then a *transient* DB blip — a failover, a brief network partition, a few
seconds of DB CPU saturation — makes **every pod flip NotReady simultaneously**.
The endpoints controller then empties the Service of *all* endpoints, and now
you have a **total outage even though every pod is perfectly healthy**. You've
converted a short, possibly self-healing dependency hiccup into a full
self-inflicted outage, and often a *longer* one (thundering-herd reconnect,
readiness flapping). This is **cascading unreadiness**.

So there are honestly two schools of thought:

**School A — "readiness must reflect real serving capability."**
If a pod genuinely cannot serve requests, it should be pulled from rotation;
sending traffic to a pod that will 500 is worse than sending it elsewhere. Deep
readiness is correct.

**School B — "don't gate readiness on shared hard dependencies."**
If the dependency is shared by *all* replicas, taking them all out of rotation
doesn't help — there's nowhere healthy to route to. Better to stay Ready, serve
what you can, surface the error via metrics/alerts, and rely on strong deploy
verification + fast rollback to catch config breakage. Reserve readiness for
things that are *per-pod* (this pod's warm cache, this pod's local queue depth).

The mature answer synthesizes both, using **mitigations** rather than picking a
dogmatic side:

1. **Never put the dependency check in liveness.** Otherwise a DB blip → mass
   restart → crash loop. Liveness = local process health only.
2. **Make the readiness dependency check lightweight, cached, and tolerant.**
   Don't open a fresh connection and run a heavy query every probe; check a
   cached/pooled connection health flag refreshed on an interval. Use
   `failureThreshold` (and reasonable `periodSeconds`/`timeoutSeconds`) so a
   one-off blip doesn't immediately flap the pod out.
3. **Use a startup probe for slow starters** so boot-time DB unavailability
   doesn't trip liveness and cause crash loops before the app is even up.
4. **Distinguish "deploy-time verification" from "steady-state readiness."**
   The strongest version: keep steady-state readiness mostly *per-pod* (avoid
   cascading unreadiness), but gate the **rollout itself** on a real end-to-end
   smoke test that exercises the DB path (below). That catches the config bug at
   deploy time without arming a cluster-wide outage trigger during normal running.
5. **Consider degraded-serving semantics.** If part of your traffic doesn't need
   the DB, a pod can stay Ready and serve that, returning explicit errors only
   for DB-dependent routes — instead of going fully dark.

There is no universally correct answer; the right call depends on whether the
dependency is per-pod or shared, whether there's a fallback, and how good your
deploy verification and rollback are. Say that out loud in the interview.

## 7. Prevention layers beyond probes (defense in depth)

Probes are one layer. A robust pipeline catches this before and during deploy:

- **Validate config & Secrets in CI, before deploy.** Schema-validate ConfigMaps
  and required env vars; assert the DB host/port/name are present and
  well-formed; verify referenced Secret keys exist. Catch typos and missing keys
  before they ever reach the cluster.
- **Detect secret-rotation drift.** Use a secrets manager (External Secrets
  Operator, Vault, cloud secret manager) as the single source of truth so the
  app's Secret and the DB's actual credentials can't silently diverge. Alert
  when a rotation happens so the dependent workloads are re-synced/restarted.
- **Progressive delivery.** Canary or blue-green so a bad version only sees a
  small slice of traffic first. Gate promotion on `kubectl rollout status`
  **and** on real SLO/error-rate metrics (Argo Rollouts / Flagger automate
  this), with **automatic rollback** when the canary's error rate spikes.
- **Post-deploy smoke test that actually exercises the DB path.** After rollout,
  run a test that hits the real request path (not `/healthz`) and asserts a 200
  from a route that touches the database. This is what would have caught the 500s
  immediately. Fail the pipeline and roll back on a failed smoke test.
- **Fast, automated rollback.** `kubectl rollout undo` (or Argo's automatic
  rollback) wired to the rollout-status/smoke-test/metric signals, so recovery
  is measured in seconds and doesn't depend on a human noticing.

## 8. One-paragraph answer (if the interviewer wants the TL;DR)

> The failure coincides exactly with the deploy, so it's a config change the new
> version shipped — most likely a rotated/wrong Secret or a wrong DB
> host/port/name. It shipped *silently* because the readiness probe was shallow:
> it checked `/healthz` (process up) instead of real serving capability, so
> Kubernetes marked the pods Ready, added them to the Service endpoints, and let
> the rollout complete while every request 500'd. The fix is a deep `/readyz`
> that verifies the DB dependency, so bad pods never become Ready — the rolling
> update then halts itself (old pods keep serving via `maxUnavailable`) and
> `rollout status` fails, which CI/CD uses to auto-rollback. The caveat is that a
> shared hard dependency in readiness can cause cascading unreadiness (a DB blip
> flips *all* pods NotReady → total outage), so keep liveness dependency-free,
> make the readiness check lightweight/cached/tolerant, prefer gating the
> *rollout* on a real smoke test over gating steady-state readiness on a shared
> DB, and back it all with CI config validation, canary + auto-rollback, and
> secret-rotation drift detection.

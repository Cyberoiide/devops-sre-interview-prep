# Deeper — follow-up interview questions

### 1. 503 vs. 502 — how do you tell them apart fast?

Run `kubectl get endpoints <svc>`. If it's **empty**, it's **503** — the ingress
had nowhere to send the request. If it's **not empty**, the ingress had a backend
to try but the connection/response failed, which is **502** (or 504 if it timed
out). 503 = "no one to call"; 502 = "called someone, got garbage / a hang-up."

### 2. What's the default NGINX Ingress proxy timeout, and how do you change it?

Default is **60 seconds** for `proxy-connect-timeout`, `proxy-send-timeout`, and
`proxy-read-timeout`. Override per-Ingress with annotations:

```yaml
nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"
nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
```

Or globally in the controller's ConfigMap (`proxy-read-timeout`, etc.).
`proxy-read-timeout` is the one that matters for slow response bodies and is the
usual fix for a legitimate-but-slow endpoint returning 504.

### 3. Why do you often see a burst of 502s during a rolling deploy?

Two classic reasons, both about connections to a pod that's going away:

- **Keep-alive races:** NGINX keeps upstream connections open for reuse. When a
  pod is terminated, NGINX may send a request over a connection to that pod just
  as it closes, getting a reset → 502 (`upstream prematurely closed`).
- **No graceful shutdown:** the pod receives SIGTERM and exits immediately while
  still in the Endpoints list (endpoint removal is eventually consistent). The
  fix is a `preStop` hook with a short sleep plus honoring
  `terminationGracePeriodSeconds`, so the pod keeps serving in-flight requests
  until it's removed from Endpoints. Readiness-gated draining and
  `proxy-next-upstream` also help.

### 4. How does the readiness probe relate to 503?

Directly. A pod that fails (or hasn't yet passed) its readiness probe is **not**
added to the Service's Endpoints. If no pod is Ready, Endpoints is empty and the
ingress returns **503**. So a too-strict or too-slow readiness probe, or a probe
hitting a broken health route, manifests as 503 at the edge even though the pods
are "running." Conversely, a proper readiness probe is what *prevents* traffic
from reaching a pod that would otherwise 502/500.

### 5. What is `proxy-next-upstream` and when does it help?

`nginx.ingress.kubernetes.io/proxy-next-upstream` controls which failure
conditions cause NGINX to **retry the request against the next backend** in the
upstream group (e.g. `error timeout http_502`). With multiple replicas, a
transient failure on one pod (a crash, a reset during deploy) can be retried on a
healthy pod instead of surfacing as a 502/504 to the client. Pair it with
`proxy-next-upstream-tries` and `proxy-next-upstream-timeout`. Caveat: don't
enable retries for non-idempotent requests (POST) unless you're sure duplicates
are safe.

### 6. Bonus codes: what are 413 and 499?

- **413 Request Entity Too Large:** the request body exceeds
  `nginx.ingress.kubernetes.io/proxy-body-size` (default **1m** in
  ingress-nginx). Common on file uploads; raise the annotation to fix.
- **499 Client Closed Request:** an NGINX-specific code meaning the **client**
  disconnected before the server responded — usually the user/browser gave up or
  a client-side timeout fired while the backend was still working. It's a client
  behavior, often a downstream symptom of a slow backend, not a server error per
  se.

### 7. How do you distinguish an app-generated 500 from an ingress-generated 500?

Compare **upstream status** vs. **returned status** in the controller access log
(`$upstream_status` vs `$status`). If both are 500, the **app** returned it and
NGINX passed it through — go read the application logs. If the upstream status is
empty/`-` and NGINX logged an internal error, the controller itself failed
(rare — e.g. a bad snippet/config, auth subrequest failure). Also: an
app-generated 500 usually carries the **application's own response body**, while
edge-generated errors carry NGINX's default error page.

### 8. You get 504 but the app logs show the request completing in 200ms. What now?

The mismatch points away from the app's own processing time. Check: (a) is the
504 actually a `proxy-connect-timeout` (can't even establish the connection —
network policy, wrong port) rather than a read timeout? (b) Is there a
service-mesh sidecar or another proxy hop adding latency or holding the
connection? (c) Is the app writing the response but not flushing/closing so NGINX
keeps waiting? Read the controller **error** log to see which timeout fired
(`connecting to upstream` vs `reading response header`) — that tells you which
leg is slow.

### 9. All four codes are "5xx" — why does the distinction matter operationally?

Because they route to completely different owners and fixes. **500** pages the
**app team** (code/dependency bug). **503** is usually a **deploy/capacity/probe**
problem (scale up, fix readiness, fix selector) and often self-heals when pods go
Ready. **504** is a **latency** problem (optimize the backend or its downstream,
or raise the timeout). **502** is a **connectivity/lifecycle** problem (wrong
port, crash, graceful-shutdown gap). Misreading a 502-during-deploy as an app bug
wastes an on-call's night; the endpoints-and-logs check keeps you honest.

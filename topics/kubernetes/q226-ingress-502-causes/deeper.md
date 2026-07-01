# Deeper — follow-up interview questions

Concise, correct answers to the follow-ups an interviewer is likely to drill
into after the main 502 question.

---

### 1. Why do 502s spike during *every* deploy, and how do you eliminate them?

Because pod termination and endpoint removal are **asynchronous and racy**. When
a pod starts terminating it (by default) receives SIGTERM and can exit almost
immediately, but the ingress controller only stops routing to it once the
EndpointSlice update propagates and it reconfigures its upstreams — tens to
hundreds of ms later. During that window nginx sends requests (or holds
keepalive connections) to a pod that is already gone → connection reset /
prematurely closed → **502**.

Eliminate it with the graceful-shutdown pattern:

- A `preStop` hook that **sleeps** (e.g. 5–15s) so the container keeps serving
  while endpoint removal propagates to the ingress.
- `terminationGracePeriodSeconds` **larger** than the preStop sleep + drain
  time (otherwise the kubelet SIGKILLs mid-drain).
- The app handles SIGTERM by finishing in-flight requests then exiting
  (graceful HTTP server close).
- A rollout strategy of `maxUnavailable: 0` + small `maxSurge`, so new **ready**
  pods exist before old ones are removed.
- A real `readinessProbe` so nginx never routes to a pod that isn't serving yet.

---

### 2. What is the keepalive timeout ordering rule, and why?

**The backend's idle/keepalive timeout must be greater than the proxy's
upstream keepalive timeout.** The proxy (nginx-ingress) pools idle connections
to pods. If the backend closes an idle connection first, nginx may later reuse
that already-closed connection, write a request onto a dead socket, and get a
502 (`upstream prematurely closed connection`). Making the backend always
outlive the proxy's idle window guarantees the *proxy* is the side that decides
a connection is stale, so it never sends on a socket the backend has closed.
(Same principle as the AWS ALB ↔ backend `idle-timeout` ordering rule.)

---

### 3. What's the root-cause difference between a 502 and a 504?

Both mean the proxy **reached** a backend (unlike 503). The difference is what
happened next:

- **504 Gateway Timeout:** the backend accepted the connection and the request,
  but did not send a response within the proxy's timeout — the app is **too
  slow / hung** (fix: speed up the app or raise `proxy-read-timeout`).
- **502 Bad Gateway:** the backend gave back something **invalid or unusable** —
  the connection was refused, reset, closed prematurely, or the reply was
  malformed HTTP. The conversation **broke**, it didn't just take too long.

504 = "answered too late / not at all"; 502 = "answered wrong / hung up on me."

---

### 4. Which nginx log line tells you the upstream connection died, and what's the difference between the common ones?

Grep the controller log for `upstream`:

- `connect() failed (111: Connection refused)` — nginx couldn't even establish
  the TCP connection; **nothing is listening** on that port (wrong `targetPort`,
  app not up). `111` = `ECONNREFUSED`.
- `recv() failed (104: Connection reset by peer)` — connection was established
  then **forcibly reset** mid-request; the pod/backend TCP stack went away.
  `104` = `ECONNRESET`.
- `upstream prematurely closed connection while reading response header from
  upstream` — the backend **closed the socket** before finishing the HTTP
  response (graceful FIN mid-response); classic for pod-dying and keepalive
  races.

`111` points at wiring (port/protocol); the other two point at a
connection that existed and then broke (pod lifecycle, keepalive, controller).

---

### 5. How does `proxy-next-upstream` help, and when does it hurt?

`nginx.ingress.kubernetes.io/proxy-next-upstream` tells nginx to **retry the
request on the next upstream** on certain failures (e.g. `error timeout
http_502`). It masks transient races (a dead keepalive connection, a pod that
just went away) by transparently retrying on a healthy pod, so the client never
sees the 502.

It **hurts on non-idempotent requests.** If nginx already sent the request body
and the failure happened after the backend partially processed it, retrying can
**apply the operation twice** (a duplicate POST/payment/side effect). So retries
are safe for idempotent methods (GET/HEAD/PUT/DELETE by contract) but dangerous
for POST unless the endpoint is idempotent (idempotency keys). Treat it as a
mitigation, not a substitute for fixing the underlying race.

---

### 6. Why can the ingress controller *itself* cause 502s?

The controller is just an nginx reverse-proxy running in a pod. If that pod is
**OOMKilled**, CPU-throttled, or hits `worker_connections` / file-descriptor
limits, its worker processes are killed and restarted, dropping in-flight
connections → 502. Heavy config churn (many Ingress/endpoint changes triggering
reloads) can also briefly disrupt connections. The tell is that 502s correlate
with the **controller's** restart count (not your app's) and often appear
cluster-wide across many Ingresses at once. Fix with proper controller
requests/limits, higher fd/worker-connection limits, and scaling the controller.

---

### 7. How does a readiness probe help prevent 502s (and how is it different from liveness)?

A **readiness** probe controls **Endpoints membership**: a pod is only added to
the Service's endpoints (and thus routed to by the ingress) when it passes
readiness, and it's removed when it fails. This prevents nginx from sending
traffic to a pod that isn't ready to serve yet (during startup) or that has
started failing — both would otherwise produce 502s. A **liveness** probe, by
contrast, **restarts** the container when it fails; it does not remove the pod
from rotation gracefully and, if misconfigured, can *cause* churn/502s by
killing pods. For 502 prevention you care most about readiness (plus the
graceful-shutdown pattern for the removal side of the lifecycle).

---

### 8. What's the difference between Service `port`, Service `targetPort`, and container `containerPort`, and how does a mismatch cause a 502?

- **`containerPort`** — the port the process inside the container actually
  listens on. (Informational in the pod spec, but it documents reality.)
- **Service `targetPort`** — the port on the **pod** that the Service forwards
  to. This must equal the real listening port.
- **Service `port`** — the port the **Service** is exposed on (what the Ingress
  backend references via `service.port.number`).

Traffic flows: client → ingress → Service `port` → `targetPort` (pod) →
process on `containerPort`. If `targetPort` points at a port nothing listens on,
nginx connects to pod-IP:targetPort and gets `connect() failed (111: Connection
refused)` → **502**. Crucially, the endpoint can still look "ready" if the
readiness probe checks a *different* working port, so the misconfiguration
hides until traffic actually flows.

---

### 9. If you see a 503 instead of a 502 during a deploy, what changed?

A **503** means the ingress has **no ready endpoints at all** for that Service —
every pod is simultaneously not-ready/terminating, so there's nobody to route
to. During a deploy this happens when all old pods are removed before new ones
become ready (e.g. `maxUnavailable` too high, slow readiness, or replicas=1).
The fix overlaps with the 502 fix but emphasizes **surge/availability**:
`maxUnavailable: 0` + `maxSurge`, more replicas, a `PodDisruptionBudget`, and
fast/accurate readiness so a ready pod always exists. (502 = talked to a dying
pod; 503 = there was no pod to talk to.)

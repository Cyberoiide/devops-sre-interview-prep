# Solution — 502 from an Ingress controller: causes, signatures, fixes

## 1. The anchor: 502 vs 503 vs 504

All three are "the gateway (the ingress/reverse proxy) has a problem serving
your request," but they say very different things about *where* the problem is.
Getting this distinction crisp is the core of the answer.

| Code | nginx meaning | What the proxy experienced | Typical root cause |
|------|---------------|----------------------------|--------------------|
| **503** Service Unavailable | No usable backend to send to | The proxy has **no ready endpoint** to route to at all | Zero ready pods; all endpoints removed; controller has an empty upstream set; rate-limited / maintenance |
| **504** Gateway Timeout | Backend reached, but **too slow** | The proxy **connected and sent** the request, but the backend did not respond within the timeout | Slow app, deadlock, DB stall, `proxy-read-timeout` too low |
| **502** Bad Gateway | Backend reached, but reply was **invalid/unusable** | The proxy **talked to a backend and the conversation broke** — connection refused, reset mid-request, closed prematurely, or malformed HTTP | Pod dying mid-request, keepalive race, wrong port/protocol, controller crash |

The one-line mental model:

- **503 = "I had nobody to talk to."**
- **504 = "I talked to them and they never answered in time."**
- **502 = "I talked to them and the conversation broke / the answer was
  garbage."**

So a 502 always means nginx **did** get to a backend socket. That is why the
debugging is about *the connection to the pod* and *the pod's lifecycle/health*,
not about "is there an endpoint" (503) or "is it slow" (504).

---

## 2. The 502 causes, each with its mechanism

### Cause A — Backend pod dying / deregistering mid-request (the deploy 502)

**Symptom:** a burst of 502s during **every rollout, scale-down, eviction,
OOMKill, or node drain**. Steady-state is clean; deploys hurt.

**Mechanism — the endpoint-removal vs connection race.** When a pod is
terminated, two things need to happen: (1) it must be removed from the
Service's Endpoints/EndpointSlice so the ingress stops sending it new requests,
and (2) it must stop running. In Kubernetes these are **not synchronized and
propagate asynchronously**:

1. The pod is marked `Terminating`; the kubelet begins shutdown and (by
   default) sends the container **SIGTERM** essentially right away (after any
   `preStop` hook).
2. In parallel, the API server updates the EndpointSlice, the endpoints
   controller notices, and the ingress controller re-reads endpoints and
   reconfigures its upstream set.

Step 2 takes tens to hundreds of milliseconds (often more under load). If the
container in step 1 exits **before** the ingress finishes step 2, nginx is
still holding the pod in its upstream set — or has an in-flight keepalive
connection to it — and sends a request to a socket that is already gone. The
TCP connection is reset or closed mid-response → **502**.

**Log signatures:**

```
upstream prematurely closed connection while reading response header from upstream
recv() failed (104: Connection reset by peer) while reading response header from upstream
```

(`104` = `ECONNRESET`.)

**Fix — the graceful shutdown pattern.** Keep the pod alive and serving for a
short window *after* termination begins, so endpoint removal reaches nginx
before the process dies:

```yaml
spec:
  terminationGracePeriodSeconds: 30      # must be > preStop sleep + drain time
  containers:
    - name: app
      readinessProbe:                     # so a pod only receives traffic when actually ready
        httpGet: { path: /healthz, port: 8080 }
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 10"]   # bridge the endpoint-removal window
```

- `preStop` runs **before** SIGTERM; the `sleep` holds the container open while
  the Endpoints update propagates and nginx stops routing new requests to it.
- The app should also handle SIGTERM by finishing in-flight requests and then
  exiting (graceful HTTP server shutdown). The `sleep` covers the propagation
  gap; graceful SIGTERM handling covers the in-flight requests.
- `terminationGracePeriodSeconds` must exceed the `preStop` sleep plus expected
  drain time, or the kubelet SIGKILLs the pod mid-drain.
- Pair with a rollout strategy that does not remove old pods before new ones
  are ready:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

### Cause B — Keepalive / connection-timeout mismatch

**Symptom:** intermittent 502s **not** correlated with deploys, often under
low or bursty traffic where connections sit idle between requests.

**Mechanism.** nginx-ingress keeps a pool of idle **keepalive** connections to
each upstream pod to avoid a TCP+TLS handshake per request. Each side has its
own idea of how long an idle connection may live. If the **backend/app server
closes idle keepalive connections sooner** than nginx does, there is a race:
nginx pulls a connection from its pool believing it is alive, but the backend
already sent a FIN and closed it. nginx writes a request onto the dead socket,
gets a reset or immediate close, and returns **502** (`upstream prematurely
closed connection`).

This is exactly the classic AWS ALB ↔ backend `idle-timeout` ordering bug: the
component *closest to the client* (the proxy) must be the one that decides a
connection is stale.

**Fix — the timeout-ordering rule:**

> **Backend idle/keepalive timeout > proxy (nginx) upstream keepalive timeout.**

Concretely: raise the app server's keepalive timeout above nginx's
`upstream-keepalive-timeout` (default 60s), e.g. gunicorn `--keep-alive`,
Node `server.keepAliveTimeout`/`headersTimeout`, Go `Server.IdleTimeout`. Or
lower nginx's `upstream-keepalive-timeout` below the backend's. Either way the
backend must never be the first to close.

nginx-ingress knobs (controller ConfigMap):
`upstream-keepalive-timeout`, `upstream-keepalive-requests`,
`upstream-keepalive-connections`.

As a mitigation, `nginx.ingress.kubernetes.io/proxy-next-upstream:
"error timeout http_502"` makes nginx retry the request on another upstream
connection — but only enable retries for **idempotent** requests (see
`deeper.md`), since a retried POST can double-apply.

### Cause C — The ingress controller itself is under-resourced

**Symptom:** 502s that correlate with the **controller pod's** restarts, not
your app's; often cluster-wide across many Ingresses at once.

**Mechanism.** If the ingress-nginx controller pod is CPU-throttled,
memory-starved, **OOMKilled**, or hitting `worker_connections` /
file-descriptor limits, its worker processes get killed/restarted and drop
in-flight connections → 502. Frequent config reloads under heavy churn (many
Ingress/endpoint changes) can also briefly disrupt connections.

**Log/inspection signatures:** controller pod `RESTARTS > 0`, `OOMKilled` in
`kubectl describe`, high `kubectl top pod` usage, `worker_connections are not
enough` in the controller log.

**Fix.** Give the controller adequate CPU/memory **requests and limits**,
monitor restarts, raise `max-worker-connections` and fd limits under high
connection counts, and scale the controller (more replicas / HPA) for churn.

### Cause D — Wrong port, wrong protocol, malformed reply

These are all "nginx reached *something* but couldn't get a valid HTTP reply."

- **Wrong `targetPort` / app not listening:** the Service `targetPort` points
  at a port nothing listens on. Endpoints may still show "ready" (the readiness
  probe hits a *different*, working port), so nginx connects to pod-IP:badport
  and gets refused. **Signature:** `connect() failed (111: Connection refused)`
  (`111` = `ECONNREFUSED`). **Fix:** make `targetPort` match the real listening
  `containerPort`.
- **Protocol mismatch (TLS):** the backend serves **HTTPS** but nginx proxies
  plain **HTTP** to it (or the reverse). nginx gets a TLS handshake where it
  expects HTTP text → unusable reply → 502. **Fix:**
  `nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"` (or fix the backend).
- **Response header too large:** upstream sends response headers bigger than
  nginx's proxy buffers → 502. **Fix:** raise `proxy-buffer-size` /
  `proxy-buffers-number`.
- **Backend emits non-HTTP / garbage** on the port (e.g. the Service points at
  a gRPC-only or raw-TCP port while nginx speaks HTTP/1.1).

---

## 3. Exact log signatures cheat-sheet

Read them with:

```bash
kubectl logs -n ingress-nginx <controller-pod> --tail=100 | grep -i upstream
```

| Log line fragment | errno | Meaning | Points to |
|-------------------|-------|---------|-----------|
| `connect() failed (111: Connection refused)` | ECONNREFUSED | nothing listening on the upstream port | wrong `targetPort`, app not up (Cause D) |
| `recv() failed (104: Connection reset by peer)` | ECONNRESET | backend killed the TCP connection mid-request | pod dying (A), keepalive race (B), controller crash (C) |
| `upstream prematurely closed connection while reading response header` | — | backend closed the socket before finishing the HTTP response | pod dying (A), keepalive race (B) |
| `upstream sent too big header` | — | response headers exceed proxy buffer | header-size (D) |
| `worker_connections are not enough` | — | controller hit its connection limit | controller resourcing (C) |
| `SSL_do_handshake() failed` / plain-HTTP-to-TLS errors | — | protocol mismatch to backend | backend-protocol (D) |

Note that A, B, and C can *all* produce the "prematurely closed / connection
reset" pair — that is why you correlate with **timing** (deploy vs steady
state), **controller restarts**, and **traffic pattern (idle vs busy)** to
disambiguate.

---

## 4. Debugging workflow (what you actually do)

1. **Confirm it's 502, not 503/504.** That alone says "nginx reached a
   backend; the connection/reply broke" — skip the "is there an endpoint" and
   "is it slow" rabbit holes.
2. **Read the controller logs** and grep for `upstream` — the line includes the
   upstream address, upstream status, and one of the signatures above. This
   usually names the cause outright.
3. **Correlate with timing.** Do the 502s spike during deploys/scale events?
   → Cause A (graceful shutdown). Random under idle traffic? → Cause B
   (keepalive). Cluster-wide with controller restarts? → Cause C.
4. **Check the app pods:** `kubectl get pod` (RESTARTS/OOMKilled),
   `kubectl describe pod`, and whether `preStop` +
   `terminationGracePeriodSeconds` + a real `readinessProbe` exist.
5. **Check the controller pod:** `kubectl get pod -n ingress-nginx` (RESTARTS),
   `kubectl describe` (OOMKilled / Last State), `kubectl top pod`.
6. **Verify wiring:** does the Service `targetPort` match a port the container
   actually listens on? Is `backend-protocol` correct for a TLS backend?
7. **Apply the matching fix** from §2 and re-run under load to confirm the 502
   rate drops to ~0.

---

## 5. Differential summary

| Cause | When it fires | Log signature | Fix |
|-------|---------------|---------------|-----|
| **A. Pod dying mid-request** | During deploys / scale-down / eviction | `upstream prematurely closed`, `104 Connection reset` | `preStop` sleep + `terminationGracePeriodSeconds` + readiness; SIGTERM graceful shutdown; `maxUnavailable: 0` |
| **B. Keepalive mismatch** | Steady state, idle/bursty traffic | `upstream prematurely closed` (no deploy correlation) | **backend idle timeout > nginx `upstream-keepalive-timeout`**; `proxy-next-upstream` for idempotent reqs |
| **C. Controller starved** | Correlates with controller restarts, cluster-wide | controller RESTARTS/OOMKilled, `worker_connections not enough` | controller CPU/mem requests+limits; raise fd/worker-connections; scale controller |
| **D. Wrong port** | Immediately, 100% of requests | `connect() failed (111: Connection refused)` | fix Service `targetPort` to the real `containerPort` |
| **D. Wrong protocol / big header / garbage** | Immediately or on large responses | `SSL handshake` / `upstream sent too big header` | `backend-protocol: HTTPS`, raise `proxy-buffer-size`, fix backend |

The headline takeaway for the interviewer: **502 means the ingress reached a
backend and the exchange failed.** The single most common production instance
is the deploy race in Cause A, fixed by the `preStop` + graceful-termination +
readiness pattern.

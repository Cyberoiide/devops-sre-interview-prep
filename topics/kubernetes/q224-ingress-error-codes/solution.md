# Solution — Reading Ingress 5xx codes

## The core distinction: edge vs. app

Every 5xx you see from an Ingress falls into one of two buckets:

1. **The app generated the error** and the ingress merely proxied it through.
   The classic case is **500**.
2. **The ingress/edge generated the error** on the app's behalf because
   something went wrong between the proxy and the backend. These are **502**,
   **503**, and **504**.

The fastest way to tell these apart in NGINX Ingress is to compare the
**upstream status** (what the backend actually returned) with the **returned
status** (what the client got). NGINX access logs expose both. If they match,
the app owns the error. If the returned status is a 502/503/504 with no real
upstream status, the edge manufactured it.

---

## 500 — Internal Server Error (application-layer)

**Meaning:** The backend application hit an error it is aware of and responded
with a complete, valid HTTP response whose status is 500 — typically an
unhandled exception, a failed DB call, a nil dereference, etc. The ingress did
its job perfectly: it connected, forwarded the request, received a well-formed
response, and passed it back unchanged.

**It is NOT** a proxy/connectivity problem. Restarting the ingress controller
will not help. The bug is in the app (or a dependency the app talks to).

> Caveat: the ingress controller *can* also emit its own 500 for genuine
> internal controller errors, but that is rare. In practice, 500 through an
> ingress means "the upstream returned 500."

**Confirm it:**
- `kubectl get endpoints <svc>` → not empty (backend is healthy/reachable).
- Controller access log: **upstream status == returned status == 500**. The app
  produced it.
- Look in the **application** logs (`kubectl logs deploy/<app>`) for the stack
  trace — that's where the fix lives.

---

## 503 — Service Unavailable (no backend to try)

**Meaning:** The ingress has **nowhere to send the request**. The upstream group
for that Service is empty — there are no Ready pods, hence no Endpoints, or all
upstreams are marked down. NGINX returns 503 without ever attempting a
connection because there is no address to connect to.

**Common triggers:**
- A deploy scaled to `replicas: 0`.
- All pods `NotReady` (failing readiness probes, still starting, crash-looping).
- A **Service selector mismatch** — the selector matches no pods, so the
  Endpoints object is empty.
- Wrong `ingressClassName` / no matching backend.

**Confirm it:**
- `kubectl get endpoints <svc>` → **`<none>`**. This is the definitive signal.
- `kubectl get pods` → zero Ready pods for that Service.
- Controller access log: status 503, upstream is empty (`-`).

This is the one code that means "I didn't even try, because there was no one to
try." Readiness probes are directly implicated: a pod that fails its readiness
probe is removed from Endpoints, and if that's the last pod, you get 503.

---

## 504 — Gateway Timeout (reached backend, it never answered in time)

**Meaning:** The ingress **did** reach a backend and forwarded the request, but
the backend did not respond within the configured timeout. The upstream is too
slow, hung, deadlocked, or blocked on a slow dependency.

The relevant NGINX timeouts (all default **60s** in ingress-nginx):
- `proxy-connect-timeout` — establishing the TCP connection to upstream.
- `proxy-send-timeout` — writing the request to upstream.
- `proxy-read-timeout` — waiting for the response from upstream (the usual
  culprit for slow endpoints).

**Confirm it:**
- `kubectl get endpoints <svc>` → not empty (a backend existed and was reached).
- Controller **error** log: `upstream timed out (110: Connection timed out)
  while reading response header from upstream`.
- The response comes back after roughly the timeout value, not the app's real
  latency — e.g. ~2s when `proxy-read-timeout: "2"` and the endpoint sleeps 10s.

**Fix direction:** either make the backend faster, or, if the slow response is
legitimate (long report, big upload), raise the timeout:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"
```

---

## 502 — Bad Gateway (reached backend, response was unusable)

**Meaning:** The ingress reached a backend and got **something**, but the
response was invalid or the connection failed in a way that isn't a clean
timeout. NGINX cannot produce a valid response to the client, so it returns 502.

**Common triggers:**
- **Connection refused / reset** — the target port has nothing listening (wrong
  `targetPort`, container not up yet, app bound to `127.0.0.1` instead of
  `0.0.0.0`).
- Backend **crashed mid-response** or closed the connection early.
- **Malformed HTTP** from the backend (bad headers, non-HTTP on the port).
- Upstream TLS handshake failure (when talking HTTPS to the backend).
- During a rolling deploy: NGINX reuses a keep-alive connection to a pod that is
  shutting down and gets a reset (see `deeper.md`).

**Confirm it:**
- `kubectl get endpoints <svc>` → not empty. This is what separates 502 from
  503: there *is* an endpoint to try; connecting to it or reading from it fails.
- Controller **error** log: `connect() failed (111: Connection refused) while
  connecting to upstream`, or `upstream prematurely closed connection`, or
  `no live upstreams`.
- Check the app: `kubectl logs deploy/<app>`, `kubectl get pods` for restarts,
  and verify the Service `targetPort` matches the container's listening port.

---

## Differential-diagnosis table

| You see | Most likely cause | First command to confirm | Tell-tale sign |
|---------|-------------------|--------------------------|----------------|
| **500** | App threw an error; ingress proxied it | `kubectl logs deploy/<app>` | Endpoints present; **upstream status == 500** in access log |
| **503** | No Ready pods / empty Endpoints / selector mismatch | `kubectl get endpoints <svc>` | Endpoints column is **`<none>`** |
| **504** | Backend reached but too slow / hung | `kubectl logs -n ingress-nginx <ctrl>` | error log: `upstream timed out`; response returns at ~timeout value |
| **502** | Connection refused/reset, crash, malformed response, wrong targetPort | `kubectl get endpoints <svc>` + controller error log | Endpoints present; error log: `connection refused` / `prematurely closed` |

## The one-question triage flow

1. **Is `kubectl get endpoints <svc>` empty?**
   - Yes → **503** (fix readiness / scaling / selector).
   - No → a backend exists, so it's 500, 502, or 504.
2. **Does the controller error log show a timeout?** → **504** (fix latency or
   raise `proxy-read-timeout`).
3. **Does it show connection refused / reset / prematurely closed?** → **502**
   (fix targetPort / crash / graceful shutdown).
4. **Neither, and upstream status in the access log is 500?** → **500** (fix the
   app; read app logs).

## Reading the logs

```bash
CTRL=$(kubectl get pods -n ingress-nginx \
  -l app.kubernetes.io/component=controller \
  -o jsonpath='{.items[0].metadata.name}')

kubectl logs -n ingress-nginx $CTRL            # access + error log combined
kubectl logs -n ingress-nginx -f $CTRL         # follow live while reproducing
```

The default ingress-nginx access log line includes both the **status** returned
to the client and the **upstream_status** received from the backend
(`$upstream_status`). Comparing those two numbers is the single most useful
habit for triaging 5xx: equal → app owns it; divergent (or upstream `-`) → the
edge generated it.

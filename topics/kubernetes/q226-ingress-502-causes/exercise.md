# Exercise — Reproduce and fix 502s from ingress-nginx

This is a runnable lab. You will stand up a local `kind` cluster with the
`ingress-nginx` controller, get a baseline 200, then **reproduce a 502 two
different ways** (wrong `targetPort`, and pod termination during a rollout) and
fix each. Two more causes (keepalive mismatch, controller starvation) are
explained with the exact knobs to inspect.

## Prerequisites

- `docker` (or podman) running
- `kind` (https://kind.sigs.k8s.io/)
- `kubectl`
- `curl`

All images used are small and public: `hashicorp/http-echo`,
`traefik/whoami`, and the ingress-nginx controller image.

---

## Step 0 — Create the cluster + install ingress-nginx

`kind` needs `extraPortMappings` so the ingress controller's NodePort is
reachable from your host on `localhost:80`. The `node-labels` +
`ingress-ready` label is what the ingress-nginx "kind" provider manifest
selects on.

Create `kind-ingress.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
```

Create the cluster and install the controller (the `kind` provider manifest):

```bash
kind create cluster --name ing502 --config kind-ingress.yaml

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Wait for the controller to be ready before continuing.
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

Handy shell variable for the controller pod (used throughout):

```bash
CTRL="$(kubectl get pod -n ingress-nginx \
  -l app.kubernetes.io/component=controller \
  -o jsonpath='{.items[0].metadata.name}')"
echo "$CTRL"
```

---

## Step 1 — Baseline: deploy an app and get a 200

`whoami` (traefik/whoami) listens on port **80** and prints request info.

Create `app.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
spec:
  replicas: 2
  selector:
    matchLabels: { app: whoami }
  template:
    metadata:
      labels: { app: whoami }
    spec:
      containers:
        - name: whoami
          image: traefik/whoami:v1.10.0
          ports:
            - containerPort: 80          # the port the container actually listens on
          readinessProbe:
            httpGet: { path: /, port: 80 }
            periodSeconds: 2
---
apiVersion: v1
kind: Service
metadata:
  name: whoami
spec:
  selector: { app: whoami }
  ports:
    - port: 80
      targetPort: 80                     # <-- CORRECT: matches containerPort 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: demo.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: whoami
                port:
                  number: 80
```

Apply and test. We send `Host: demo.local` so the ingress routes it:

```bash
kubectl apply -f app.yaml
kubectl rollout status deploy/whoami

curl -s -o /dev/null -w "%{http_code}\n" -H "Host: demo.local" http://localhost/
# Expect: 200
```

---

## Step 2 — Reproduce a 502 with a wrong `targetPort` (connection refused)

Point the Service `targetPort` at a port **nothing is listening on** (9999).
The endpoints will still be "ready" (readiness probe hits :80, which works),
but nginx will proxy to pod-IP:9999 and get a TCP `connection refused`.

```bash
kubectl patch svc whoami --type merge -p '{"spec":{"ports":[{"port":80,"targetPort":9999}]}}'

# Give endpoints a moment to update, then hit it:
sleep 3
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: demo.local" http://localhost/
# Expect: 502
```

Now look at the controller log — this is the signature you must recognize:

```bash
kubectl logs -n ingress-nginx "$CTRL" --tail=20
# Look for a line like:
#   connect() failed (111: Connection refused) while connecting to upstream,
#   ... upstream: "http://10.244.0.x:9999/" ...
```

`111` is the Linux `ECONNREFUSED` errno. nginx reached the pod IP but nothing
was listening on that port → 502.

**Fix** — put `targetPort` back to the real container port:

```bash
kubectl patch svc whoami --type merge -p '{"spec":{"ports":[{"port":80,"targetPort":80}]}}'
sleep 3
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: demo.local" http://localhost/
# Expect: 200
```

> Contrast with 503: if you instead scaled the Deployment to **0** replicas,
> the ingress would have **no endpoints at all** and return **503**, not 502.
> Try it if you want to see the difference:
> `kubectl scale deploy/whoami --replicas=0` → curl → `503`, then
> `kubectl scale deploy/whoami --replicas=2` → `200`.

---

## Step 3 — The money demo: 502s during a rollout (pod dying mid-request)

This reproduces the classic "502s during every deploy." We deploy an app that
terminates **immediately** on SIGTERM with **no `preStop` and a short grace
period**, then hammer it with a curl loop while rolling the Deployment. Because
the pod dies before nginx stops sending it traffic (endpoint removal is
asynchronous), some in-flight connections are cut → 502.

### 3a — Deploy the "ungraceful" version

`http-echo` serves a fixed body on `-listen`. We give it a very short grace
period and no `preStop`, so it disappears the instant it is told to stop.

Create `echo-ungraceful.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
spec:
  replicas: 4
  selector:
    matchLabels: { app: echo }
  template:
    metadata:
      labels: { app: echo }
    spec:
      terminationGracePeriodSeconds: 1        # dies almost instantly on SIGTERM
      containers:
        - name: echo
          image: hashicorp/http-echo:1.0
          args: ["-listen=:5678", "-text=hello"]
          ports:
            - containerPort: 5678
          readinessProbe:
            httpGet: { path: /, port: 5678 }
            periodSeconds: 2
---
apiVersion: v1
kind: Service
metadata:
  name: echo
spec:
  selector: { app: echo }
  ports:
    - port: 80
      targetPort: 5678
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo
spec:
  ingressClassName: nginx
  rules:
    - host: echo.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: echo
                port:
                  number: 80
```

```bash
kubectl apply -f echo-ungraceful.yaml
kubectl rollout status deploy/echo

# Sanity check:
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: echo.local" http://localhost/   # 200
```

### 3b — Load loop in one terminal, rollout in another

**Terminal A** — a tight loop that prints only non-200 responses and a running
count. Leave it running:

```bash
n=0; bad=0
while true; do
  n=$((n+1))
  code=$(curl -s -o /dev/null -w "%{http_code}" -H "Host: echo.local" http://localhost/)
  if [ "$code" != "200" ]; then bad=$((bad+1)); echo "$(date +%T)  HTTP $code   (bad=$bad / total=$n)"; fi
done
```

**Terminal B** — force several rollouts (each restarts all pods):

```bash
for i in 1 2 3; do
  kubectl rollout restart deploy/echo
  kubectl rollout status deploy/echo
done
```

**Expected:** Terminal A prints intermittent `HTTP 502` lines during each
rollout. (You may occasionally see 504 too; the dominant symptom here is 502
from connections cut mid-flight.) Confirm the controller log signature:

```bash
kubectl logs -n ingress-nginx "$CTRL" --tail=50 | grep -i "upstream"
# Look for:
#   upstream prematurely closed connection while reading response header from upstream
#   -- or --
#   recv() failed (104: Connection reset by peer) while reading response header from upstream
```

`104` is `ECONNRESET`: the pod's TCP stack went away mid-response. "upstream
prematurely closed connection" is nginx saying the backend closed the socket
before finishing the HTTP response. **This is the deploy-502 fingerprint.**

Stop Terminal A with Ctrl-C for now (note the `bad` count).

### 3c — Fix: graceful shutdown (preStop sleep + grace period)

The fix is to keep the pod alive and serving **after** it receives the signal
to terminate, long enough for its endpoint removal to propagate to nginx and
for in-flight requests to finish. The standard pattern is a `preStop` hook that
sleeps, plus a `terminationGracePeriodSeconds` comfortably larger than the
sleep.

> Why a plain `sleep`? On SIGTERM, Kubernetes runs `preStop` **before** sending
> SIGTERM to the container, but crucially the pod is marked `Terminating` and
> removed from Endpoints **at the same time** the shutdown begins. The
> `preStop` sleep buys time for the Endpoints/EndpointSlice update to reach the
> ingress controller so it stops routing new traffic, while the container keeps
> serving existing connections. `http-echo` has no `sh`, so we use a version
> that does — swap to an image with a shell, or use the `whoami` app which we
> can wrap. Here we switch the echo container to `busybox`-style httpd is
> overkill; simplest is to add the hook to the `whoami` app which ships a
> shell. We do that below.

Create `echo-graceful.yaml` using `traefik/whoami` (it responds on :80 and the
image has `/bin/sh` for the preStop hook):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
spec:
  replicas: 4
  selector:
    matchLabels: { app: echo }
  template:
    metadata:
      labels: { app: echo }
    spec:
      terminationGracePeriodSeconds: 30        # must exceed the preStop sleep
      containers:
        - name: echo
          image: traefik/whoami:v1.10.0
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet: { path: /, port: 80 }
            periodSeconds: 2
          lifecycle:
            preStop:
              exec:
                # Keep serving while endpoint removal propagates to nginx.
                command: ["/bin/sh", "-c", "sleep 10"]
---
apiVersion: v1
kind: Service
metadata:
  name: echo
spec:
  selector: { app: echo }
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f echo-graceful.yaml
kubectl rollout status deploy/echo
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: echo.local" http://localhost/   # 200
```

Re-run the load loop (Terminal A) and the rollouts (Terminal B) exactly as in
3b. **Expected:** the `bad` counter stays at (or very near) 0 through all three
rollouts. The pods now drain gracefully: they leave the Endpoints set first,
nginx stops sending them new requests, and in-flight requests complete before
the container exits.

That before/after — 502s during rollout with a 1s grace + no hook, ~0 with a
`preStop` sleep + 30s grace — is the whole point of the exercise.

> Also relevant, though we do not force it here: adding a
> `rollingUpdate` strategy with `maxUnavailable: 0` and a small `maxSurge`
> ensures new ready pods exist before old ones go away, further shrinking the
> window.

---

## Step 4 — (Explained) keepalive mismatch → 502

nginx-ingress keeps a pool of **idle keepalive** TCP connections open to your
upstream pods to avoid reconnecting per request. If your **backend closes idle
connections faster than nginx expects**, nginx can pick a connection from its
pool that the backend has *already* closed, send a request on it, and the
request dies → 502 with `upstream prematurely closed connection`.

This is the same class of bug as the well-known AWS ALB + backend
`idle-timeout` ordering problem: **the backend's idle/keepalive timeout must be
longer than the proxy's keepalive timeout**, so the proxy is always the one to
decide a connection is stale.

How to reproduce deliberately: run an app server with a very short keepalive
(e.g. a Node/Express server with `server.keepAliveTimeout = 1000`, or gunicorn
with a low `--keep-alive`) behind the ingress, and drive low, sporadic traffic
so connections sit idle between requests. You will see intermittent 502s with
the "prematurely closed" signature and no correlation to deploys.

The knobs (nginx-ingress):

```bash
# ConfigMap of the controller (name/namespace per your install):
#   upstream-keepalive-timeout        (default 60s) — how long nginx keeps idle upstream conns
#   upstream-keepalive-requests       — max requests per kept-alive conn
#   upstream-keepalive-connections    — pool size
#
# Per-Ingress annotation to make nginx retry on a broken upstream connection:
#   nginx.ingress.kubernetes.io/proxy-next-upstream: "error timeout http_502"
#   nginx.ingress.kubernetes.io/proxy-next-upstream-tries: "3"
```

**Fix rule:** set the **backend keepAlive/idle timeout GREATER than**
`upstream-keepalive-timeout` (or lower nginx's). `proxy-next-upstream` can mask
the occasional race by retrying on another connection — but only enable
retries for **idempotent** requests, because a retried non-idempotent POST can
be applied twice (see `deeper.md`).

Related "talked to a backend but the reply was unusable" 502 causes worth
naming:

- **Wrong protocol:** backend expects **HTTPS** but the ingress speaks **HTTP**
  to it (or vice versa). Fix with
  `nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"`.
- **Response header too large:** upstream sends headers bigger than nginx's
  proxy buffers → 502. Tune `proxy-buffer-size` (annotation/ConfigMap).
- **Backend returns non-HTTP / garbage** on the port (e.g. you pointed the
  Service at a gRPC or TLS port while nginx speaks plain HTTP).

---

## Step 5 — (Explained / optional demo) controller starvation → 502

If the **ingress-nginx controller pod itself** is CPU-throttled, memory-starved,
OOMKilled, or hitting worker-connection / file-descriptor limits, its worker
processes can be killed and restarted, dropping connections mid-flight → 502s
that correlate with controller restarts, not with your app.

Check the controller's health first whenever 502s look random:

```bash
kubectl get pod -n ingress-nginx -l app.kubernetes.io/component=controller   # RESTARTS column
kubectl describe pod -n ingress-nginx "$CTRL" | grep -iA3 -e "Last State" -e "OOMKilled"
kubectl top pod -n ingress-nginx "$CTRL"                                     # needs metrics-server
```

Optional demo — impose an absurdly tiny memory limit and watch it get
OOMKilled and restart (this degrades the controller; do it last):

```bash
kubectl -n ingress-nginx set resources deploy/ingress-nginx-controller \
  --limits=memory=40Mi --requests=memory=40Mi
# Watch it churn:
kubectl get pod -n ingress-nginx -w    # expect CrashLoopBackOff / OOMKilled restarts
# Undo:
kubectl -n ingress-nginx set resources deploy/ingress-nginx-controller \
  --limits=memory=- --requests=memory=-
```

**Fix:** give the controller adequate CPU/memory requests+limits, monitor its
restarts, and raise `max-worker-connections` / file-descriptor limits under
high connection churn.

---

## Cleanup

```bash
kind delete cluster --name ing502
rm -f kind-ingress.yaml app.yaml echo-ungraceful.yaml echo-graceful.yaml
```

---

## What you demonstrated

| Step | Cause | Signature in controller log | Fix |
|------|-------|------------------------------|-----|
| 2 | Wrong `targetPort` / nothing listening | `connect() failed (111: Connection refused)` | Correct `targetPort` to the real container port |
| 3 | Pod dies mid-request during rollout | `upstream prematurely closed connection` / `recv() failed (104: Connection reset by peer)` | `preStop` sleep + `terminationGracePeriodSeconds` + readiness; `maxUnavailable: 0` |
| 4 | Keepalive mismatch | `upstream prematurely closed connection` (no deploy correlation) | backend idle timeout > nginx `upstream-keepalive-timeout`; `proxy-next-upstream` for idempotent reqs |
| 5 | Controller starved / OOMKilled | controller RESTARTS, `OOMKilled` in describe | resource requests/limits, fd/worker-connection tuning |

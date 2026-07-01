# q210 — Hands-on Lab: Reproduce the "Ready but broken" rollout, then fix it

## Goal

Reproduce the exact bug — pods go **Ready** and receive traffic while the app
cannot reach the database and returns HTTP 500 — then fix it by turning the
**shallow** readiness probe into a **deep** one so the bad rollout halts by
itself and the old pods keep serving.

Everything here is runnable with a local `kind` cluster, `kubectl`, and
`docker`. No custom image is built: the tiny app is Python stdlib code injected
via a ConfigMap and run on the stock `python:3.12-slim` image. The database is
stock `postgres:16-alpine`.

> The app breaks by pointing at a **wrong DB host** (a config change shipped by
> the "new version"). A wrong host/port is caught by a TCP-level deep check. A
> wrong Secret (rotated password) is the same class of bug — the deep check
> would just need to run a real authenticated query instead of a bare TCP
> connect. We use the host break because it is 100% runnable with the stdlib.

## Prerequisites

```bash
kind version        # https://kind.sigs.k8s.io/  (needs docker running)
kubectl version --client
docker info >/dev/null && echo "docker ok"
```

---

## Step 0 — Create the cluster

```bash
kind create cluster --name q210
kubectl config use-context kind-q210
kubectl get nodes
```

---

## Step 1 — Deploy PostgreSQL (with a working Secret)

```bash
cat <<'YAML' | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  POSTGRES_PASSWORD: "s3cret-pw"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels: { app: postgres }
  template:
    metadata:
      labels: { app: postgres }
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_DB
              value: appdb
          # A DB doing readiness RIGHT: pg_isready actually asks Postgres if it
          # can accept connections. This is a *local* deep check, which is fine.
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "postgres", "-d", "appdb"]
            initialDelaySeconds: 5
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector: { app: postgres }
  ports:
    - port: 5432
      targetPort: 5432
YAML

kubectl rollout status deploy/postgres --timeout=120s
```

---

## Step 2 — Deploy the app (v1, correct config, SHALLOW readiness)

The app source lives in a ConfigMap. Note the three request paths:

- `/healthz` → always 200 (process is up). Used by **liveness** and, in v1, by
  the **shallow readiness** probe.
- `/readyz`  → 200 only if it can open a TCP connection to the DB (**deep**).
- `/`        → the real request path: 200 if the DB is reachable, else **500**.

```bash
cat <<'YAML' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-src
data:
  app.py: |
    import os, socket
    from http.server import BaseHTTPRequestHandler, HTTPServer

    DB_HOST = os.environ.get("DB_HOST", "postgres")
    DB_PORT = int(os.environ.get("DB_PORT", "5432"))

    def db_ok():
        try:
            with socket.create_connection((DB_HOST, DB_PORT), timeout=1):
                return True
        except OSError:
            return False

    class H(BaseHTTPRequestHandler):
        def _send(self, code, body):
            self.send_response(code)
            self.send_header("Content-Type", "text/plain")
            self.end_headers()
            self.wfile.write(body.encode())

        def do_GET(self):
            if self.path == "/healthz":
                # SHALLOW: the process is alive. No dependency is checked.
                self._send(200, "ok\n")
            elif self.path == "/readyz":
                # DEEP: only ready if we can actually reach the database.
                if db_ok():
                    self._send(200, "ready\n")
                else:
                    self._send(503, "NOT READY: cannot reach %s:%d\n" % (DB_HOST, DB_PORT))
            else:
                # Real traffic path exercises the DB dependency.
                if db_ok():
                    self._send(200, "served (db reachable)\n")
                else:
                    self._send(500, "ERROR: cannot reach database %s:%d\n" % (DB_HOST, DB_PORT))

        def log_message(self, *a):
            pass

    print("listening on :8080, DB_HOST=%s DB_PORT=%d" % (DB_HOST, DB_PORT), flush=True)
    HTTPServer(("0.0.0.0", 8080), H).serve_forever()
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels: { app: web }
spec:
  replicas: 3
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
        - name: web
          image: python:3.12-slim
          command: ["python", "/app/app.py"]
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              value: "postgres"          # <-- correct host (v1)
            - name: DB_PORT
              value: "5432"
          volumeMounts:
            - name: src
              mountPath: /app
          # liveness must NEVER check the DB (see solution.md). Shallow only.
          livenessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 5
            periodSeconds: 10
          # v1 BUG: readiness is SHALLOW. It never touches the DB.
          readinessProbe:
            httpGet: { path: /healthz, port: 8080 }
            initialDelaySeconds: 3
            periodSeconds: 5
      volumes:
        - name: src
          configMap:
            name: app-src
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector: { app: web }
  ports:
    - port: 80
      targetPort: 8080
YAML

kubectl rollout status deploy/web --timeout=120s
kubectl get pods -l app=web -o wide
```

Confirm the healthy baseline. In a second terminal:

```bash
kubectl port-forward svc/web 8080:80
# then, in another terminal:
curl -s localhost:8080/          # -> "served (db reachable)"
curl -s -o /dev/null -w "%{http_code}\n" localhost:8080/    # -> 200
```

Stop the port-forward (Ctrl-C) before the next step.

---

## Step 3 — Ship the "new version" that breaks the config (reproduce the bug)

The new release accidentally sets a wrong DB host (typo, wrong service name, or
a bad value templated in from CI). Everything else is unchanged, and readiness
is **still shallow**.

```bash
kubectl set env deploy/web DB_HOST=postgres-typo
kubectl rollout status deploy/web --timeout=120s
```

**Observe the failure that Kubernetes did not catch:**

```bash
# All pods are 1/1 READY even though they cannot reach the DB:
kubectl get pods -l app=web

# The Service endpoints list the broken pods -> they receive live traffic:
kubectl get endpoints web

# But the app is down. Port-forward and curl the real path:
kubectl port-forward svc/web 8080:80 &
sleep 2
curl -s localhost:8080/healthz     # 200  <- shallow probe sees this, says Ready
curl -s localhost:8080/            # "ERROR: cannot reach database postgres-typo:5432"
curl -s -o /dev/null -w "%{http_code}\n" localhost:8080/   # -> 500
kill %1

# The logs confirm the DB host is wrong:
kubectl logs -l app=web --tail=5
```

**This is the bug reproduced:** rollout succeeded, pods are Ready, endpoints are
populated, real traffic gets HTTP 500. The readiness probe only proved the
process answers `/healthz`; it never proved the pod could actually serve.

---

## Step 4 — Fix: make readiness DEEP, then watch the bad rollout halt itself

### 4a. Restore good config AND switch readiness to the deep `/readyz`

We also add a `startupProbe` (correct pattern for apps that may boot slowly) and
keep **liveness shallow** on `/healthz` so a DB blip can never trigger restarts.

```bash
kubectl set env deploy/web DB_HOST=postgres      # good host again

kubectl patch deploy/web --type merge -p '{
  "spec": { "template": { "spec": { "containers": [ {
    "name": "web",
    "readinessProbe": {
      "httpGet": { "path": "/readyz", "port": 8080 },
      "initialDelaySeconds": 3,
      "periodSeconds": 5,
      "timeoutSeconds": 2,
      "failureThreshold": 3
    },
    "startupProbe": {
      "httpGet": { "path": "/healthz", "port": 8080 },
      "periodSeconds": 3,
      "failureThreshold": 20
    }
  } ] } } } }'

kubectl rollout status deploy/web --timeout=120s
kubectl get pods -l app=web        # 3/3 Ready, now backed by a REAL DB check
```

### 4b. Ship the broken config again — this time the deep probe stops it

```bash
kubectl set env deploy/web DB_HOST=postgres-typo
```

Now watch the rollout **fail to progress** instead of silently succeeding:

```bash
# Watch new pods come up but NEVER reach Ready (Postgres check fails):
kubectl get pods -l app=web -w
# (Ctrl-C after ~30s. You will see new pods stuck at 0/1 READY.)

# The rollout does NOT complete — it times out because new pods aren't Ready:
kubectl rollout status deploy/web --timeout=60s ; echo "exit=$?"   # non-zero

# Because maxUnavailable keeps old pods up until new ones are Ready,
# the OLD good pods are still serving. Endpoints still point to healthy pods:
kubectl get endpoints web
kubectl get pods -l app=web

# Prove the service is still up despite the bad deploy:
kubectl port-forward svc/web 8080:80 &
sleep 2
curl -s -o /dev/null -w "%{http_code}\n" localhost:8080/    # still 200
kill %1

# Inspect WHY the new pods won't become Ready:
kubectl describe pod -l app=web | grep -A3 -i readiness
```

You have now shown the fix: a bad config change can no longer take the service
down silently. The new pods never become Ready, so they never enter the Service
endpoints, the rolling update stalls, and `kubectl rollout status` returns
non-zero — which is exactly the signal a CI/CD pipeline should gate on to
trigger an **automatic rollback**.

### 4c. Roll back (what CI/CD should do automatically)

```bash
kubectl rollout undo deploy/web
kubectl rollout status deploy/web --timeout=120s
kubectl get pods -l app=web        # back to 3/3 Ready, serving
```

---

## Step 5 — Cleanup

```bash
kind delete cluster --name q210
```

---

## What to take away from the lab

- With a **shallow** readiness probe, Kubernetes marked broken pods Ready and
  routed traffic to them → silent outage that the rollout reported as success.
- With a **deep** readiness probe, the same broken config produced **0/1 Ready**
  pods, a **stalled** rollout, and a **non-zero** `kubectl rollout status` —
  while `maxUnavailable`/`maxSurge` kept the old pods serving.
- **Liveness stayed shallow** the entire time, so no pod was ever restarted for
  a dependency problem (no crash loop).
- The remaining question — *should* a deep DB check live in readiness at all,
  given that a real DB blip would then flip every pod NotReady at once? — is the
  nuance covered in `solution.md` and `deeper.md`.

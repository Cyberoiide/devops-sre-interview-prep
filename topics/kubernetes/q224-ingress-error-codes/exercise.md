# Exercise — Force every Ingress 5xx code on purpose

This is a fully runnable lab. You will stand up a local `kind` cluster with
`ingress-nginx`, deploy an app, then deliberately trigger **503**, **504**,
**502**, and **500**, confirming each from endpoints and controller logs.

## Prerequisites

- `docker` running
- `kind` (https://kind.sigs.k8s.io/)
- `kubectl`
- `curl`

All the app images used are small and public:
`kennethreitz/httpbin` (has `/status/500` and `/delay/10`) and
`traefik/whoami`.

---

## Step 0 — Create a kind cluster with ingress support + install ingress-nginx

`kind` needs port mappings so the ingress controller's ports are reachable from
your host, and the control-plane node must carry the
`ingress-ready=true` label the kind manifest schedules against.

Save as `kind-ingress.yaml`:

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

Create the cluster and install the controller (the `kind` provider manifest is
purpose-built for this setup):

```bash
kind create cluster --name ingress-lab --config kind-ingress.yaml

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Wait for the controller to be ready before continuing
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

Handy variables used throughout:

```bash
# The controller pod, for reading logs
CTRL=$(kubectl get pods -n ingress-nginx \
  -l app.kubernetes.io/component=controller \
  -o jsonpath='{.items[0].metadata.name}')

# Tail logs in a second terminal while you run the curls:
#   kubectl logs -n ingress-nginx -f $CTRL
```

We will send every request to host `demo.local`. Because kind maps host port 80
to the controller, we can hit `localhost` and just set the `Host` header — no
`/etc/hosts` edit needed:

```bash
curl -i -H "Host: demo.local" http://localhost/
```

---

## Step 1 — A normal app that returns 200

Save as `01-app.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
        - name: whoami
          image: traefik/whoami
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet: { path: /, port: 80 }
            initialDelaySeconds: 1
            periodSeconds: 2
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector: { app: web }
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
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
                name: web
                port:
                  number: 80
```

```bash
kubectl apply -f 01-app.yaml
kubectl wait --for=condition=available deploy/web --timeout=60s

curl -i -H "Host: demo.local" http://localhost/
# Expect: HTTP/1.1 200 OK  (traefik/whoami echoes request info)

kubectl get endpoints web
# Expect: one ADDRESS:PORT listed — a healthy backend exists.
```

---

## Step 2 — Force **503** (no backend to send to)

Scale to zero. There are now no Ready pods, so the Service has **no
endpoints**, so the ingress has nowhere to route.

```bash
kubectl scale deploy/web --replicas=0
kubectl wait --for=delete pod -l app=web --timeout=60s 2>/dev/null || sleep 5

curl -i -H "Host: demo.local" http://localhost/
# Expect: HTTP/1.1 503 Service Temporarily Unavailable

kubectl get endpoints web
# Expect: ENDPOINTS column is <none> — the smoking gun for 503.

kubectl logs -n ingress-nginx $CTRL | tail -n 5
# The access log line shows status 503 with an empty/"-" upstream.
```

> A broken Service **selector** produces the exact same symptom (zero
> endpoints). Try it: `kubectl patch svc web -p '{"spec":{"selector":{"app":"nope"}}}'`
> then curl again → 503, `get endpoints` empty. Restore with
> `kubectl patch svc web -p '{"spec":{"selector":{"app":"web"}}}'`.

Scale back up before continuing:

```bash
kubectl scale deploy/web --replicas=1
kubectl wait --for=condition=available deploy/web --timeout=60s
```

---

## Step 3 — Force **504** (reached backend, it never answered in time)

Deploy `httpbin`, whose `/delay/N` endpoint sleeps N seconds before responding.
Set a deliberately short `proxy-read-timeout` of **2s** on its Ingress, then hit
`/delay/10`. The controller connects fine, forwards the request, waits 2s, gives
up → 504.

Save as `03-slow.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: slow
spec:
  replicas: 1
  selector:
    matchLabels: { app: slow }
  template:
    metadata:
      labels: { app: slow }
    spec:
      containers:
        - name: httpbin
          image: kennethreitz/httpbin
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: slow
spec:
  selector: { app: slow }
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: slow
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "2"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "2"
spec:
  ingressClassName: nginx
  rules:
    - host: slow.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: slow
                port:
                  number: 80
```

```bash
kubectl apply -f 03-slow.yaml
kubectl wait --for=condition=available deploy/slow --timeout=90s

# Fast endpoint works fine:
curl -i -H "Host: slow.local" http://localhost/get
# Expect: 200

# Slow endpoint exceeds the 2s read timeout:
curl -i -H "Host: slow.local" http://localhost/delay/10
# Expect: HTTP/1.1 504 Gateway Time-out (after ~2s, not 10s)

kubectl get endpoints slow            # NOT empty — a backend exists and was reached
kubectl logs -n ingress-nginx $CTRL | grep 504 | tail -n 3
# error.log also shows: upstream timed out (110: Connection timed out) while reading response
```

The key contrast with 503: `get endpoints slow` is **not empty**. The backend
exists and was reached; it was just too slow.

---

## Step 4 — Force **502** (reached backend, response was unusable)

Point the Service at a `targetPort` where **nothing is listening**. The
controller resolves an endpoint IP and tries to connect to that port; the
connection is refused / reset, so NGINX has no valid upstream response → 502.

Save as `04-bad.yaml` (note `targetPort: 9999`, which the container does not
listen on):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bad
spec:
  replicas: 1
  selector:
    matchLabels: { app: bad }
  template:
    metadata:
      labels: { app: bad }
    spec:
      containers:
        - name: whoami
          image: traefik/whoami
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: bad
spec:
  selector: { app: bad }
  ports:
    - port: 80
      targetPort: 9999   # nothing listens here → connection refused → 502
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bad
spec:
  ingressClassName: nginx
  rules:
    - host: bad.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: bad
                port:
                  number: 80
```

```bash
kubectl apply -f 04-bad.yaml
kubectl wait --for=condition=available deploy/bad --timeout=60s

curl -i -H "Host: bad.local" http://localhost/
# Expect: HTTP/1.1 502 Bad Gateway

kubectl get endpoints bad
# NOT empty — this is what distinguishes 502 from 503:
# the ingress HAS an endpoint to try, the connection to it just fails.

kubectl logs -n ingress-nginx $CTRL | grep 502 | tail -n 3
# error.log shows: connect() failed (111: Connection refused) while connecting to upstream
```

---

## Step 5 — Observe **500** (the app's own error, passed through)

`httpbin`'s `/status/500` route deliberately returns a 500. The ingress reaches
a perfectly healthy backend, gets a complete, valid HTTP response — whose status
line just happens to be 500 — and passes it straight through. This is an
**application** error, not an ingress/connectivity problem.

We reuse the `slow` deployment (it's httpbin) but hit it through an Ingress with
a normal timeout so nothing at the edge interferes.

Save as `05-app500.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app500
spec:
  ingressClassName: nginx
  rules:
    - host: err.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: slow      # httpbin from Step 3
                port:
                  number: 80
```

```bash
kubectl apply -f 05-app500.yaml

curl -i -H "Host: err.local" http://localhost/status/500
# Expect: HTTP/1.1 500 INTERNAL SERVER ERROR  (body is the app's, from httpbin)

kubectl get endpoints slow
# NOT empty — backend healthy and reachable.

kubectl logs -n ingress-nginx $CTRL | grep '/status/500' | tail -n 2
# Access log shows the RETURNED status 500 and the UPSTREAM status ALSO 500 —
# i.e. the upstream itself produced the 500. Contrast with 502/503/504 where
# the returned status is generated by NGINX, not received from the app.
```

The tell: for a 500 the **upstream status equals the returned status** (the app
sent 500). For 502/503/504 the returned status is manufactured by the edge.

---

## Cleanup

```bash
kind delete cluster --name ingress-lab
```

That removes the cluster, the controller, and every workload in one shot.

# Exercise — See the L4 vs L7 difference on a real cluster

You will:

1. Create a local `kind` cluster wired so an ingress controller is reachable
   from your laptop.
2. Deploy **two different apps**, each behind its own `ClusterIP` Service.
3. Expose one the **L4 way** (`NodePort`, the local stand-in for a cloud
   `type=LoadBalancer`) and watch it forward blindly by port.
4. Install **ingress-nginx** and create **one Ingress** that host- and
   path-routes to **both** apps behind a **single** entry point (L7).
5. Count how many load balancers you would need without Ingress.
6. (Optional, read-only) Peek at the equivalent **Gateway API** objects.
7. Clean up.

## Prerequisites

```bash
# Docker (or Podman) running, plus:
kind version       # https://kind.sigs.k8s.io/  (e.g. v0.23+)
kubectl version --client
```

---

## Step 1 — Create a kind cluster with ingress-ready port mappings

`kind` runs Kubernetes nodes as containers, so there is no cloud load balancer.
To reach an ingress controller from the host we map host ports 80/443 into the
control-plane node and label that node so ingress-nginx's `kind` manifest
schedules onto it.

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

```bash
kind create cluster --name ingress-demo --config kind-ingress.yaml
kubectl cluster-info --context kind-ingress-demo
```

> If host ports 80/443 are already in use on your machine, change `hostPort` to
> e.g. `8080`/`8443` and adjust every `curl` below to that port.

---

## Step 2 — Deploy two distinct apps, each with a ClusterIP Service

We use `hashicorp/http-echo`, which just returns a fixed string, so you can
*see* which backend answered. `ClusterIP` (the default) is reachable only from
inside the cluster — no external exposure at all yet.

Save as `apps.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  replicas: 1
  selector:
    matchLabels: { app: app1 }
  template:
    metadata:
      labels: { app: app1 }
    spec:
      containers:
        - name: http-echo
          image: hashicorp/http-echo:1.0
          args: ["-text=HELLO FROM APP 1", "-listen=:5678"]
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: app1
spec:
  type: ClusterIP
  selector: { app: app1 }
  ports:
    - port: 80
      targetPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2
spec:
  replicas: 1
  selector:
    matchLabels: { app: app2 }
  template:
    metadata:
      labels: { app: app2 }
    spec:
      containers:
        - name: http-echo
          image: hashicorp/http-echo:1.0
          args: ["-text=hello from app 2", "-listen=:5678"]
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: app2
spec:
  type: ClusterIP
  selector: { app: app2 }
  ports:
    - port: 80
      targetPort: 5678
```

```bash
kubectl apply -f apps.yaml
kubectl rollout status deploy/app1
kubectl rollout status deploy/app2
kubectl get svc app1 app2      # both ClusterIP, no EXTERNAL-IP
```

---

## Step 3 — The L4 approach: NodePort forwards blindly by port

In a real cloud you would set `Service type=LoadBalancer` and the cloud would
provision **one L4 network load balancer for this single service**. `kind` has
no cloud, so we use `NodePort`, which is the same idea one layer down: a raw
`IP:port` entry point that forwards TCP with **zero knowledge of HTTP**.

```bash
# Turn app1's Service into a NodePort (L4 entry point for ONE service)
kubectl patch svc app1 -p '{"spec":{"type":"NodePort"}}'
kubectl get svc app1 -o wide
# Note the assigned port, e.g. 80:31234/TCP  -> 31234 is the NodePort
```

Reach it via the node. On `kind`, the control-plane container's IP works from
the host's Docker network; simplest is a temporary port-forward to prove the
"blind by port" behaviour:

```bash
NODEPORT=$(kubectl get svc app1 -o jsonpath='{.spec.ports[0].nodePort}')
echo "app1 NodePort = $NODEPORT"

# Port-forward the node port for a quick local test:
kubectl port-forward svc/app1 8081:80 >/tmp/pf.log 2>&1 &
sleep 2
curl -s http://localhost:8081/            # -> HELLO FROM APP 1
curl -s http://localhost:8081/anything    # -> HELLO FROM APP 1  (path ignored!)
curl -s -H "Host: whatever.example" http://localhost:8081/   # -> HELLO FROM APP 1 (host ignored!)
kill %1 2>/dev/null
```

**Observe the L4 behaviour:** every request hitting that port lands on `app1`,
regardless of the URL path or `Host` header. The L4 entry point cannot even see
them — it forwards TCP connections. To also expose `app2` this way you would
need a **second** NodePort / **second** cloud load balancer. That is the fan-out
cost: **one LB per service.**

---

## Step 4 — Install ingress-nginx (the L7 controller)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.2/deploy/static/provider/kind/deploy.yaml

# Wait for the controller to be ready:
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=180s
```

The `kind` provider manifest exposes the controller on host ports 80/443 (that
is what our `extraPortMappings` + `ingress-ready=true` label were for). The
controller itself is fronted by a single entry point — in a cloud this would be
one `Service type=LoadBalancer` (**one** cloud L4 LB) sitting in front of the
controller.

---

## Step 5 — ONE Ingress that host- and path-routes to BOTH apps

Note: before creating the Ingress, revert app1 back to `ClusterIP` (Ingress
routes to ClusterIP Services; it does not need the NodePort):

```bash
kubectl patch svc app1 -p '{"spec":{"type":"ClusterIP"}}'
```

Save as `ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    # Host-based virtual hosting: same IP, different hostnames -> different apps
    - host: foo.localhost
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app1
                port:
                  number: 80
    - host: bar.localhost
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app2
                port:
                  number: 80
    # Path-based routing on a single host: /a -> app1, /b -> app2
    - host: apps.localhost
      http:
        paths:
          - path: /a
            pathType: Prefix
            backend:
              service:
                name: app1
                port:
                  number: 80
          - path: /b
            pathType: Prefix
            backend:
              service:
                name: app2
                port:
                  number: 80
```

```bash
kubectl apply -f ingress.yaml
kubectl get ingress demo
```

### Prove L7 routing — one entry point, many backends

All of these hit the **same** IP and **same** port (localhost:80). Only the
`Host` header and path change, and the controller routes accordingly:

```bash
# Host-based virtual hosting:
curl -s -H "Host: foo.localhost" http://localhost/     # -> HELLO FROM APP 1
curl -s -H "Host: bar.localhost" http://localhost/     # -> hello from app 2

# Path-based routing on one host:
curl -s -H "Host: apps.localhost" http://localhost/a   # -> HELLO FROM APP 1
curl -s -H "Host: apps.localhost" http://localhost/b   # -> hello from app 2
```

Same L4 target, different L7 outcome. That is the whole point: the ingress
controller reads HTTP and dispatches; the L4 LB only ever moved bytes.

---

## Step 6 — Count the load balancers

```bash
kubectl get svc -A
```

- **Without Ingress:** to expose N HTTP services externally you would set
  `type=LoadBalancer` on each → **N cloud load balancers**, N external IPs, N×
  the cost.
- **With Ingress:** exactly one external entry point (the controller's
  `Service`, which is the single `type=LoadBalancer` in a cloud) fronts *all* of
  them via host/path rules. Add app3, app4, … app100 by editing one Ingress
  object — no new load balancers.

---

## Step 7 (optional, read-only) — the Gateway API equivalent

The Gateway API is the successor to Ingress. It splits responsibilities across
objects: **GatewayClass** (the controller implementation, cluster-scoped),
**Gateway** (a concrete listener/entry point, owned by infra/platform teams),
and **HTTPRoute** (routing rules, owned by app teams). This needs a
Gateway-API-capable controller installed (e.g. NGINX Gateway Fabric, Istio,
Envoy Gateway); `ingress-nginx` does **not** serve these objects. Treat this as
illustrative unless you install such a controller.

```yaml
# Install CRDs first (only if you want to actually apply this):
#   kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: demo-gateway
spec:
  gatewayClassName: nginx        # must match an installed GatewayClass
  listeners:
    - name: http
      protocol: HTTP
      port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: demo-route
spec:
  parentRefs:
    - name: demo-gateway
  hostnames:
    - "foo.localhost"
    - "bar.localhost"
  rules:
    - matches:
        - headers:
            - name: Host
              value: foo.localhost
      backendRefs:
        - name: app1
          port: 80
    - matches:
        - headers:
            - name: Host
              value: bar.localhost
      backendRefs:
        - name: app2
          port: 80
```

Same L7 idea, but role-oriented and more expressive (it also handles L4/TCP
routes via `TCPRoute`/`UDPRoute`), which is why Ingress is now effectively
feature-frozen in its favour.

---

## Step 8 — Cleanup

```bash
kill %1 2>/dev/null           # any lingering port-forward
kind delete cluster --name ingress-demo
rm -f kind-ingress.yaml apps.yaml ingress.yaml
```

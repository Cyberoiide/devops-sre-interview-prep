# Exercise — Migrate workloads onto a new node pool on `kind`

This lab simulates two node pools with node labels, pins a Deployment (with a
**PodDisruptionBudget**) to the **old** pool, adds a **DaemonSet**, and then
performs a live **cordon + drain** migration onto the **new** pool — watching
the PDB gate the evictions so the app never drops below its floor.

## Prerequisites

- `docker` running
- [`kind`](https://kind.sigs.k8s.io/) installed
- `kubectl` installed

## 0. Create a multi-node cluster (1 control-plane + 4 workers)

Four workers let us model two pools of two nodes each.

```bash
cat <<'EOF' > /tmp/kind-q206.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
  - role: worker
EOF

kind create cluster --name q206 --config /tmp/kind-q206.yaml

kubectl get nodes
```

You should see five nodes: `q206-control-plane`, `q206-worker`, `q206-worker2`,
`q206-worker3`, `q206-worker4`.

## 1. Simulate two node pools with labels

On managed platforms a node pool comes with its own node label (e.g.
`cloud.google.com/gke-nodepool`). We fake that with a plain label.

```bash
# Old pool
kubectl label node q206-worker  q206-worker2 pool=old
# New pool
kubectl label node q206-worker3 q206-worker4 pool=new

kubectl get nodes -L pool
```

Expected:

```
NAME                 STATUS   ROLES           AGE   VERSION   POOL
q206-control-plane   Ready    control-plane   2m    v1.31.x
q206-worker          Ready    <none>          2m    v1.31.x   old
q206-worker2         Ready    <none>          2m    v1.31.x   old
q206-worker3         Ready    <none>          2m    v1.31.x   new
q206-worker4         Ready    <none>          2m    v1.31.x   new
```

## 2. Deploy an app pinned to the OLD pool, with a PDB

Four nginx replicas, `nodeSelector: pool=old`, and a PDB requiring at least
**3 of 4** available — so at most **one** replica may be voluntarily disrupted
at a time.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 4
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      nodeSelector:
        pool: old
      # spread replicas across the two old nodes
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels: { app: web }
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "50m"
              memory: "32Mi"
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 3
  selector:
    matchLabels: { app: web }
EOF

kubectl rollout status deploy/web
```

Confirm all four pods are on the **old** pool:

```bash
kubectl get pods -l app=web -o wide
```

Expected (pods only on `q206-worker` / `q206-worker2`):

```
NAME                   READY   STATUS    ...   NODE           ...
web-5c9d...-2xk4l      1/1     Running   ...   q206-worker    ...
web-5c9d...-7bhtn      1/1     Running   ...   q206-worker2   ...
web-5c9d...-m9c4p      1/1     Running   ...   q206-worker    ...
web-5c9d...-qz8vd      1/1     Running   ...   q206-worker2   ...
```

Check the PDB — 4 healthy, 1 disruption allowed:

```bash
kubectl get pdb web-pdb
```

```
NAME      MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
web-pdb   3               N/A               1                     20s
```

## 3. Add a DaemonSet (so `--ignore-daemonsets` is meaningful)

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
spec:
  selector:
    matchLabels: { app: node-agent }
  template:
    metadata:
      labels: { app: node-agent }
    spec:
      containers:
        - name: pause
          image: registry.k8s.io/pause:3.9
EOF

kubectl rollout status ds/node-agent
kubectl get pods -l app=node-agent -o wide   # one per worker node
```

## 4. Cordon the old pool — no NEW pods, existing ones untouched

Cordon marks both old nodes unschedulable. Nothing moves yet.

```bash
kubectl cordon q206-worker q206-worker2
kubectl get nodes -L pool
```

Expected — old nodes now `SchedulingDisabled`:

```
NAME                 STATUS                     ROLES           ...   POOL
q206-worker          Ready,SchedulingDisabled   <none>          ...   old
q206-worker2         Ready,SchedulingDisabled   <none>          ...   old
q206-worker3         Ready                      <none>          ...   new
q206-worker4         Ready                      <none>          ...   new
```

The `web` pods are all still `Running` on the old nodes — cordon alone evicts
nothing. See the taint cordon added:

```bash
kubectl describe node q206-worker | grep -i taint
# Taints:  node.kubernetes.io/unschedulable:NoSchedule
```

## 5. Let the new pool accept the app

The Deployment is still pinned to `pool=old`. Repoint it at `pool=new` so
evicted pods have somewhere valid to land.

```bash
kubectl patch deploy web --type=merge \
  -p '{"spec":{"template":{"spec":{"nodeSelector":{"pool":"new"}}}}}'
```

> This triggers a rolling update; new-spec pods schedule onto the new pool
> immediately (the old nodes are cordoned anyway). In a real node-pool upgrade
> where the workload isn't pinned, you'd **skip this step** and just drain — the
> scheduler would place evicted pods on whatever schedulable capacity exists
> (the new pool). We pin here to make the "steering" explicit and observable.

## 6. Drain the old nodes — watch the PDB gate evictions

Open a watch in a second terminal:

```bash
kubectl get pods -l app=web -o wide -w
```

Then drain the first old node. `--ignore-daemonsets` skips `node-agent`;
`--delete-emptydir-data` would allow evicting emptyDir pods (none here, but it's
the safe habit).

```bash
kubectl drain q206-worker \
  --ignore-daemonsets \
  --delete-emptydir-data
```

Expected drain output — note it evicts through the **Eviction API** and waits:

```
node/q206-worker already cordoned
Warning: ignoring DaemonSet-managed Pods: default/node-agent-xxxxx
evicting pod default/web-5c9d...-2xk4l
evicting pod default/web-5c9d...-m9c4p
pod/web-5c9d...-2xk4l evicted
pod/web-5c9d...-m9c4p evicted
node/q206-worker drained
```

Because `minAvailable: 3`, the Eviction API only ever lets **one** `web` pod be
down at a time; drain evicts one, waits for its replacement to become Ready on
the new pool, then evicts the next. If you had set the PDB too strict you'd see
this instead (drain retrying, not proceeding):

```
error when evicting pods/"web-..." -n "default" (will retry after 5s):
  Cannot evict pod as it would violate the pod's disruption budget.
```

Drain the second old node:

```bash
kubectl drain q206-worker2 \
  --ignore-daemonsets \
  --delete-emptydir-data
```

Confirm every `web` pod now runs on the **new** pool:

```bash
kubectl get pods -l app=web -o wide
```

Expected (only `q206-worker3` / `q206-worker4`):

```
NAME                   READY   STATUS    ...   NODE           ...
web-6f7a...-4d2mn      1/1     Running   ...   q206-worker3   ...
web-6f7a...-8xk9p      1/1     Running   ...   q206-worker4   ...
web-6f7a...-j5tqr      1/1     Running   ...   q206-worker3   ...
web-6f7a...-w2vlc      1/1     Running   ...   q206-worker4   ...
```

The old nodes now run only the DaemonSet pod (and system pods):

```bash
kubectl get pods -A -o wide --field-selector spec.nodeName=q206-worker
```

## 7. On a real platform: delete the old pool

At this point the old pool is empty of application workloads. On GKE/EKS/AKS you
would now delete the node pool, e.g.:

```bash
# GKE (illustrative — do NOT run against kind)
# gcloud container node-pools delete old-pool --cluster my-cluster
```

On `kind` there's no cloud pool to delete. If instead this were **maintenance**
and you wanted the nodes back, `uncordon` makes them schedulable again:

```bash
kubectl uncordon q206-worker q206-worker2
kubectl get nodes -L pool     # old nodes back to Ready (schedulable)
```

## Teardown

```bash
kind delete cluster --name q206
rm -f /tmp/kind-q206.yaml
```

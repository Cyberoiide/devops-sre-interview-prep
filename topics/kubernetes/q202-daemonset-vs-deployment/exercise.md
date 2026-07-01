# Exercise — DaemonSet vs Deployment placement on `kind`

This lab builds a multi-node cluster and demonstrates, on a real scheduler, the
core difference: a **DaemonSet** runs exactly one pod per node and tracks the
node set automatically, while a **Deployment** scatters a fixed replica count
wherever it fits. It also shows a DaemonSet targeting a **subset** of nodes and
tolerating the **control-plane taint**.

## Prerequisites

- `docker` running
- [`kind`](https://kind.sigs.k8s.io/) installed
- `kubectl` installed

## 0. Create a multi-node cluster

One control-plane + three workers, so per-node placement is actually visible.

```bash
cat <<'EOF' > /tmp/kind-q202.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
EOF

kind create cluster --name q202 --config /tmp/kind-q202.yaml

# Confirm the cluster is up and see how nodes are named.
kubectl get nodes
```

You should see four nodes: `q202-control-plane`, `q202-worker`,
`q202-worker2`, `q202-worker3`.

> The control-plane node carries the taint
> `node-role.kubernetes.io/control-plane:NoSchedule`, so ordinary pods
> (including a plain DaemonSet without a matching toleration) will **not** land
> there. Confirm it:
>
> ```bash
> kubectl describe node q202-control-plane | grep -i taint
> # Taints:  node-role.kubernetes.io/control-plane:NoSchedule
> ```

---

## 1. DaemonSet — exactly one pod per node

Deploy a tiny agent that reads a host path (a stand-in for a log collector /
node-exporter that inspects the node). Note there is **no `replicas` field** — a
DaemonSet has none.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
spec:
  selector:
    matchLabels:
      app: node-agent
  template:
    metadata:
      labels:
        app: node-agent
    spec:
      containers:
        - name: agent
          image: busybox:1.36
          command: ["sh", "-c", "while true; do cat /host-var-log/README 2>/dev/null; sleep 3600; done"]
          volumeMounts:
            - name: varlog
              mountPath: /host-var-log
              readOnly: true
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
EOF
```

**Observe — one pod per worker node:**

```bash
kubectl rollout status daemonset/node-agent
kubectl get daemonset node-agent
kubectl get pods -l app=node-agent -o wide
```

`kubectl get daemonset` shows the count is driven by the node set, not a
replica field — and here it is **3** because the control-plane taint excludes
that node:

```
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
node-agent   3         3         3       3            3           <none>          30s
```

`get pods -o wide` proves exactly one pod on each of the three workers (and
none on the control-plane):

```
NAME               READY   STATUS    ... NODE
node-agent-4xk2p   1/1     Running   ... q202-worker
node-agent-7bqzn   1/1     Running   ... q202-worker2
node-agent-lm9td   1/1     Running   ... q202-worker3
```

> `DESIRED` equals the number of **schedulable, matching** nodes. You never
> typed "3" — the controller computed it. Add or remove a worker and this
> number tracks it automatically (demonstrated in step 3).

---

## 2. Deployment — N replicas scattered by fit

Now the contrast. A Deployment with `replicas: 5` does **not** aim for one per
node; the scheduler just places five pods wherever they fit.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 5
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: registry.k8s.io/pause:3.9
EOF
```

**Observe — uneven spread across the three workers:**

```bash
kubectl rollout status deployment/web
kubectl get pods -l app=web -o wide

# Count pods per node — they will NOT be evenly one-per-node.
kubectl get pods -l app=web -o wide --no-headers | awk '{print $7}' | sort | uniq -c
```

A typical result puts multiple replicas on some nodes and skips others — the
scheduler optimizes for fit and spread heuristics, not a per-node guarantee:

```
   2 q202-worker
   2 q202-worker2
   1 q202-worker3
```

Scale it and the count changes, but the "one per node" property never appears —
that is simply not what a Deployment does:

```bash
kubectl scale deployment/web --replicas=2
kubectl get pods -l app=web -o wide --no-headers | awk '{print $7}' | sort | uniq -c
# e.g. both land on the same node, or two different ones — no per-node rule
```

Clean up the Deployment before the next step:

```bash
kubectl delete deployment web
```

---

## 3. Target a subset of nodes — and watch a pod appear automatically

Real DaemonSets often run only on a class of nodes (GPU nodes, SSD nodes). Add
a `nodeSelector` and the count follows the **labeled** subset. Label only two
workers first.

```bash
kubectl label node q202-worker  disk=ssd
kubectl label node q202-worker2 disk=ssd

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ssd-agent
spec:
  selector:
    matchLabels:
      app: ssd-agent
  template:
    metadata:
      labels:
        app: ssd-agent
    spec:
      nodeSelector:
        disk: ssd
      containers:
        - name: agent
          image: registry.k8s.io/pause:3.9
EOF
```

**Observe — only the two labeled nodes get a pod:**

```bash
kubectl get daemonset ssd-agent
# DESIRED = 2 (only q202-worker and q202-worker2 match)
kubectl get pods -l app=ssd-agent -o wide
```

```
NAME              DESIRED   CURRENT   READY   ... NODE SELECTOR   AGE
ssd-agent         2         2         2       ... disk=ssd        20s
```

**Now label the third worker and watch the DaemonSet react on its own** — no
edit to the DaemonSet, no scaling command:

```bash
# Watch in one terminal:
kubectl get pods -l app=ssd-agent -o wide -w &

# Label the third node — this simulates a new matching node joining.
kubectl label node q202-worker3 disk=ssd

# Within a second or two a new ssd-agent pod is created on q202-worker3.
sleep 5
kubectl get daemonset ssd-agent          # DESIRED is now 3
kubectl get pods -l app=ssd-agent -o wide
kill %1 2>/dev/null                       # stop the background watch
```

```
NAME              DESIRED   CURRENT   READY   ... NODE SELECTOR   AGE
ssd-agent         3         3         3       ... disk=ssd        90s
```

This is the headline DaemonSet behavior: **the controller adds a pod the moment
a node enters the matching set** (a new node joining, or here a node gaining the
label), and removes it when the node leaves. A Deployment would need a manual
`kubectl scale` and still would not guarantee placement on that specific node.

Clean up before the last step:

```bash
kubectl delete daemonset ssd-agent
kubectl label node q202-worker  disk-
kubectl label node q202-worker2 disk-
kubectl label node q202-worker3 disk-
```

---

## 4. Run on the control-plane too — tolerate the taint

The `node-agent` from step 1 skipped the control-plane because of its taint. A
real node-level agent (metrics, logging, CNI) usually needs to run **everywhere,
including control-plane nodes**, so it carries a matching toleration. Patch it:

```bash
kubectl patch daemonset node-agent --type merge -p '
spec:
  template:
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
'
```

**Observe — the count rises to 4 and a pod now runs on the control-plane:**

```bash
kubectl rollout status daemonset/node-agent
kubectl get daemonset node-agent
kubectl get pods -l app=node-agent -o wide
```

```
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
node-agent   4         4         4       4            4           <none>          5m

NAME               READY   STATUS    ... NODE
node-agent-...     1/1     Running   ... q202-control-plane
node-agent-...     1/1     Running   ... q202-worker
node-agent-...     1/1     Running   ... q202-worker2
node-agent-...     1/1     Running   ... q202-worker3
```

Adding the toleration made the control-plane node "matching," so the controller
grew the DaemonSet to 4 automatically. `operator: Exists` with no `value`
tolerates the taint regardless of its value — the common pattern for
infrastructure DaemonSets.

---

## Teardown

```bash
kind delete cluster --name q202
rm -f /tmp/kind-q202.yaml
```

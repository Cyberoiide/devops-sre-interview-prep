# Exercise — Reproduce and fix `Pending` pods on `kind`

This lab reproduces **all three** root-cause families of a `Pending` pod on a
real cluster, shows the exact diagnostic output, and fixes each one.

## Prerequisites

- `docker` running
- [`kind`](https://kind.sigs.k8s.io/) installed
- `kubectl` installed

## 0. Create a small multi-node cluster

We use one control-plane + two workers so we can demonstrate node labels,
taints, and per-node allocatable resources realistically.

```bash
cat <<'EOF' > /tmp/kind-q221.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF

kind create cluster --name q221 --config /tmp/kind-q221.yaml

# Confirm the cluster is up and see how nodes are named.
kubectl get nodes
```

You should see three nodes: `q221-control-plane`, `q221-worker`,
`q221-worker2`.

> `kind` ships with the **`rancher.io/local-path`** provisioner and a default
> StorageClass named **`standard`**. We use that for the volume fix at the end.
> Confirm it:
>
> ```bash
> kubectl get storageclass
> # NAME                 PROVISIONER             ... DEFAULT ...
> # standard (default)   rancher.io/local-path   ... true    ...
> ```

---

## Cause 1 — No node matches placement constraints

### 1a. Impossible `nodeSelector`

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pending-nodeselector
spec:
  nodeSelector:
    disktype: nonexistent          # no node has this label
  containers:
    - name: pause
      image: registry.k8s.io/pause:3.9
EOF
```

**Diagnose:**

```bash
kubectl get pod pending-nodeselector          # STATUS = Pending
kubectl describe pod pending-nodeselector | sed -n '/Events:/,$p'
```

You will see a `FailedScheduling` event similar to:

```
Warning  FailedScheduling  ...  0/3 nodes are available:
  1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: },
  2 node(s) didn't match Pod's node affinity/selector.
  preemption: 0/3 nodes are available: ...
```

The key phrase is **"didn't match Pod's node affinity/selector"** — the label
`disktype=nonexistent` exists on no node. (Note the control-plane is also
excluded by its taint — see Cause 1b.)

Confirm no node has the label:

```bash
kubectl get nodes --show-labels | grep -i disktype   # returns nothing
```

**Fix — label a node so it matches:**

```bash
kubectl label node q221-worker disktype=nonexistent
kubectl get pod pending-nodeselector -w              # -> Running (Ctrl-C to stop)
```

(Alternatively, remove/relax the `nodeSelector`.) Clean up:

```bash
kubectl delete pod pending-nodeselector
kubectl label node q221-worker disktype-             # remove the label
```

### 1b. Untolerated taint

Taint a worker so the scheduler will not place a normal pod on it, then cordon
the other worker so **only** the tainted worker and the (already tainted)
control-plane remain as candidates. This forces the untolerated-taint failure.

```bash
# Taint one worker.
kubectl taint node q221-worker dedicated=gpu:NoSchedule

# Cordon the other worker so it is unschedulable, isolating the demo.
kubectl cordon q221-worker2

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pending-taint
spec:
  containers:
    - name: pause
      image: registry.k8s.io/pause:3.9
EOF
```

**Diagnose:**

```bash
kubectl describe pod pending-taint | sed -n '/Events:/,$p'
```

Expect:

```
Warning  FailedScheduling  ...  0/3 nodes are available:
  1 node(s) had untolerated taint {dedicated: gpu},
  1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: },
  1 node(s) were unschedulable. ...
```

`untolerated taint {dedicated: gpu}` is the tell. Inspect the taints directly:

```bash
kubectl describe node q221-worker | grep -i taint
# Taints:  dedicated=gpu:NoSchedule
```

**Fix — add a matching toleration:**

```bash
kubectl delete pod pending-taint

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pending-taint
spec:
  tolerations:
    - key: dedicated
      operator: Equal
      value: gpu
      effect: NoSchedule
  containers:
    - name: pause
      image: registry.k8s.io/pause:3.9
EOF

kubectl get pod pending-taint -w                     # -> Running
```

Clean up:

```bash
kubectl delete pod pending-taint
kubectl taint node q221-worker dedicated=gpu:NoSchedule-   # remove taint
kubectl uncordon q221-worker2
```

---

## Cause 2 — Insufficient resources

Request far more CPU than any node has allocatable. On a laptop each `kind`
node reports the host's CPU count; requesting `100` cores is guaranteed to be
unschedulable anywhere.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pending-resources
spec:
  containers:
    - name: hog
      image: registry.k8s.io/pause:3.9
      resources:
        requests:
          cpu: "100"          # 100 full cores — no node can satisfy this
          memory: "1Gi"
EOF
```

**Diagnose:**

```bash
kubectl describe pod pending-resources | sed -n '/Events:/,$p'
```

Expect:

```
Warning  FailedScheduling  ...  0/3 nodes are available:
  1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: },
  2 Insufficient cpu. ...
```

`Insufficient cpu` is the tell (you'd see `Insufficient memory` for RAM, or
`Insufficient nvidia.com/gpu` for GPUs). Check what a node can actually offer —
note **Allocatable** (schedulable) vs the raw hardware, and the already
**Allocated** requests:

```bash
kubectl describe node q221-worker | sed -n '/Allocatable:/,/Allocated resources/p'
kubectl top nodes            # needs metrics-server; may not be installed on kind
```

Scheduling compares your **requests** against `Allocatable - sum(existing
requests)`. `100` cores exceeds allocatable on every node, so it never binds.

**Fix — request a sane amount:**

```bash
kubectl delete pod pending-resources

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pending-resources
spec:
  containers:
    - name: hog
      image: registry.k8s.io/pause:3.9
      resources:
        requests:
          cpu: "100m"         # 0.1 core — easily fits
          memory: "64Mi"
EOF

kubectl get pod pending-resources -w                 # -> Running
kubectl delete pod pending-resources
```

> **Real-world note:** if a **Cluster Autoscaler** or **Karpenter** is
> configured, an unschedulable pod triggers a new node to be provisioned and
> the pod schedules once it joins. Without an autoscaler (as on `kind`), an
> over-request just stays `Pending` forever.

---

## Cause 3 — Volume problems (unbound PVC)

### 3a. PVC referencing a nonexistent StorageClass

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-nonexistent-sc
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: does-not-exist     # no such StorageClass
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: pending-volume
spec:
  containers:
    - name: app
      image: registry.k8s.io/pause:3.9
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-nonexistent-sc
EOF
```

**Diagnose:**

```bash
kubectl get pvc                                       # data-nonexistent-sc = Pending
kubectl get pod pending-volume                        # Pending
kubectl describe pod pending-volume | sed -n '/Events:/,$p'
```

The pod event reads:

```
Warning  FailedScheduling  ...  0/3 nodes are available:
  ... pod has unbound immediate PersistentVolumeClaims. ...
```

The root cause is on the **PVC**, so describe it:

```bash
kubectl describe pvc data-nonexistent-sc | sed -n '/Events:/,$p'
# Warning  ProvisioningFailed  ... storageclass.storage.k8s.io "does-not-exist" not found
```

**Fix — use the real default StorageClass (`standard` on kind):**

```bash
kubectl delete pod pending-volume
kubectl delete pvc data-nonexistent-sc

cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-ok
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: standard            # kind's local-path default SC
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: pending-volume
spec:
  containers:
    - name: app
      image: registry.k8s.io/pause:3.9
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-ok
EOF

kubectl get pvc data-ok -w            # -> Bound (Ctrl-C once bound)
kubectl get pod pending-volume -w     # -> Running
```

> **`WaitForFirstConsumer` note:** `kind`'s `standard` StorageClass uses
> `volumeBindingMode: WaitForFirstConsumer`, so the PVC intentionally stays
> `Pending` (status "waiting for first consumer to be created before
> binding") until a pod that uses it is scheduled. That is normal, not an
> error — binding and pod scheduling happen together. Contrast with `Immediate`
> binding, where an unsatisfiable PVC is `Pending` on its own and blocks the
> pod as in 3a. Confirm the mode:
>
> ```bash
> kubectl get storageclass standard -o jsonpath='{.volumeBindingMode}{"\n"}'
> # WaitForFirstConsumer
> ```

Clean up:

```bash
kubectl delete pod pending-volume
kubectl delete pvc data-ok
```

---

## Teardown

```bash
kind delete cluster --name q221
rm -f /tmp/kind-q221.yaml
```

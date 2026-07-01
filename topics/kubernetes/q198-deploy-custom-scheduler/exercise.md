# Exercise — Run a second scheduler on `kind`

This lab deploys a **second scheduler** alongside the default `kube-scheduler`,
routes a pod to it via `spec.schedulerName`, and proves two things:

1. a pod naming **`my-scheduler`** is bound by *our* scheduler (the `Scheduled`
   event's source is `my-scheduler`, not `default-scheduler`);
2. a pod naming a scheduler that **isn't running** stays `Pending` forever.

We take the **most robust approach**: run a **second instance of the upstream
`kube-scheduler`** image, configured with a `KubeSchedulerConfiguration` whose
profile advertises `schedulerName: my-scheduler`. This avoids compiling a
scheduler from source while exercising the exact same deploy/RBAC/leader-election
mechanics a real custom scheduler needs.

## Prerequisites

- `docker` running
- [`kind`](https://kind.sigs.k8s.io/) installed
- `kubectl` installed

## 0. Create a cluster

A single node is enough — we are demonstrating *who binds the pod*, not
placement math.

```bash
kind create cluster --name q198

kubectl get nodes
# NAME                 STATUS   ROLES           AGE   VERSION
# q198-control-plane   Ready    control-plane   30s   v1.31.x
```

> **Version-awareness (important):** the second scheduler **must** use a
> `kube-scheduler` image whose Kubernetes **minor version matches the cluster**.
> A mismatched scheduler may not understand the config `apiVersion` or the API
> objects. Find the cluster version:
>
> ```bash
> kubectl version
> # Server Version: v1.31.x   <-- match the scheduler image to this minor
> ```

Capture the **exact** image the running control-plane scheduler uses. Note the
default `kube-scheduler` is a **static pod**, so you find its image via the pod
(labelled `component=kube-scheduler`), not a Deployment:

```bash
SCHED_IMAGE=$(kubectl get pod -n kube-system -l component=kube-scheduler \
  -o jsonpath='{.items[0].spec.containers[0].image}')
echo "$SCHED_IMAGE"
# registry.k8s.io/kube-scheduler:v1.31.x
```

We reuse this same image for the second scheduler, guaranteeing a version match.

---

## 1. ServiceAccount + RBAC (in `kube-system`)

The scheduler is an ordinary workload, so it needs an identity and permissions:
read/bind pods, read nodes, create events, and use a **leader-election Lease**.
The simplest correct approach is to **bind the two built-in scheduler
ClusterRoles** (`system:kube-scheduler` and `system:volume-scheduler`) to our
ServiceAccount, plus a small namespaced `Role` for **our own** lease (the
built-in role's lease permission is scoped to the lease named `kube-scheduler`,
so our distinct lease `my-scheduler` needs its own grant).

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-scheduler
  namespace: kube-system
---
# Reuse the built-in scheduler permissions (read/bind pods, read nodes, events).
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: my-scheduler-as-kube-scheduler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-scheduler
subjects:
  - kind: ServiceAccount
    name: my-scheduler
    namespace: kube-system
---
# Volume scheduling permissions (PV/PVC/StorageClass/CSI aware binding).
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: my-scheduler-as-volume-scheduler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:volume-scheduler
subjects:
  - kind: ServiceAccount
    name: my-scheduler
    namespace: kube-system
---
# Leader-election Lease named "my-scheduler" lives in kube-system; grant it.
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-scheduler-lease
  namespace: kube-system
rules:
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["create", "get", "update", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-scheduler-lease
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: my-scheduler-lease
subjects:
  - kind: ServiceAccount
    name: my-scheduler
    namespace: kube-system
EOF
```

---

## 2. KubeSchedulerConfiguration (a ConfigMap)

This is where the second scheduler declares its name. The profile
`schedulerName: my-scheduler` is what pods target. Note the **current, stable**
config API version `kubescheduler.config.k8s.io/v1`. We give leader election a
**distinct** `resourceName` so it doesn't collide with the default scheduler's
`kube-scheduler` lease.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-scheduler-config
  namespace: kube-system
data:
  config.yaml: |
    apiVersion: kubescheduler.config.k8s.io/v1
    kind: KubeSchedulerConfiguration
    leaderElection:
      leaderElect: true
      resourceName: my-scheduler
      resourceNamespace: kube-system
    profiles:
      - schedulerName: my-scheduler
EOF
```

---

## 3. Deploy the second scheduler (a normal Deployment)

`replicas: 1` with leader election on: only one active instance ever binds
pods. In-cluster credentials come from the ServiceAccount, so no kubeconfig flag
is needed. `$SCHED_IMAGE` is expanded from step 0 — this heredoc is **not**
quoted so the variable substitutes.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-scheduler
  namespace: kube-system
  labels:
    app: my-scheduler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-scheduler
  template:
    metadata:
      labels:
        app: my-scheduler
    spec:
      serviceAccountName: my-scheduler
      containers:
        - name: kube-scheduler
          image: ${SCHED_IMAGE}
          command:
            - /usr/local/bin/kube-scheduler
            - --config=/etc/kubernetes/my-scheduler/config.yaml
            - -v=2
          volumeMounts:
            - name: config
              mountPath: /etc/kubernetes/my-scheduler
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: my-scheduler-config
EOF
```

**Confirm it's running and won the lease:**

```bash
kubectl get pods -n kube-system -l app=my-scheduler
# NAME                            READY   STATUS    RESTARTS   AGE
# my-scheduler-xxxxxxxxxx-xxxxx   1/1     Running   0          20s

# The leader-election Lease it created:
kubectl get lease -n kube-system my-scheduler
# NAME           HOLDER                          AGE
# my-scheduler   my-scheduler-xxxxxxxxxx-xxxxx   25s

# Logs should show it acquired leadership and started serving the profile.
kubectl logs -n kube-system -l app=my-scheduler | grep -iE 'leader|my-scheduler'
```

> If the pod `CrashLoopBackOff`s, the usual cause is a **version mismatch**
> (image minor != cluster minor) or an RBAC gap — `kubectl logs` names it
> directly (e.g. `leases.coordination.k8s.io ... is forbidden`).

---

## 4. Route a pod to `my-scheduler`

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: uses-my-scheduler
spec:
  schedulerName: my-scheduler          # opt in to the second scheduler
  containers:
    - name: pause
      image: registry.k8s.io/pause:3.9
EOF
```

**Verify our scheduler bound it** — the `Scheduled` event's source is
`my-scheduler`, not `default-scheduler`:

```bash
kubectl get pod uses-my-scheduler
# NAME                READY   STATUS    ...
# uses-my-scheduler   1/1     Running   ...

kubectl describe pod uses-my-scheduler | sed -n '/Events:/,$p'
```

Expect:

```
Events:
  Type    Reason     Age   From          Message
  ----    ------     ----  ----          -------
  Normal  Scheduled  10s   my-scheduler  Successfully assigned default/uses-my-scheduler to q198-control-plane
  Normal  Pulled     9s    kubelet       Container image already present on machine
  ...
```

The **`From  my-scheduler`** column is the proof. Confirm the field on the pod:

```bash
kubectl get pod uses-my-scheduler -o jsonpath='{.spec.schedulerName}{"\n"}'
# my-scheduler
```

For contrast, a pod with **no** `schedulerName` is bound by `default-scheduler`:

```bash
kubectl run uses-default --image=registry.k8s.io/pause:3.9
kubectl describe pod uses-default | grep -A2 Events:
# ... From  default-scheduler  Successfully assigned ...
```

---

## 5. A missing scheduler leaves the pod `Pending` forever

If a pod names a `schedulerName` that **no running scheduler** claims, **no
scheduler will ever touch it** — it is not the default scheduler's job either.
It sits `Pending` with **no `FailedScheduling` event**, because nothing is even
attempting to schedule it.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: orphan-scheduler
spec:
  schedulerName: does-not-exist         # no scheduler answers to this name
  containers:
    - name: pause
      image: registry.k8s.io/pause:3.9
EOF
```

**Diagnose:**

```bash
kubectl get pod orphan-scheduler
# NAME               READY   STATUS    ...
# orphan-scheduler   0/1     Pending   ...

kubectl describe pod orphan-scheduler | sed -n '/Events:/,$p'
# Events:  <none>          <-- telling: NOT a FailedScheduling, just untouched

kubectl get pod orphan-scheduler -o jsonpath='{.spec.schedulerName}{"\n"}'
# does-not-exist
```

The absence of any event (vs a `FailedScheduling` event) is the tell that the
pod is waiting on a scheduler that isn't there — a classic "why is my pod stuck
and there's no scheduling error?" moment. Fix = deploy a scheduler with that
name, or correct/remove the `schedulerName`.

Clean up the demo pods:

```bash
kubectl delete pod uses-my-scheduler uses-default orphan-scheduler
```

---

## Teardown

```bash
# Remove the scheduler workload + config + RBAC.
kubectl delete deployment my-scheduler -n kube-system
kubectl delete configmap my-scheduler-config -n kube-system
kubectl delete clusterrolebinding my-scheduler-as-kube-scheduler my-scheduler-as-volume-scheduler
kubectl delete role my-scheduler-lease -n kube-system
kubectl delete rolebinding my-scheduler-lease -n kube-system
kubectl delete serviceaccount my-scheduler -n kube-system
kubectl delete lease my-scheduler -n kube-system --ignore-not-found

# Delete the cluster.
kind delete cluster --name q198
```

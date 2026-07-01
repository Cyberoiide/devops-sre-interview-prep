# Solution — Diagnosing a `Pending` pod

## The core insight

`Pending` almost always means **the scheduler could not bind the pod to a
node**. The scheduler does not fail silently — on every failed scheduling
cycle it writes a **`FailedScheduling`** event on the pod that states the
reason node by node. Your job is to read that event, classify it into one of a
few root-cause families, and act. Do not guess; the cluster already told you.

## Step 0 — Read the event (always the first move)

```bash
kubectl describe pod <name>            # scroll to the Events: section
# or, cluster-wide and time-ordered:
kubectl get events --sort-by=.lastTimestamp
```

The `FailedScheduling` message has a very consistent shape:

```
0/<N> nodes are available: <count> <reason>, <count> <reason>, ...
```

It enumerates, per group of nodes, *why each was rejected*. The counts always
add up to the total node count, which is how you know you've accounted for
every node. The reason strings are what you pattern-match on.

## The three root-cause families

### 1. No node matches the pod's placement constraints

The pod asked to go somewhere specific and nowhere qualifies.

| Mechanism | Tell-tale event string |
|---|---|
| `nodeSelector` / required `nodeAffinity` | `didn't match Pod's node affinity/selector` |
| `podAffinity` / `podAntiAffinity` | `didn't match pod affinity/anti-affinity rules` (or `...topology key...`) |
| Node **taint** with no matching **toleration** | `had untolerated taint {key: value}` |

**Taints and effects** — a taint repels pods that lack a matching toleration:

- `NoSchedule` — won't schedule new pods without a toleration (already-running
  pods stay).
- `PreferNoSchedule` — soft version; scheduler *tries* to avoid the node but
  will use it if needed.
- `NoExecute` — won't schedule, **and evicts** already-running pods that don't
  tolerate it.

Common built-in taints you'll actually meet: the control-plane taint
`node-role.kubernetes.io/control-plane:NoSchedule` (why your workloads never
land on masters), and node-condition taints the node controller adds
automatically such as `node.kubernetes.io/not-ready:NoExecute`,
`node.kubernetes.io/unreachable:NoExecute`,
`node.kubernetes.io/disk-pressure`, `node.kubernetes.io/memory-pressure`, and
`node.kubernetes.io/unschedulable` (set when you `kubectl cordon`).

**Investigate:**

```bash
kubectl get nodes --show-labels                 # do any nodes have the label?
kubectl describe node <node> | grep -i taint    # what taints exist?
kubectl get pod <name> -o yaml | grep -A20 -E 'nodeSelector|affinity|tolerations'
```

**Fix:** label a node to match, relax/remove the selector or affinity, or add
the missing toleration. Never blindly add a toleration for a condition taint
(`not-ready`, `disk-pressure`) — those signal a genuinely broken node.

### 2. Insufficient resources

The pod's **requests** don't fit anywhere.

Tell-tale strings: `Insufficient cpu`, `Insufficient memory`, `Insufficient
ephemeral-storage`, or for extended resources `Insufficient nvidia.com/gpu`.

**Requests, not limits, drive scheduling.** The scheduler places a pod by
checking whether the sum of the pod's container **`requests`** fits into a
node's remaining allocatable (`Allocatable − Σ requests of pods already on that
node`). **Limits are ignored by the scheduler** — they only cap runtime usage
(CPU is throttled at the limit; memory over the limit gets OOM-killed). A pod
with no requests but huge limits schedules trivially; a pod with huge requests
and no limits may never schedule.

**Allocatable vs capacity:** `capacity` is the node's total hardware.
`allocatable` is what's left for pods after subtracting reservations for the
kubelet, the container runtime, and eviction thresholds (`kube-reserved`,
`system-reserved`, `eviction-hard`). The scheduler only ever uses
**allocatable**, so a node with 4 CPU capacity might advertise ~3.8 allocatable.

**Investigate:**

```bash
kubectl describe node <node>          # see Capacity, Allocatable, and
                                      # "Allocated resources" (sum of requests)
kubectl top nodes                     # actual live usage (needs metrics-server)
```

Note `kubectl top` shows *usage*, which can be far below *requests* — a node
can be "full" for scheduling (all requests claimed) while sitting near-idle.
Scheduling cares about requests, not current usage.

**Fix:** lower the pod's requests to something realistic, or add capacity.
If **Cluster Autoscaler** or **Karpenter** is configured, an unschedulable pod
is the *trigger* for it to provision a new node; the pod then schedules once
the node joins (typically a few minutes). Without an autoscaler you are simply
stuck until a node frees up or you add one.

### 3. Volume problems

The pod references a PVC that isn't `Bound`.

Tell-tale pod event: `pod has unbound immediate PersistentVolumeClaims` (or a
message about `waiting for a volume to be created`). The real diagnosis lives
on the **PVC**, not the pod.

**Investigate:**

```bash
kubectl get pvc                           # is the claim Pending or Bound?
kubectl describe pvc <claim>              # Events explain why (ProvisioningFailed, etc.)
kubectl get storageclass                  # does the requested SC exist? is one default?
kubectl get pv                            # for static provisioning, is a matching PV free?
```

Common causes:

- **StorageClass doesn't exist** → `storageclass ... not found`.
- **No default StorageClass** and the PVC named none → nothing provisions it.
- **No matching PV** (static provisioning) — size/accessMode/selector mismatch.
- **Topology / zone mismatch** — the volume can only be created in zone A but
  the only schedulable nodes are in zone B (common with zonal EBS/PD disks).
- **`WaitForFirstConsumer` behaving correctly** — the PVC *is supposed* to stay
  `Pending` (message: waiting for first consumer) until a pod using it is
  scheduled; binding and scheduling then resolve together. This is not a bug.
  Contrast `Immediate` binding, which provisions/binds the PV as soon as the
  PVC is created — if that can't be satisfied, the PVC blocks the pod on its
  own.

**Fix:** point the PVC at an existing StorageClass (or mark one default),
create/adjust a matching PV, or fix the topology constraint so a volume can be
created where a schedulable node exists.

## Other, less common `Pending` causes (mention them)

- **No schedulable nodes at all** — every node is `NotReady`, cordoned, or the
  cluster has zero worker nodes. `kubectl get nodes` shows it immediately.
- **Admission rejections: `ResourceQuota` / `LimitRange`.** A namespace over
  its quota, or a pod violating a `LimitRange` min/max/default, is rejected at
  **admission** — often the pod (or its controller's replica) never gets
  created, and you see the error on the ReplicaSet/Deployment events or the
  `kubectl apply` output rather than a `FailedScheduling`. Worth checking with
  `kubectl describe quota` / `kubectl describe limitrange` and the controller's
  events.
- **`ContainerCreating` / `ImagePullBackOff` are NOT `Pending`.** Image pulling
  happens on the kubelet *after* the pod is already bound to a node, so a bad
  image shows up as `ContainerCreating` or `ImagePullBackOff`, never as classic
  `Pending`. If you see those, scheduling already succeeded — look at the
  kubelet, not the scheduler.

## Decision tree

```
Pod is Pending
│
├─ kubectl describe pod <name>  → read the FailedScheduling event
│
├─ "didn't match Pod's node affinity/selector"
│        → check `kubectl get nodes --show-labels`; fix selector/affinity or label a node
│
├─ "had untolerated taint {k: v}"
│        → `kubectl describe node | grep -i taint`; add toleration OR remove taint
│          (if it's a condition taint like not-ready/disk-pressure, fix the NODE)
│
├─ "Insufficient cpu | memory | ephemeral-storage | <extended>"
│        → `kubectl describe node` allocatable vs Allocated; lower requests
│          or add a node (Cluster Autoscaler/Karpenter does this automatically)
│
├─ "unbound ... PersistentVolumeClaims" / volume message
│        → `kubectl get pvc`; `kubectl describe pvc`; `kubectl get storageclass`
│          fix SC name / default SC / matching PV / topology
│
└─ No FailedScheduling event at all?
         → `kubectl get nodes` (all NotReady / zero nodes?)
         → check ResourceQuota / LimitRange (admission), and the controlling
           Deployment/ReplicaSet events
         → confirm it's really scheduling-Pending, not ContainerCreating/ImagePull
```

## One-liner summary for the interviewer

> "`Pending` means the scheduler couldn't bind the pod, so I `describe` the pod
> and read the `FailedScheduling` event. It's essentially always one of three
> things: no node matches its placement constraints (selector/affinity or an
> untolerated taint), its resource **requests** don't fit any node's
> allocatable, or a PVC it needs is unbound. The event string tells me which,
> and each has a direct fix — with an autoscaler, the resource case fixes
> itself by adding a node."

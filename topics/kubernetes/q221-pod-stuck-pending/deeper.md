# Deeper — follow-up interview questions

### 1. Requests vs limits — which one does the scheduler use, and why?

Only **requests**. The scheduler places a pod by checking whether the sum of
its containers' **requests** fits in a node's remaining allocatable
(`allocatable − Σ requests already scheduled there`). **Limits are invisible to
the scheduler** — they're enforced at runtime by the kubelet/cgroups: CPU is
throttled at its limit, memory over its limit triggers an OOM kill. So a pod
with tiny requests and huge limits schedules easily (and can overcommit a
node), while a pod with huge requests never schedules even if it would barely
use anything. Requests also determine QoS class (Guaranteed / Burstable /
BestEffort) and eviction ordering.

### 2. What's the difference between `Pending` and `ContainerCreating`?

They happen at different stages and point at different components.
`Pending` = the **scheduler** hasn't bound the pod to a node yet (a scheduling
problem). `ContainerCreating` = the pod is **already bound** and the **kubelet**
on that node is doing its setup: pulling images, attaching/mounting volumes,
setting up the network sandbox. So `ImagePullBackOff`, `ErrImagePull`, and
volume-*mount* failures show up during `ContainerCreating`, never as classic
`Pending`. If you're `Pending`, debug the scheduler; if `ContainerCreating`,
debug the kubelet/CRI on the assigned node.

### 3. What are the built-in taints, and what is the control-plane taint?

Control-plane nodes carry `node-role.kubernetes.io/control-plane:NoSchedule`,
which keeps ordinary workloads off the masters unless they explicitly tolerate
it. The node controller also adds **condition-based** taints automatically:
`node.kubernetes.io/not-ready` and `node.kubernetes.io/unreachable` (both
`NoExecute`, so they evict), plus `memory-pressure`, `disk-pressure`,
`pid-pressure`, `network-unavailable`, and `node.kubernetes.io/unschedulable`
(added by `kubectl cordon`). Seeing these in a `FailedScheduling` event usually
means the node is genuinely unhealthy — fix the node rather than tolerating the
taint.

### 4. The three taint effects?

- **`NoSchedule`** — don't schedule new pods that lack a matching toleration;
  existing pods stay.
- **`PreferNoSchedule`** — soft/best-effort: the scheduler tries to avoid the
  node but will use it if nothing else fits.
- **`NoExecute`** — don't schedule **and evict** already-running pods that don't
  tolerate it. With `tolerationSeconds` a tolerating pod can be given a grace
  period before eviction.

### 5. `WaitForFirstConsumer` vs `Immediate` binding mode?

A StorageClass's `volumeBindingMode` controls *when* a PVC binds/provisions.
- **`Immediate`** — the volume is provisioned and bound as soon as the PVC is
  created, before any pod uses it. Risk: on multi-zone clusters the volume may
  land in a zone where the pod later can't be scheduled (topology mismatch).
- **`WaitForFirstConsumer`** — the PVC deliberately stays `Pending` (status
  "waiting for first consumer") until a pod that uses it is scheduled; the
  scheduler picks a node first, and the volume is then provisioned in that
  node's zone. This is the default for most cloud/local provisioners (including
  `kind`'s local-path) precisely because it makes topology and scheduling agree.
  A `Pending` PVC under this mode is normal, not an error.

### 6. How does Cluster Autoscaler (or Karpenter) change a `Pending` pod?

An unschedulable pod is the **trigger**. Cluster Autoscaler watches for pods
that fail scheduling due to insufficient resources, simulates which node group
would let them fit, and scales that group up; the pod schedules once the new
node registers (usually a couple of minutes). Karpenter goes further — it
provisions right-sized nodes directly from pod requirements rather than fixed
node groups. Both only help the **resource** family — they will not fix an
impossible `nodeSelector`, an untolerated taint the new node also has, or an
unbound PVC. Without any autoscaler, an over-request just stays `Pending`
indefinitely.

### 7. Pod affinity vs node affinity vs anti-affinity?

- **Node affinity** — attract a pod to nodes by **node labels** (the richer
  successor to `nodeSelector`; supports operators and preferences).
- **Pod affinity** — co-locate a pod near **other pods** matching a label
  selector, within a topology domain (`topologyKey`, e.g. same zone/host) — good
  for latency-sensitive pairs.
- **Pod anti-affinity** — keep a pod **away** from matching pods across a
  topology — the usual way to spread replicas across nodes/zones for HA.

Each comes in `requiredDuringSchedulingIgnoredDuringExecution` (hard — an
unsatisfiable required rule keeps the pod `Pending`) and
`preferredDuring...` (soft — best effort, won't block scheduling). A common
`Pending` cause is a hard anti-affinity ("no two replicas on the same node")
with more replicas than nodes.

### 8. Allocatable vs capacity?

`capacity` is the node's total resources (raw hardware the node reports).
`allocatable` is what's actually available to pods after subtracting system
reservations: `kube-reserved` (kubelet, runtime), `system-reserved` (OS
daemons), and the hard eviction threshold. The scheduler **only** uses
`allocatable`, so a 4-core / 16 Gi node might advertise ~3.8 cores / ~15 Gi
allocatable. `kubectl describe node` shows both, plus "Allocated resources"
(the sum of requests already committed) — that's what you compare your pod's
requests against.

### 9. How do `ResourceQuota` and `LimitRange` cause failures, and how do they
differ from a scheduling failure?

These act at **admission time**, before scheduling. A **`ResourceQuota`** caps
aggregate consumption in a namespace (total CPU/memory requests+limits, object
counts, PVC storage, etc.); a request that would exceed it is **rejected by the
API server**. A **`LimitRange`** sets per-object min/max and default
requests/limits in a namespace; a pod outside those bounds is rejected, and one
with no requests may get defaults injected (which can then be too big to
schedule). Because these fire at admission, the failure typically appears on
the `kubectl apply` output or on the **controlling Deployment/ReplicaSet
events** ("failed quota", "forbidden: exceeded quota"), and often the pod object
is never even created — so there's no `FailedScheduling` event to find. Check
with `kubectl describe quota` and `kubectl describe limitrange`, and read the
owning controller's events, not just the pod's.

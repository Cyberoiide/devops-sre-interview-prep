# Solution — Deploying a second / custom scheduler

## The core insight

The scheduler holds **no special place** in Kubernetes. It is a **plain
controller** that watches the API server for pods it owns, chooses a node, and
writes the binding. Because it's ordinary, you can **run several at once**, and
each pod picks its scheduler with **`spec.schedulerName`** (default:
`default-scheduler`). Deploying a custom one is therefore just "deploy a
Deployment with the right identity and permissions" — **not** flipping a magic
component switch.

## What the scheduler actually does (the loop)

```
watch API server for pods where:
    spec.nodeName == ""            (not yet placed)
    spec.schedulerName == <mine>   (this scheduler is responsible)
  → filter nodes that can run the pod   (predicates / Filter plugins)
  → score the survivors                  (priorities / Score plugins)
  → Bind: POST /api/v1/.../binding writing spec.nodeName
kubelet on the chosen node then pulls images and starts containers
```

Multiple schedulers coexist safely because each only acts on pods whose
`schedulerName` matches its own. A pod is never fought over.

## Correcting the "component type" myth

> A popular source video says *"the component **type** must be `scheduler`."*
> **This is wrong / imprecise.**

There is **no Kubernetes field or "component type"** that turns a workload into
a scheduler. What actually makes something a scheduler is purely behavioural: a
process that watches for unscheduled pods and calls the **Bind** API. You ship
it as:

- a **normal Deployment** (or a static pod on the control-plane) in
  `kube-system`;
- a **ServiceAccount** giving it an identity;
- **RBAC** granting the permissions it needs.

The `component=kube-scheduler` you may have seen is only a **label** on the
control-plane's static-pod manifest — a convention for selecting the pod, not a
type that grants scheduler powers. Anyone claiming you set `type: scheduler`
has confused a label/manifest convention with a nonexistent API concept.

## Steps to deploy a second scheduler

1. **Get/build the binary as an image.** For the classic demo, **reuse the
   upstream `kube-scheduler` image** and give it a distinct profile
   `schedulerName` via `KubeSchedulerConfiguration` (what the exercise does), or
   use the community `my-scheduler` example. For a *real* custom scheduler you
   build your Go binary (typically on the **Scheduler Framework**) into a
   container image and push it to a registry.
2. **Create a ServiceAccount + RBAC** in `kube-system`: bind the built-in
   `system:kube-scheduler` (and `system:volume-scheduler`) ClusterRoles, or
   author your own ClusterRole. It needs to **read/bind pods**, **read nodes**,
   **create events**, and **use leader-election `Leases`** (a distinct lease
   name needs its own grant — the built-in role's lease permission is scoped to
   `kube-scheduler`).
3. **Deploy it as a Deployment** with **leader election on** so exactly one
   replica is active and binds pods (others stand by).
4. **Create pods with `spec.schedulerName: <your-scheduler>`** and confirm your
   scheduler binds them — the `Scheduled` event's **`From`** column shows your
   scheduler's name.

### The RBAC, concretely

| Permission | Why the scheduler needs it |
|---|---|
| `get/list/watch` **pods**, `update` **pods/binding** | find unscheduled pods and bind them to a node |
| `get/list/watch` **nodes** | filter and score candidate nodes |
| `create/patch` **events** | write the `Scheduled` / `FailedScheduling` events |
| `get/list/watch` **pv, pvc, storageclass, csinode** | volume-aware scheduling (from `system:volume-scheduler`) |
| `create/get/update` **leases** (`coordination.k8s.io`) | leader election so only one replica binds |

Binding `system:kube-scheduler` + `system:volume-scheduler` covers the first
four rows. The lease row needs a small extra `Role`/`RoleBinding` **only because
your lease has a different name** than the built-in `kube-scheduler` lease.

## Two failure modes worth knowing

- **Scheduler down / never deployed, but pods target it** → those pods sit
  `Pending` with **no `FailedScheduling` event at all**. Nothing is trying to
  schedule them; the default scheduler ignores them because their
  `schedulerName` isn't `default-scheduler`. Contrast this with a real
  scheduling failure, which *does* emit `FailedScheduling`.
- **Two active binders** → if you disable leader election and run multiple
  replicas, two schedulers can both try to bind the same pod, causing conflict
  errors. Keep `leaderElect: true`.

## The modern alternatives (usually the RIGHT answer)

A whole separate scheduler binary is heavy. In most real cases you want one of
these instead:

- **Scheduling profiles** — a **single** `kube-scheduler` can expose **multiple
  `schedulerName`s** via multiple `profiles:` in one
  `KubeSchedulerConfiguration`, each enabling/disabling/tuning plugins. Pods
  pick a profile by `schedulerName`, with **no extra Deployment to run**. This
  is the lowest-effort way to get "different scheduling behaviour for different
  pods."
- **Scheduler Framework + plugins** — the supported extension model. You write
  plugins at defined extension points (`PreFilter`, `Filter`, `Score`,
  `Reserve`, `Permit`, `Bind`, …) and compile them into a scheduler binary.
  This is how you build genuinely custom logic.
- **Scheduler extenders** — the older **webhook** approach: the default
  scheduler calls out to your HTTP service to filter/prioritize/bind. Higher
  latency, largely superseded by the Framework, but still seen.
- **`scheduler-plugins` project** — ready-made out-of-tree plugins:
  **coscheduling** (gang scheduling), **capacity scheduling**, **network-aware**
  placement, node-resources-fit variants.

## Why write a real custom scheduler at all?

- **Gang / batch scheduling** — all-or-nothing placement for MPI/Spark jobs
  (start N pods together or none).
- **Topology / cost-aware placement** — pack onto cheapest zones/instance types,
  respect rack/failure-domain layouts.
- **ML / GPU bin-packing** — fit fractional GPUs, keep whole GPUs free for large
  jobs, honor NVLink/NUMA topology.

For anything less exotic, reach for **profiles** or an **existing plugin** first.

## Decision guide

```
Need different scheduling behaviour?
│
├─ Just different plugin weights / a named profile for some pods?
│      → add a PROFILE to the existing kube-scheduler (no new Deployment)
│
├─ Need a well-known behaviour (gang, capacity, network-aware)?
│      → use the scheduler-plugins project's plugin
│
├─ Need bespoke logic you must write in Go?
│      → Scheduler Framework plugin, compiled into a scheduler binary,
│        deployed as a Deployment + SA + RBAC (leader election on)
│
├─ Have an external service that must make placement calls?
│      → scheduler extender (webhook) — legacy, higher latency
│
└─ Truly a separate scheduler process?
       → Deployment + ServiceAccount + RBAC in kube-system;
         pods opt in via spec.schedulerName; NOT a magic "component type"
```

## One-liner summary for the interviewer

> "A scheduler is just a controller that watches for pods with no `nodeName` and
> the matching `schedulerName`, picks a node, and calls Bind — so you can run
> several. You deploy a second one as an **ordinary Deployment in `kube-system`
> with a ServiceAccount and RBAC** (bind `system:kube-scheduler` plus a lease
> grant, leader election on); there is **no magic `type: scheduler`**. Pods opt
> in with `spec.schedulerName`, and one that names a scheduler that isn't
> running sits `Pending` forever with no event. But in practice I'd first reach
> for **scheduling profiles** in the existing kube-scheduler or the **Scheduler
> Framework**, not a whole separate binary."

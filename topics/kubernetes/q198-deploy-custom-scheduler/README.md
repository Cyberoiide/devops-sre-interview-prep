# q198 — Deploy and run a second / custom scheduler

## The interview question

> "Your team wants pods placed by their own scheduling logic, not the default
> `kube-scheduler`. Walk me through how you'd deploy a **second scheduler**
> alongside the default one, how a pod chooses it, and what a scheduler
> actually *is* under the hood. Also — a training video says you just set the
> component **type** to `scheduler`; is that right?"

This question tests whether you understand that the scheduler is **not a magic
built-in slot** but an ordinary controller, and whether you know the *modern*
ways to customise scheduling (profiles, the Scheduler Framework) instead of
reaching for a whole separate binary. It also contains a deliberate trap you
should correct.

## What a scheduler actually is

The scheduler is **just a regular program** (a controller) that:

1. **Watches** the API server for pods that are unscheduled — pods whose
   `spec.nodeName` is still empty **and** whose `spec.schedulerName` matches the
   name this scheduler is responsible for.
2. **Picks a node** (filter + score) for each such pod.
3. **Binds** the pod to that node by issuing a **Bind** API call (writes
   `spec.nodeName`). The kubelet on that node then takes over.

There is nothing privileged about it. You can run **multiple schedulers
side-by-side** in one cluster. A pod says which scheduler should handle it via
**`spec.schedulerName`**. If that field is unset, it defaults to
**`default-scheduler`**, which is the name the built-in `kube-scheduler` claims.

## Correcting the common misconception

> A popular video claims *"the component **type** must be `scheduler`"*. That is
> **imprecise / wrong.**

There is **no special Kubernetes "component type"** you set to make something a
scheduler. You deploy a custom scheduler as an **ordinary workload** — a normal
**Deployment** (or a static pod) in the `kube-system` namespace — with:

- a **ServiceAccount**,
- **RBAC** (a `ClusterRole` + `ClusterRoleBinding`, or simply bind the built-in
  `system:kube-scheduler` role) granting it permission to **read/bind pods**,
  **read nodes**, **create events**, and **use leader-election `Leases`**,

and pods **opt in** by setting `spec.schedulerName`. Nothing about a magic
`type: scheduler`. The `component=kube-scheduler` label you may have seen is
only a **label** the control-plane manifests happen to use — it is not what
"makes" a scheduler.

## What you should be able to answer

- The scheduler is a **watch → filter/score → Bind** controller; you can run
  several at once, and pods route to one via `spec.schedulerName`.
- A custom scheduler is deployed as a **normal Deployment + ServiceAccount +
  RBAC**, not a special component type.
- **Leader election** so only one active replica binds pods.
- A pod whose `schedulerName` names a scheduler that **isn't running** stays
  `Pending` **forever** (good teaching point).
- The **modern alternatives** that are usually the *right* answer:
  **scheduling profiles** in a single `kube-scheduler` (one binary, multiple
  `schedulerName`s via `KubeSchedulerConfiguration`), the **Scheduler
  Framework** with plugins, and older **extenders** (webhooks). Plus the
  `scheduler-plugins` project (coscheduling/gang, capacity, network-aware).
- **Why** you'd ever write a real custom scheduler: gang/batch scheduling,
  topology/cost-aware placement, ML/GPU bin-packing.

## Files in this topic

- `exercise.md` — a runnable `kind` lab that deploys a **second scheduler**
  (a second `kube-scheduler` instance with its own profile `schedulerName:
  my-scheduler`), routes a pod to it, and shows a pod naming a nonexistent
  scheduler stuck `Pending`.
- `solution.md` — the full deploy walkthrough, the RBAC/lease reasoning, the
  misconception correction, and when to use profiles/framework instead.
- `deeper.md` — follow-up interview questions with concise answers.

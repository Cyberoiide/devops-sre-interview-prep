# q206 — Move workloads off a node pool onto a new one

## The interview question

> "You need to retire a group of nodes — say you're upgrading the node pool to a
> new machine type or a new Kubernetes version. Walk me through how you move all
> the running workloads off the old nodes onto a fresh pool **without taking the
> application down**, and how you make sure a strict availability requirement is
> respected while you do it."

This is a bread-and-butter operational task: node maintenance, kernel/OS
upgrades, machine-type migrations, and node-pool version bumps all come down to
the same primitive — **drain the old nodes, let the workloads reschedule onto
new ones**. A strong candidate knows the exact two commands (`cordon` then
`drain`), *why* each flag exists, and — crucially — that `drain` goes through
the **Eviction API** so it respects **PodDisruptionBudgets**, which is what
keeps the app up during the move.

## The two-step core

1. **`kubectl cordon <node>`** marks the node **unschedulable**
   (`SchedulingDisabled`). It adds the taint
   `node.kubernetes.io/unschedulable:NoSchedule` so **no new pods land** there.
   Pods already running keep running — cordon alone moves nothing.
2. **`kubectl drain <node> --ignore-daemonsets --delete-emptydir-data`** cordons
   the node first (automatically) and then **evicts** the existing pods so their
   controllers reschedule them elsewhere. On managed platforms the emptied node
   is then deleted when you delete the pool.

`kubectl uncordon <node>` reverses a cordon and makes the node schedulable
again — the move you'd do after maintenance if the node survives.

## Why the flags matter

- **`--ignore-daemonsets`** — DaemonSet pods are pinned one-per-node by the
  DaemonSet controller; drain **refuses to run** if it would leave them
  unmanaged, so you acknowledge them explicitly. They aren't rescheduled
  elsewhere (that's not how DaemonSets work) and are removed when the node goes.
- **`--delete-emptydir-data`** — drain **refuses** to evict a pod with an
  `emptyDir` volume unless you opt in, because that data is **local to the node
  and lost** on eviction. Call this out: it's an explicit acceptance of data
  loss for scratch volumes.
- **`--force`** — required to evict **bare pods** (pods not backed by a
  controller); without an owner nothing recreates them, so drain won't touch
  them unless you force it (they are simply deleted).
- **`--grace-period`** / **`--timeout`** — bound how long each pod's graceful
  shutdown gets and how long the whole drain waits before giving up.

## The bit interviewers are really probing: PDBs

`drain` does **not** raw-`delete` pods — it calls the **Eviction API**, which
**honours PodDisruptionBudgets**. A PDB (`minAvailable` or `maxUnavailable`)
caps how many replicas of an app may be down *voluntarily* at once. So drain
evicts pods **gradually**, blocking and retrying whenever the next eviction
would violate the budget — the application stays above its floor throughout the
migration. The flip side: a **too-strict PDB** (e.g. `minAvailable` equal to the
replica count) makes eviction impossible and **drain hangs forever**.

## Steering pods to the NEW pool

Draining tells pods to *leave*; it does **not** by itself decide *where they go*
— the scheduler re-runs its normal placement logic. To make sure they land on
the new pool:

- **Label the new nodes** (`pool=new`) and pin the workload with
  **`nodeSelector`** or a required **`nodeAffinity`**, or
- **Taint the old pool** so pods without a matching toleration are repelled off
  it and onto the new nodes.

`nodeSelector`/affinity *attracts* pods to nodes you choose; taints *repel* pods
off nodes — often you use both. New pods only reschedule onto the new pool if it
has **capacity** and the **placement rules** allow it.

## What you should be able to answer

- The precise difference between **cordon** (no new pods) and **drain**
  (evict existing pods), and that drain cordons first.
- Every flag above and the failure it prevents.
- That drain uses the **Eviction API** and therefore respects **PDBs** — and how
  a bad PDB deadlocks a drain.
- How **Deployments/ReplicaSets** reschedule evicted pods automatically, and the
  extra care **StatefulSets** need.
- The real managed-Kubernetes pattern: **GKE node pools**, **EKS managed node
  groups**, **AKS node pools** — create new pool, cordon+drain old, delete old —
  and how **surge/blue-green** upgrades and the **Cluster Autoscaler** fit in.

## Files in this topic

- `exercise.md` — a runnable `kind` lab: two simulated node pools, a Deployment
  pinned to the old pool with a PDB and a DaemonSet, then a live cordon+drain
  migration onto the new pool with the PDB respected.
- `solution.md` — the full migration walkthrough, flag-by-flag, the Eviction
  API / PDB mechanics, steering strategies, a decision tree, and a one-liner.
- `deeper.md` — follow-up interview questions with concise answers.

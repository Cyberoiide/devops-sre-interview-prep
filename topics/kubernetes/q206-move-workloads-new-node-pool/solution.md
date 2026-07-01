# Solution — Migrating workloads to a new node pool

## The core insight

Moving workloads off a set of nodes is two primitives: **cordon** stops *new*
pods landing there; **drain** *evicts* the pods already there so their
controllers reschedule them elsewhere. Drain does this through the **Eviction
API**, not raw deletes, which is exactly why it **respects
PodDisruptionBudgets** and can keep an app available while you migrate. Where
the evicted pods *go* is a separate concern you control with labels + selectors
or taints.

## Step 0 — Understand what each command does

```bash
kubectl cordon <node>     # mark unschedulable; nothing moves
kubectl drain  <node>     # cordon (if needed) THEN evict existing pods
kubectl uncordon <node>   # make schedulable again
```

- **`cordon`** sets `.spec.unschedulable=true` and adds the taint
  `node.kubernetes.io/unschedulable:NoSchedule`. The node shows
  `Ready,SchedulingDisabled`. **Existing pods keep running**; the scheduler just
  won't place new ones there.
- **`drain`** cordons first (so nothing you evict comes back to the same node),
  then walks the node's pods and **evicts** them. A Deployment/ReplicaSet
  notices the missing replica and creates a new one — on some other schedulable
  node.
- **`uncordon`** removes the unschedulable mark. Use it after maintenance if the
  node lives on. On managed platforms a drained node that's being replaced is
  simply **deleted** with its pool.

## Step 1 — The drain flags and the failure each prevents

```bash
kubectl drain <node> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force \
  --grace-period=30 \
  --timeout=5m
```

| Flag | Why it exists |
|---|---|
| `--ignore-daemonsets` | DaemonSet pods are managed **per node**; drain refuses to proceed with them present unless told to ignore. They aren't rescheduled elsewhere — they go away when the node does. **Almost always required.** |
| `--delete-emptydir-data` | Drain refuses to evict a pod with an `emptyDir` volume unless you opt in, because that data is **node-local and lost** on eviction. Explicit acceptance of scratch-data loss. |
| `--force` | Evicts **bare pods** (no controller). Nothing recreates them, so they're **deleted, not rescheduled**. Without `--force`, their presence aborts the drain. |
| `--grace-period` | Overrides each pod's termination grace period (seconds) for shutdown. |
| `--timeout` | Caps how long drain waits overall before giving up (helps you not hang forever on a stuck PDB). |

## Step 2 — Eviction API and PodDisruptionBudgets (the key mechanic)

Drain issues **evictions**, not deletes. An eviction is checked against any PDB
selecting the pod:

- **`minAvailable: N`** — at least `N` pods (absolute or `%`) must stay Ready.
- **`maxUnavailable: N`** — at most `N` may be down voluntarily at once.

When the next eviction *would* breach the budget, the Eviction API returns
`429 Cannot evict pod as it would violate the pod's disruption budget`, and
`kubectl drain` **retries** until a replacement pod becomes Ready and headroom
opens up. Net effect: drain proceeds **one disruption at a time**, and the app
never dips below its floor. This is the whole reason PDBs exist — voluntary
disruptions (drains, node-pool upgrades) are gated; involuntary ones (a node
crashing) are not.

**The trap:** a PDB with `minAvailable` equal to the replica count (or
`maxUnavailable: 0`) means **zero** disruptions are ever allowed, so drain can
**never** evict and hangs (or fails at `--timeout`). `kubectl get pdb` shows
`ALLOWED DISRUPTIONS: 0` — that's your smoking gun.

```bash
kubectl get pdb
kubectl describe pdb <name>   # ALLOWED DISRUPTIONS, current healthy count
```

## Step 3 — Steer evicted pods onto the NEW pool

Drain decides *when pods leave*, not *where they go* — the scheduler re-places
them by normal rules. Two levers (often combined):

1. **Attract** — label new nodes and pin the workload:
   ```bash
   kubectl label node <new-node> pool=new
   ```
   with `nodeSelector: {pool: new}` or a required `nodeAffinity`.
2. **Repel** — taint the old pool so untolerating pods won't return to it:
   ```bash
   kubectl taint node <old-node> pool=old:NoSchedule
   ```

**`nodeSelector`/affinity vs taints/tolerations:**

- `nodeSelector`/`nodeAffinity` is **pod-driven attraction** — the pod says
  "put me on nodes with this label." Good default for "this workload belongs on
  that pool."
- **Taints** are **node-driven repulsion** — the node says "keep pods off me
  unless they tolerate this." Good for **reserving** a pool (GPU, licensed) so
  *only* opted-in workloads use it, and for pushing everything else off during a
  migration.
- They're orthogonal: a taint stops the wrong pods coming; a selector/affinity
  makes the right pods choose the pool. Real migrations use both.

Whichever you use, the new pool must have **enough capacity** and satisfy the
**placement rules**, or evicted pods just go `Pending` (see q221).

## Step 4 — Controller behaviour during the move

- **Deployment / ReplicaSet** — evicted pod → ReplicaSet recreates it → scheduler
  places it on a schedulable node (the new pool). Fully automatic. Combine with
  a PDB + `topologySpreadConstraints`/anti-affinity for a clean rolling move.
- **StatefulSet** — each pod has stable identity and usually a PVC. Eviction
  recreates the *same* ordinal, which reattaches its volume — fine if the volume
  is reachable from the new pool (watch **zone/topology**: a zonal disk only
  attaches to nodes in its zone). Give StatefulSets a PDB too; drain one node at
  a time. Quorum systems (etcd, Kafka) need `maxUnavailable: 1` so you never
  take down two members at once.
- **Bare pods** — no owner, so nothing recreates them; `--force` deletes them and
  they're gone. Real workloads should be behind a controller.

## The managed-Kubernetes migration pattern

The canonical "upgrade the node pool" runbook (new machine type, new K8s
version, new OS image):

1. **Create a new node pool** alongside the old one (GKE node pool / EKS managed
   node group or self-managed ASG / AKS node pool).
2. **Cordon** every node in the old pool (`kubectl cordon`, or the platform's
   "upgrade" does it for you).
3. **Drain** the old nodes one by one with `--ignore-daemonsets`
   `--delete-emptydir-data`; PDBs keep the apps up as pods reschedule onto the
   new pool.
4. **Delete the old pool** once empty.

Related mechanisms:

- **Surge upgrades** (GKE `maxSurge`/`maxUnavailable`, AKS max surge) add extra
  temporary nodes so pods have landing space before old nodes drain — cordon +
  drain under the hood, orchestrated for you and PDB-aware.
- **Blue-green node pools** — stand up a whole parallel pool, shift traffic, then
  retire the old one; lower risk for big version jumps, easy rollback.
- **Cluster Autoscaler** — during a drain it provisions capacity for pods that
  would otherwise be `Pending`, and later scales the emptied old pool to zero.
  It also **honours PDBs** when it scales a node group down, for the same reason
  drain does.

## Decision tree

```
Need to move workloads off nodes
│
├─ Just stop new pods (pre-maintenance staging)?
│        → kubectl cordon <node>        (existing pods stay)
│
├─ Actually evacuate the node?
│        → kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
│          ├─ bare (uncontrolled) pods present? add --force
│          └─ want to bound the wait?           add --timeout / --grace-period
│
├─ Drain hangs / "would violate disruption budget"?
│        → kubectl get pdb   → ALLOWED DISRUPTIONS: 0 = PDB too strict
│          relax minAvailable/maxUnavailable, or add capacity so replacements
│          go Ready and headroom opens
│
├─ Evicted pods land back on old nodes or go Pending?
│        → make sure old nodes are cordoned; label/taint so pods target the
│          NEW pool; check new pool has capacity + matching placement (see q221)
│
├─ Node coming back after maintenance?
│        → kubectl uncordon <node>
│
└─ Managed cluster, retiring the pool?
         → create new pool → cordon+drain old → delete old pool
           (or let surge/blue-green upgrade do it, PDB-aware)
```

## One-liner summary for the interviewer

> "I `cordon` the old nodes so nothing new lands, then `drain` them with
> `--ignore-daemonsets --delete-emptydir-data`, which evicts the running pods so
> their Deployments reschedule them. Drain goes through the Eviction API, so it
> respects PodDisruptionBudgets — it moves pods one disruption at a time and the
> app never drops below its floor. I steer the pods to the new pool with node
> labels + `nodeSelector` (or by tainting the old pool), and on a managed
> cluster I create the new pool first and delete the old one once it's empty.
> The classic gotcha is a PDB set to zero allowed disruptions, which makes drain
> hang forever."

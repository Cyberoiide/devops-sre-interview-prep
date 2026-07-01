# Deeper — follow-up interview questions

### 1. What's the exact difference between `cordon` and `drain`?

`cordon` marks a node **unschedulable** (`.spec.unschedulable=true`, plus the
`node.kubernetes.io/unschedulable:NoSchedule` taint) so **no new pods** get
placed there — but it **moves nothing**; pods already running stay running.
`drain` is the eviction step: it **cordons the node first** (so what you evict
doesn't come straight back), then **evicts** the existing pods so their
controllers reschedule them elsewhere. Rule of thumb: cordon = "stop accepting,"
drain = "cordon and evacuate."

### 2. Why does `drain` use the Eviction API instead of just deleting pods?

Because the Eviction API **honours PodDisruptionBudgets** and raw `delete` does
not. Eviction is a request the API server checks against every PDB selecting the
pod; if allowing it would drop the app below `minAvailable` (or above
`maxUnavailable`), the request is rejected with `429` and drain **retries**.
That's what makes a drain safe for a live app — it proceeds only as fast as the
budget allows. A `kubectl delete pod` bypasses all of that and can take you
below your floor instantly.

### 3. What is a PodDisruptionBudget and how does it interact with drain?

A PDB caps **voluntary** disruptions for a set of pods via `minAvailable`
(minimum that must stay Ready) or `maxUnavailable` (maximum down at once), as an
absolute number or a percentage. During a drain the Eviction API consults it and
lets pods go **one budget-slot at a time**, waiting for replacements to become
Ready before evicting more. PDBs guard only *voluntary* disruptions (drain,
node-pool upgrade, autoscaler scale-down) — a node hard-crashing is *involuntary*
and ignores the PDB.

### 4. My drain is hanging. What's the most likely cause?

A **too-strict PDB**. If `minAvailable` equals the replica count (or
`maxUnavailable: 0`), the budget allows **zero** disruptions, so no eviction can
ever succeed and drain retries forever (until `--timeout`). Check
`kubectl get pdb` — `ALLOWED DISRUPTIONS: 0` confirms it. Other causes: no
schedulable capacity for replacement pods (they stay `Pending`, so the budget
never recovers), or pods stuck terminating. Fixes: relax the PDB, add capacity
(new pool / autoscaler) so replacements go Ready, or bound the wait with
`--timeout`.

### 5. Why `--ignore-daemonsets` and `--delete-emptydir-data`?

DaemonSet pods are pinned one-per-node by their controller and are **not**
rescheduled onto other nodes — draining them elsewhere is meaningless, so drain
**refuses** unless you pass `--ignore-daemonsets` to acknowledge they'll simply
be removed with the node. `--delete-emptydir-data` is required whenever a pod
uses an `emptyDir`: that volume is **local to the node and lost** on eviction, so
Kubernetes makes you explicitly accept the data loss rather than silently
discarding scratch data.

### 6. What about `--force`, and what happens to bare pods?

`--force` lets drain evict **bare pods** — pods with no controller (ReplicaSet,
Job, StatefulSet, DaemonSet) behind them. Without an owner, **nothing recreates
them**, so drain won't touch them by default (their presence aborts the drain).
With `--force` they are **deleted and gone for good**. The lesson: run real
workloads under a controller so they survive node evacuation.

### 7. How do you make sure evicted pods land on the NEW pool, not back on old?

Cordoning the old nodes already stops pods returning there. To positively steer
them: **label** the new nodes and pin the workload with `nodeSelector` or a
required `nodeAffinity`, or **taint** the old pool (`pool=old:NoSchedule`) so
pods without a toleration are repelled onto the new pool. `nodeSelector`/affinity
is pod-side *attraction*; taints are node-side *repulsion* — real migrations
often use both. Either way the new pool needs enough capacity and matching
placement rules, or the pods go `Pending`.

### 8. `nodeSelector`/affinity vs taints/tolerations — when do you reach for each?

`nodeSelector` and `nodeAffinity` express **"this pod wants that kind of node"**
— attraction driven by the pod. Taints + tolerations express **"this node
repels pods unless they opt in"** — repulsion driven by the node, ideal for
**reserving** a pool (GPU, licensed software, a maintenance pool) so only
tolerating workloads use it. They're complementary: a taint alone keeps the
wrong pods off but doesn't pull the right ones in; a selector alone attracts but
doesn't stop others sharing the node. Use a taint to reserve and a
selector/affinity to target.

### 9. Do StatefulSets need special handling during a drain?

Yes. Each StatefulSet pod has a **stable identity** and usually a bound PVC.
Eviction recreates the *same ordinal*, which reattaches its volume — fine **only
if that volume is reachable from the new node**. On clouds a zonal disk (EBS/PD)
attaches solely to nodes in its zone, so the new pool must have capacity in the
right zone (`WaitForFirstConsumer` volume binding helps topology agree). Always
give StatefulSets a PDB and drain **one node at a time**; for quorum systems
(etcd, ZooKeeper, Kafka) set `maxUnavailable: 1` so you never lose two members
simultaneously.

### 10. What's the real node-pool upgrade pattern on GKE/EKS/AKS?

Create a **new node pool** (new machine type / K8s version / OS), then cordon and
drain the **old** pool node by node so workloads reschedule onto the new one, and
finally **delete the old pool**. Managed **surge upgrades** (GKE
`maxSurge`/`maxUnavailable`, AKS max surge) automate this by adding temporary
surge nodes first so pods have somewhere to go, doing PDB-aware cordon+drain
under the hood. **Blue-green node pools** stand up a full parallel pool and shift
over — safer for big jumps with easy rollback. The primitives are always the same
`cordon`/`drain`/`uncordon` you'd run by hand.

### 11. How does the Cluster Autoscaler interact with a drain / node-pool move?

Two ways. During a drain, pods that can't fit are `Pending`, which is the
Autoscaler's **trigger** to add capacity (or Karpenter to provision right-sized
nodes) so replacements schedule. And when the old pool sits empty, the
Autoscaler **scales it down** (to zero) to reclaim cost — importantly, its own
scale-down also goes through eviction and **respects PDBs**, so it won't yank a
node out from under an app that's at its disruption floor. It only ever helps the
capacity/scaling part — it won't fix an impossible selector or an unbound volume.

### 12. Is `uncordon` always needed after a migration?

Only if the node **survives**. For maintenance (kernel patch, reboot) where the
same node comes back, `kubectl uncordon <node>` clears the unschedulable mark and
the scheduler uses it again. In a node-pool *replacement*, the old nodes are
**deleted** with the pool, so there's nothing to uncordon — the cordon just kept
them quiet until they were destroyed.

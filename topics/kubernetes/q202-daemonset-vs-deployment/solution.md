# Solution — Choosing a DaemonSet vs a Deployment

## The core insight

Pick the controller by **what it guarantees about placement**:

- A **DaemonSet** guarantees **one pod per node** (or per matching subset). You
  never set a replica count — the effective count *equals the number of
  matching nodes*, and the controller tracks that set as nodes join and leave.
  Use it for **node-level infrastructure** — anything that must exist once on
  every node.
- A **Deployment** guarantees **N interchangeable replicas**, scheduled
  wherever they fit. There is **no per-node relationship**. Use it for
  **stateless application workloads** you scale by demand.

If the answer to "does this need to run on *every* node because it works *with*
the node?" is yes, it's a DaemonSet. Otherwise it's almost always a Deployment.

## How each controller places pods

**DaemonSet.** The DaemonSet controller ensures a pod exists on every node that
matches its `nodeSelector`/affinity (all nodes if none is set). Modern
Kubernetes doesn't bypass the scheduler: the controller injects **node affinity**
onto each pod so the **default scheduler** binds it to the intended node, which
means DaemonSet pods honor **taints and tolerations** like any other pod. That
is why an infrastructure DaemonSet typically carries **broad tolerations** — it
needs to run even on tainted and control-plane nodes. When a **new node joins**,
the controller creates a pod there automatically; when a node leaves, its pod is
removed. No `replicas` field exists.

**Deployment.** A Deployment owns a ReplicaSet that maintains `replicas: N`
pods. The scheduler places each replica on whatever node best fits its requests
(with spread/affinity heuristics). Two replicas can share a node; a node can
hold none. You change the count with `kubectl scale` or an **HPA**; you never
get a per-node guarantee without explicitly adding pod anti-affinity or topology
spread constraints (and even then it's a scheduling preference, not the
DaemonSet's intrinsic one-per-node contract).

## The use-case families for a DaemonSet

Everything here shares one trait: **it interacts with the node itself.**

| Category | Examples | Why per-node |
|---|---|---|
| Log collection / forwarding | Fluent Bit, Fluentd, Filebeat, Vector | Reads host log files via `hostPath` on every node |
| Monitoring / metrics agents | Prometheus **node-exporter**, Datadog agent, Dynatrace OneAgent, New Relic infra agent | Exposes node-level metrics; one exporter per node |
| CNI network plugins | Calico, Cilium, Flannel | Programs each node's networking / iptables / eBPF |
| Core networking | **kube-proxy** | Maintains service routing rules on every node |
| Storage daemons | CSI **node** plugins, GlusterFS, Ceph | Attaches/mounts volumes on the node they run on |
| Security / runtime agents | Falco, node problem detector | Watches syscalls / node health locally |

If you're being asked about a **log agent, metrics agent, CNI, kube-proxy, or a
CSI node plugin**, the answer is a DaemonSet.

## When it's a Deployment instead

Stateless services scaled horizontally by demand: web servers, REST/gRPC APIs,
frontends, workers behind a queue. They don't care which node they land on and
you want the *count* to track load (HPA), not the node population. Multiple
replicas per node are perfectly fine.

## Where StatefulSet fits (so you place all three)

A **StatefulSet** is for **stateful** workloads that need **stable network
identity and ordered lifecycle**: each pod gets a stable name
(`db-0`, `db-1`, …), its own `PersistentVolumeClaim` (via `volumeClaimTemplates`)
that survives rescheduling, and ordered/graceful rolling updates. Use it for
databases, message brokers, and clustered stores (PostgreSQL, Kafka, etcd,
Elasticsearch). It is **not** one-per-node (that's DaemonSet) and its replicas
are **not** interchangeable (that's Deployment) — they have identity.

## Updates differ too

- **DaemonSet** — `updateStrategy` is either **`RollingUpdate`** (with
  `maxUnavailable`, optionally `maxSurge` on newer versions) which replaces
  node pods gradually, or **`OnDelete`** which updates a node's pod only when you
  delete it manually (useful for carefully staged node-agent rollouts).
- **Deployment** — `RollingUpdate` with **`maxSurge`/`maxUnavailable`**,
  implemented by spinning up a **new ReplicaSet** and scaling the old one down;
  this also gives you rollback to a previous ReplicaSet. A DaemonSet has no
  surging *new ReplicaSet* model because it's bounded to the node set.

## Targeting a subset and handling taints

- **Subset of nodes:** add a `nodeSelector` (e.g. `disk: ssd`) or
  `nodeAffinity` (e.g. only GPU nodes) to the pod template; the DaemonSet then
  runs only on matching nodes, and its count follows that subset as labels
  change.
- **Taints:** because DaemonSet pods go through the scheduler, a taint (like the
  control-plane `node-role.kubernetes.io/control-plane:NoSchedule`) keeps the
  pod off a node unless the template has a matching **toleration**.
  Infrastructure DaemonSets typically tolerate broadly (`operator: Exists`) so
  they still cover tainted and control-plane nodes.

## Decision guidance

```
Does the workload need to run once on EVERY node (or every node of a class),
because it reads host files / exposes node metrics / programs node networking?
│
├─ YES → DaemonSet
│        (no replica count; add nodeSelector/affinity for a subset,
│         add tolerations to cover tainted/control-plane nodes)
│
└─ NO
   │
   ├─ Stateless, scale by demand, replicas interchangeable? → Deployment (+ HPA)
   │
   └─ Needs stable identity / per-pod storage / ordered rollout? → StatefulSet
```

## One-liner summary for the interviewer

> "For a log or metrics agent I'd use a **DaemonSet**, because it guarantees
> exactly one pod per node and automatically adds a pod when a new node joins —
> that's the right fit for node-level infrastructure that reads host logs or
> exposes node metrics. A **Deployment** is for stateless apps: you set a
> replica count and the scheduler scatters them by fit, with no per-node
> guarantee. If the workload needed stable identity and storage I'd reach for a
> **StatefulSet** instead. And since DaemonSet pods go through the scheduler,
> I'd give an infrastructure agent broad **tolerations** so it also covers
> tainted and control-plane nodes, plus a `nodeSelector` if it should only run
> on a subset like GPU nodes."

# Deeper — follow-up interview questions

### 1. Why doesn't a DaemonSet have a `replicas` field?

Because the count isn't something you choose — it's **derived from the node
set**. A DaemonSet's contract is "one pod per matching node," so the effective
number of pods always equals the number of schedulable nodes that match its
`nodeSelector`/affinity (and tolerate/aren't-tainted-against). If you could set
`replicas`, it would contradict that guarantee. The controller recomputes the
count continuously as nodes join, leave, or gain/lose matching labels, which is
exactly the behavior you want for node-level infrastructure.

### 2. How does a DaemonSet pod actually get scheduled — does it bypass the scheduler?

Not on modern Kubernetes. Historically the DaemonSet controller bound pods to
nodes directly, but since ~1.12 it instead **injects `nodeAffinity`** (pinning
the pod to a specific node) onto each pod and lets the **default scheduler** do
the binding. The practical consequences: DaemonSet pods respect **taints and
tolerations**, resource **requests** are considered, and priority/preemption
works normally. That's why an infrastructure DaemonSet must carry the right
tolerations to land on tainted or control-plane nodes.

### 3. A new node joins the cluster at 3am. What happens to each controller?

- **DaemonSet:** the controller notices the new node matches, and creates a pod
  on it automatically — no human action, no scaling. This is the whole point.
- **Deployment:** nothing changes. The replica count is fixed, so no new pod is
  created just because a node appeared (the scheduler might *later* place a
  rescheduled or newly-scaled replica there, but joining a node doesn't itself
  add pods).

### 4. Can you achieve "one Deployment pod per node" with anti-affinity instead of a DaemonSet? Should you?

You *can* approximate it with `requiredDuringSchedulingIgnoredDuringExecution`
pod anti-affinity keyed on `kubernetes.io/hostname` (or `topologySpread
Constraints`), forcing at most one replica per node. But it's the wrong tool:
you'd still have to keep `replicas` in sync with the node count manually, a hard
anti-affinity rule with more replicas than nodes leaves pods **`Pending`**, and
a new node wouldn't get a pod automatically. A DaemonSet gives you all of that
for free. Use anti-affinity to *spread* a scalable app; use a DaemonSet when you
genuinely need coverage of every node.

### 5. How do DaemonSet rolling updates differ from Deployment rolling updates?

- **Deployment** rolls by creating a **new ReplicaSet** and shifting pods from
  old to new using `maxSurge` (extra pods allowed above desired) and
  `maxUnavailable`. Because the old ReplicaSet is kept, you get easy rollback.
- **DaemonSet** has no surging-ReplicaSet model — it's bound to the node set, so
  it can't create "extra" pods on nodes that already have one. Its
  `updateStrategy` is either **`RollingUpdate`** (replace node pods gradually,
  governed by `maxUnavailable`, with `maxSurge` available on recent versions) or
  **`OnDelete`** (a node's pod updates only when you delete it), which lets
  operators stage node-agent rollouts node by node.

### 6. How do you run a DaemonSet on only some nodes — say GPU nodes?

Add a `nodeSelector` or (for richer matching) `nodeAffinity` to the pod
template, e.g. `nodeSelector: {accelerator: nvidia-gpu}` or a required
`nodeAffinity` matching a GPU label. The DaemonSet then runs only on matching
nodes and its count follows that subset. If those GPU nodes are also **tainted**
(common — GPU nodes are often dedicated), the template needs a matching
**toleration** as well; selector and toleration are independent gates that both
must pass.

### 7. Why do DaemonSets so often need tolerations, and what does `operator: Exists` do?

Because DaemonSet workloads (CNI, kube-proxy, logging, metrics) usually must
cover **every** node — including control-plane nodes and nodes that were tainted
to keep ordinary apps off. Since DaemonSet pods go through the scheduler, a
`NoSchedule` taint blocks them just like any pod unless they tolerate it. A
toleration with `operator: Exists` and **no `value`** matches the taint key
regardless of its value (and with no `key` at all, tolerates *everything*) —
which is why system DaemonSets like `kube-proxy` tolerate broadly. Contrast
`operator: Equal`, which must match a specific `key=value`.

### 8. DaemonSet, Deployment, or StatefulSet — one line each on when to reach for them.

- **DaemonSet** — node-level infrastructure that must run once per node and
  interact with the node (log/metrics agents, CNI, kube-proxy, CSI node
  plugins). Count = node count.
- **Deployment** — stateless apps/APIs scaled by demand; interchangeable
  replicas placed by fit, driven by `replicas`/HPA.
- **StatefulSet** — stateful workloads needing stable identity, per-pod
  persistent storage, and ordered rollout (databases, Kafka, etcd).

### 9. Do DaemonSet pods count against node resources and scheduling?

Yes. Their **requests** are subtracted from each node's allocatable just like
any pod, and they're scheduled with normal priority (system DaemonSets often run
at a high `priorityClass` like `system-node-critical` so they aren't preempted
and can even preempt lower-priority app pods to guarantee coverage). This
matters for capacity planning: a fleet of DaemonSets (logging + metrics + CNI +
CSI) reserves a slice of every node before any application pod schedules, so
those requests should be sized carefully.

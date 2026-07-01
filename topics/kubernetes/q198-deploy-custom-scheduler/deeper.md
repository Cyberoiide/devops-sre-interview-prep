# Deeper — follow-up interview questions

### 1. Is there really no special "component type" for a scheduler?

Correct — there isn't. A scheduler is defined purely by **behaviour**: a process
that watches the API server for unscheduled pods and issues **Bind** calls. You
run it as an **ordinary Deployment (or static pod)** in `kube-system` with a
ServiceAccount and RBAC. The `component=kube-scheduler` string is just a
**label** on the control-plane's static-pod manifest, used to select the pod —
not a type that confers scheduler powers. Any claim that you set
`type: scheduler` is confusing a labelling convention with a nonexistent API
field.

### 2. How does a pod choose which scheduler handles it?

Via `spec.schedulerName`. Each scheduler only acts on pods whose
`schedulerName` matches the name it was configured with (a profile's
`schedulerName` in `KubeSchedulerConfiguration`). If `spec.schedulerName` is
unset, it defaults to **`default-scheduler`**, which the built-in
`kube-scheduler` claims. This is how multiple schedulers coexist without
fighting over pods.

### 3. What happens to a pod whose `schedulerName` has no running scheduler?

It stays **`Pending` indefinitely** and — crucially — there is **no
`FailedScheduling` event**, because *no* scheduler is even attempting to place
it (the default scheduler ignores it too). The absence of any scheduling event
is the diagnostic tell. Fix by deploying a scheduler with that name or
correcting/removing `spec.schedulerName`.

### 4. Why leader election if you only want one scheduler?

To guarantee **exactly one active binder** even across restarts, rolling
updates, or accidental scale-up. With `leaderElect: true`, replicas contend for
a `Lease`; only the holder schedules, the rest stand by (fast failover). Without
it, two replicas could both try to bind the same pod, producing conflict errors
and double-scheduling races. Give your scheduler a **distinct lease name** so it
doesn't clash with the default scheduler's `kube-scheduler` lease.

### 5. What RBAC does a custom scheduler need, and why the extra lease Role?

It needs to `get/list/watch` pods and nodes, `update` pod **bindings**, `create`
**events**, and read volume objects (PV/PVC/StorageClass/CSINode) for
volume-aware scheduling. The easiest correct setup binds the built-in
`system:kube-scheduler` and `system:volume-scheduler` ClusterRoles. You still
add a small namespaced `Role`/`RoleBinding` for **leader-election Leases**
because the built-in role's lease permission is scoped to the lease named
`kube-scheduler`; your scheduler's lease has a **different name**, so it needs
its own `create/get/update` grant on `leases` in `kube-system`.

### 6. Second scheduler vs scheduling profiles — when each?

**Profiles** let a **single** `kube-scheduler` expose **multiple
`schedulerName`s**, each with its own plugin config, via several `profiles:` in
one `KubeSchedulerConfiguration`. No extra Deployment, no extra RBAC, no second
lease — it's the low-effort default when you just want *different behaviour* for
some pods. Run a **separate scheduler binary** only when you need logic you must
write yourself (a Scheduler Framework plugin build) or must isolate/version it
independently. Most "we need custom scheduling" asks are actually satisfied by a
profile or an existing plugin.

### 7. Scheduler Framework vs extenders?

The **Scheduler Framework** is the modern, in-process extension model: you write
plugins at defined extension points (`PreFilter`, `Filter`, `PreScore`, `Score`,
`Reserve`, `Permit`, `PreBind`, `Bind`, `PostBind`) and compile them into a
scheduler. It's fast (no network hop) and the supported path.
**Extenders** are the older approach: the default scheduler makes **HTTP webhook**
calls to your external service to filter/prioritize/bind. Extenders add latency
and can't participate in every phase, so they've largely been superseded by
Framework plugins — but you still meet them in legacy setups and some
integrations.

### 8. What's the `KubeSchedulerConfiguration` apiVersion, and what goes in it?

The current stable version is **`kubescheduler.config.k8s.io/v1`** (older
clusters used `v1beta*`). It configures leader election (`leaderElect`,
`resourceName`, `resourceNamespace`), client settings, and — the important part
— a list of **`profiles`**, each with a `schedulerName` and optional `plugins`
(enable/disable at each extension point) and `pluginConfig` (per-plugin args).
Multiple profiles in one file = one binary serving several `schedulerName`s.

### 9. Give real reasons to run a genuinely custom scheduler.

- **Gang / batch scheduling** — all-or-nothing placement so an MPI/Spark job's N
  pods start together or not at all (the default scheduler places pods
  independently and can deadlock on partial placement).
- **Topology / cost-aware placement** — pack onto the cheapest zones or instance
  types, respect rack/failure-domain constraints beyond stock affinity.
- **ML / GPU bin-packing** — allocate fractional GPUs, keep whole GPUs free for
  large jobs, honor NVLink/NUMA locality.

The `scheduler-plugins` project already ships coscheduling (gang), capacity, and
network-aware plugins — prefer those before writing your own.

### 10. Can two schedulers try to place the same pod?

Not if each pod's `schedulerName` matches exactly one scheduler — that's the
whole point of the field. The risk appears **within one scheduler** if you run
multiple active replicas **without leader election**: both watch the same pods
and race to bind, and the loser gets an `AlreadyScheduled`/conflict error.
Leader election prevents this by ensuring a single active instance. Across
*different* schedulers, non-overlapping `schedulerName`s keep them in separate
lanes.

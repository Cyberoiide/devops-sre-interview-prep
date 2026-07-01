# q211 — Solution & walkthrough

## Framing the answer

This question rewards a **structured, phased** answer over a pile of commands.
State clearly up front that a database is a *stateful* workload, so the naive
"just run it in a Deployment" answer is wrong, and that minimal downtime
requires **live replication**, not a dump-and-restore. Then walk the four
phases: provision, replicate, cut over, verify (+ rollback).

## Phase 1 — Provision the target correctly

**Use a `StatefulSet`, not a `Deployment`.** Databases need:

- **Stable network identity.** StatefulSet pods get ordinal, sticky names
  (`orders-db-0`, `orders-db-1`, …) and, via a **headless Service**
  (`clusterIP: None`), stable DNS records
  (`orders-db-0.orders-db.<ns>.svc.cluster.local`). Replication topology and
  client connection strings depend on knowing *which* pod is which — a
  Deployment's random pod names and load-balanced Service can't provide that.
- **Stable, durable storage.** `volumeClaimTemplates` give each pod its **own**
  PersistentVolumeClaim that is re-attached to the same pod ordinal across
  rescheduling and restarts. A Deployment's shared/ephemeral storage model
  doesn't fit.
- **Ordered, graceful lifecycle.** StatefulSets create/update/delete pods in
  order and wait for readiness — important when pod-0 is the primary and pod-1+
  are replicas.

**Size and match the storage to the VM.** The PVC's `storage` request must be
≥ the VM's disk usage with headroom, and the `storageClassName` must map to a
disk with comparable IOPS/throughput (e.g. `premium-ssd`/`gp3`/`pd-ssd`). If
you migrate a DB from an NVMe VM onto a slow default storage class, you'll
"succeed" and then get paged for latency. This is exactly what the lab's
`storage_size` / `storage_class` variables make explicit.

**Terraform's role here:** provision the cluster resources (namespace,
StatefulSet, Service, PVCs, secrets, network policy) declaratively so the target
is reviewable and reproducible. The lab renders the manifests locally so you
can validate them before they touch a cluster.

## Phase 2 — Replicate live (the key to "minimal downtime")

Do **not** do `pg_dump` on the VM and `pg_restore` into K8s as the primary
migration path — for a busy DB the dump is stale the instant it starts, and the
restore window is your outage.

Instead, bring up the K8s database as a **streaming replica** of the VM
primary:

- Seed it once (`pg_basebackup` from the VM, or restore a base backup).
- Point it at the VM as its upstream (`primary_conninfo`), so it continuously
  streams and replays WAL. It runs read-only as a **hot standby**.
- Now the K8s copy tracks the VM in near-real-time. Downtime is no longer
  "time to copy the whole DB"; it's "time to replay the last few KB of WAL and
  flip a switch."

The same pattern generalizes: MySQL binlog replication, MongoDB replica-set
member, or a managed logical-replication tool (e.g. `pglogical`, DMS-style
CDC). The principle is identical — **the target is a live replica, not a cold
copy.**

## Phase 3 — Cut over (the brief read-only window)

Ordering is everything, because the enemy is **split-brain** (both databases
accepting writes and diverging):

1. **Make the VM primary read-only first.** Block new writes at the DB
   (`default_transaction_read_only = on`) and/or at the app. No writes can land
   on the old primary from this instant.
2. **Drain the last WAL.** Wait until the standby's replay LSN catches the
   VM's flush LSN — replication lag = 0. Now the replica is byte-for-byte
   current.
3. **Promote** the StatefulSet pod to primary (`pg_promote()` /
   `pg_ctl promote`). It becomes writable.
4. **Repoint application traffic** to the new primary via the stable headless
   Service DNS (or a normal Service / connection-pooler like PgBouncer in
   front). Update the connection secret and roll the app deployment.

The only window where writes are refused is between steps 1 and 4 — typically
seconds to a couple of minutes — and there is **no data loss** because no write
was ever accepted on one side without being replicated to the other.

## Phase 4 — Verify and keep rollback ready

- **Observability:** watch write success/error rate, query latency (p50/p99),
  active connections, disk usage/IOPS on the new PVC, and — if you keep the VM
  as a downstream replica — replication lag in the new direction. Compare
  against the VM's historical baseline; a migrated DB that's 3× slower is a
  failed migration even if it "works."
- **Bake period:** leave the VM intact (ideally as a **downstream replica** of
  the new K8s primary) for a defined window before decommissioning, so rollback
  stays cheap.

### Rollback = the cutover in reverse

1. Put the **new** K8s primary read-only.
2. Make the **VM** writable again.
3. Repoint app traffic back to the VM.

This is lossless *only if* the VM still has every write. Two ways to guarantee
that: roll back before accepting meaningful writes on the new primary, or set
up the VM as a downstream replica during the bake so it keeps receiving the new
primary's WAL and can be promoted back cleanly. Say this explicitly — a
rollback plan that silently loses the writes taken after cutover is not a real
rollback plan.

## Why the lab is a scaffold, not a live migration

You can't stand up two real production databases and stream WAL between them in
a credential-free laptop exercise, and pretending to would teach the wrong
thing. The genuinely testable, reviewable artifacts are:

- the **target topology** (StatefulSet + PVC template + headless Service), which
  the lab renders and validates as real Kubernetes YAML, and
- the **cutover runbook**, which is what actually gets rehearsed in a real
  migration game day.

So the lab makes those two concrete and correct, and treats the live
replication/promotion as the documented runbook it would be in practice.

## The 60-second version to say out loud

> "A database is stateful, so the target is a StatefulSet with per-pod PVCs
> sized and perf-matched to the VM's disk, fronted by a headless Service for
> stable pod DNS. I don't dump-and-restore — I bring the K8s pod up as a live
> streaming replica of the VM so it tracks it in real time. At cutover I make
> the VM read-only first to prevent split-brain, wait for replication lag to
> hit zero, promote the StatefulSet pod to primary, and repoint the app at the
> Service DNS. I watch latency/error/lag metrics, and I keep the VM alive as a
> downstream replica during a bake period so rollback is just the same sequence
> reversed, with no data loss."

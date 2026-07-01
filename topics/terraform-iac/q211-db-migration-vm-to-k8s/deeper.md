# q211 — Interviewer follow-up questions

---

**Q: Why a StatefulSet and not a Deployment for the database?**

Deployments assume interchangeable, stateless pods with random names and
shared/ephemeral storage behind a load-balancing Service. A database needs the
opposite: a **stable identity** (so you know pod-0 is the primary),
**stable per-pod storage** (each pod keeps its own PVC across reschedules), and
**ordered lifecycle** (bring up/scale/roll in a known order). StatefulSet
provides all three; Deployment provides none.

---

**Q: What exactly does a headless Service give you that a normal Service doesn't?**

A normal `ClusterIP` Service gives one virtual IP and **load-balances** across
pods — fatal for a database, because a client would randomly hit a replica when
it meant to hit the primary. A **headless** Service (`clusterIP: None`) creates
**per-pod DNS A records** instead
(`orders-db-0.orders-db.<ns>.svc.cluster.local`), so replication config and
clients can address a *specific* pod deterministically. You often still put a
normal Service or a pooler (PgBouncer) in front for the app's read/write
endpoint, but the headless Service is what makes stable identity resolvable.

---

**Q: How do you measure replication lag, and what's an acceptable value at cutover?**

For PostgreSQL: on the primary, `pg_stat_replication` shows each standby's
`replay_lsn` vs the primary's `pg_current_wal_lsn()`; on the standby,
`pg_last_wal_receive_lsn()` vs `pg_last_wal_replay_lsn()`. Lag is the byte
distance (or time via `pg_last_xact_replay_timestamp()`). At the *cutover
instant* you want lag = **0** — that's why you make the primary read-only first
and then wait for the standby to replay the final WAL before promoting. During
the pre-cutover soak, small steady-state lag is fine; a lag that keeps *growing*
means the standby can't keep up and you must fix throughput (storage IOPS,
network) before cutting over.

---

**Q: What is split-brain and precisely how does your sequence prevent it?**

Split-brain is two nodes both believing they're primary and both accepting
writes, so they diverge and you can't cleanly reconcile them. The sequence
prevents it by ensuring there is **never more than one writable node**: the old
primary is made read-only *before* the new one is promoted, and there's a wait
for lag=0 in between. At no instant are both writable. Fencing (STONITH-style)
and consensus tooling (Patroni, etcd-backed leader election) enforce this
automatically in HA setups.

---

**Q: Your DB is 500 GB. Doesn't the initial replica seed take hours — isn't that downtime?**

No — the seed (`pg_basebackup`) runs against the *live* VM with **zero
downtime**; the VM keeps serving reads and writes while the base backup copies.
Once seeded, the standby streams WAL to catch up and then stays caught up. The
only downtime is the brief read-only window at cutover to drain the final WAL
and flip. The multi-hour copy happens entirely online, ahead of time.

---

**Q: Would you actually run a production database in Kubernetes? What are the risks?**

It's viable but non-trivial. Risks: storage performance and correctness (the
CSI driver and storage class must give real durable, fast block storage —
`ReadWriteOnce` per pod), pod rescheduling causing failovers, and the operational
burden of backups/upgrades/failover. In practice you'd use a mature **operator**
(CloudNativePG, Zalando/Crunchy Postgres, Vitess for MySQL) rather than a bare
StatefulSet, because operators handle failover, backups, and replication
lifecycle. Be honest in the interview: for many orgs a managed DB (RDS/Cloud
SQL) is the better call, and "should this even move to K8s?" is a fair question
to raise.

---

**Q: How would you use an operator like CloudNativePG instead of a raw StatefulSet?**

You declare a `Cluster` custom resource (instances count, storage, backups,
bootstrap-from-existing) and the operator manages the underlying StatefulSet(s),
Services, failover, and replication for you. For this migration you'd use the
operator's `bootstrap.pg_basebackup` / `externalClusters` to seed from the VM as
a replica, then trigger a controlled promotion. The raw StatefulSet in the lab
is the right thing to *understand*; an operator is the right thing to *run*.

---

**Q: Where does Terraform stop and where do runbook/imperative steps take over?**

Terraform is great for the **declarative, reproducible provisioning**: cluster,
namespace, StatefulSet/operator CR, Services, PVCs, secrets, network policies,
monitoring. It's a poor fit for the **imperative, time-ordered cutover**
(make read-only → wait for lag=0 → promote → repoint), which is inherently
stateful, needs human judgment and health gates, and shouldn't be encoded as a
`terraform apply`. So: Terraform for the target infra, a scripted/observed
runbook (or an operator's promotion API) for the cutover itself.

---

**Q: How do you handle the app's connection string change without its own downtime?**

Put a stable indirection in front — a Service DNS name or a connection pooler
(PgBouncer) — so the app always points at one name, and cutover just re-points
that name/pooler backend rather than requiring an app redeploy. If you must
change a secret, roll the app with a readiness-gated rolling update so
in-flight connections drain. Ideally the app already reconnects on failure, so
flipping the backend is close to seamless.

---

**Q: What observability signals tell you the migration succeeded vs. quietly regressed?**

Compare against the VM's historical baseline, not just "is it up": write/read
error rate, query latency p50/p99, transactions/sec, active/idle connection
counts, replication lag (in the new direction if VM is now downstream), and
storage IOPS/throughput/latency and free space on the new PVC. A migration that
"works" but doubles p99 latency because the storage class is slower is a failed
migration — catch it with the metrics before customers do.

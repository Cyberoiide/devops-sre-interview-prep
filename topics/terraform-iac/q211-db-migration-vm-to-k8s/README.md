# q211 — Migrate a database from a VM to Kubernetes with minimal downtime

## The scenario (as an interviewer would pose it)

> "We run our primary PostgreSQL database on a single beefy VM. Leadership wants
> it moved onto our Kubernetes cluster so it's managed the same way as
> everything else. It's a busy production database — we can tolerate a *brief*
> read-only window at cutover, but not an extended outage, and absolutely no
> data loss.
>
> Design the migration. How do you provision the target on Kubernetes, how do
> you move the data with minimal downtime, how do you cut over, and what's your
> rollback plan if it goes wrong? Be specific about the Kubernetes primitives
> you'd use and why."

## What the interviewer is really probing

1. Do you know the right Kubernetes primitives for **stateful** workloads —
   `StatefulSet`, `PersistentVolumeClaim`s, and **headless Services** — and
   *why* each matters for a database?
2. Do you understand that minimal downtime requires **live replication** (the
   K8s side is a replica syncing in real time), **not** a cold dump-and-restore?
3. Do you handle **split-brain** correctly at cutover (make the old primary
   read-only *before* promoting the new one)?
4. Do you have a concrete, reversible **rollback** plan?
5. Do you think about **observability** — how you *know* the cutover is safe
   (replication lag ≈ 0) and healthy afterward?

## The short answer

This is an architecture-and-runbook question more than a "write the HCL"
question. The migration is four phases:

1. **Provision the target.** A `StatefulSet` with `PersistentVolumeClaim`
   templates sized and performance-matched to the VM's disk, fronted by a
   **headless Service** so each pod gets a stable, predictable network identity.
2. **Replicate live.** Stand up the K8s database pod as a **streaming replica**
   of the VM's database so data flows continuously and the replica stays
   nearly in sync — not a one-shot copy that's stale the moment it finishes.
3. **Cut over.** When replication lag is ~0: put the VM database **read-only**
   (stops writes, prevents split-brain), let the replica drain the last bit of
   WAL, then **promote** the StatefulSet pod to primary and repoint application
   traffic to the headless/Service DNS name.
4. **Verify & keep rollback ready.** Watch metrics; if something's wrong,
   reverse the steps — new primary read-only, VM writable again, repoint back.

The lab in this folder is a **Terraform + Kubernetes-manifest scaffold** that
models the StatefulSet, the PVC template, and the headless Service, plus a
written cutover runbook — all runnable locally with no cloud and no live
database, using the `local_file` and `local` providers to render and validate
the manifests.

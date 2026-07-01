# q211 — Hands-on lab: scaffold the K8s target and write the cutover runbook

This is an architecture exercise, so the lab is a **scaffold**, not a live
database migration. You'll use Terraform (with the `local` provider) to
*render and validate* the Kubernetes manifests for the target — a StatefulSet,
its PVC template, and a headless Service — and then work through a cutover
runbook/checklist.

No cloud, no Kubernetes cluster, and no live database are required. Everything
runs locally with `terraform`/`tofu`. (If you *do* have `kubectl` and a
throwaway cluster like `kind` or `minikube`, an optional final step lets you
apply the rendered manifests for real.)

Verified against Terraform 1.15.

---

## 0. Set up the working directory

```bash
mkdir -p /tmp/q211-db-migration && cd /tmp/q211-db-migration
```

## 1. The Terraform that renders the target manifests

Create `main.tf`. It builds the manifests as HCL objects, `yamlencode`s them,
and writes them to `rendered/` — so you can diff and validate the exact YAML
before it ever touches a cluster.

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.5"
    }
  }
}

variable "db_name"       { default = "orders" }
variable "replicas"      { default = 1 }        # bump to 3 for an HA cluster
variable "storage_size"  { default = "100Gi" }  # MATCH the VM's disk size
variable "storage_class" { default = "fast-ssd" } # MATCH the VM's disk perf
variable "pg_image"      { default = "postgres:16" }

locals {
  # Headless Service: clusterIP=None gives each pod a STABLE DNS name
  # (orders-db-0.orders-db.<ns>.svc.cluster.local) instead of load-balancing.
  headless_service = yamlencode({
    apiVersion = "v1"
    kind       = "Service"
    metadata = {
      name   = "${var.db_name}-db"
      labels = { app = "${var.db_name}-db" }
    }
    spec = {
      clusterIP = "None"
      selector  = { app = "${var.db_name}-db" }
      ports     = [{ name = "postgres", port = 5432, targetPort = 5432 }]
    }
  })

  # StatefulSet: stable pod identity + per-pod persistent storage.
  statefulset = yamlencode({
    apiVersion = "apps/v1"
    kind       = "StatefulSet"
    metadata   = { name = "${var.db_name}-db" }
    spec = {
      serviceName = "${var.db_name}-db"   # binds SS to the headless Service
      replicas    = var.replicas
      selector    = { matchLabels = { app = "${var.db_name}-db" } }
      template = {
        metadata = { labels = { app = "${var.db_name}-db" } }
        spec = {
          terminationGracePeriodSeconds = 30
          containers = [{
            name  = "postgres"
            image = var.pg_image
            ports = [{ containerPort = 5432, name = "postgres" }]
            volumeMounts = [{
              name      = "data"
              mountPath = "/var/lib/postgresql/data"
            }]
            # Readiness gates traffic until Postgres actually accepts queries.
            readinessProbe = {
              exec                = { command = ["pg_isready", "-U", "postgres"] }
              initialDelaySeconds = 10
              periodSeconds       = 5
            }
          }]
        }
      }
      # PVC template: each pod gets its own PV. Size/class MUST match the VM
      # so the migrated DB isn't slower or out of space.
      volumeClaimTemplates = [{
        metadata = { name = "data" }
        spec = {
          accessModes      = ["ReadWriteOnce"]
          storageClassName = var.storage_class
          resources        = { requests = { storage = var.storage_size } }
        }
      }]
    }
  })
}

resource "local_file" "service" {
  filename = "${path.module}/rendered/service.yaml"
  content  = local.headless_service
}

resource "local_file" "statefulset" {
  filename = "${path.module}/rendered/statefulset.yaml"
  content  = local.statefulset
}

output "pod_dns_hint" {
  value = "Pod 0 stable DNS: ${var.db_name}-db-0.${var.db_name}-db.<namespace>.svc.cluster.local"
}
```

## 2. Render and inspect

```bash
terraform init
terraform apply -auto-approve
```

Look at the generated manifests:

```bash
cat rendered/service.yaml
cat rendered/statefulset.yaml
terraform output pod_dns_hint
```

Note in the output the three things that make this a *stateful* deployment:

- `clusterIP: None` on the Service → **headless** → stable per-pod DNS.
- `serviceName` on the StatefulSet → ties pods to that headless Service.
- `volumeClaimTemplates` → each pod gets its **own** persistent volume that
  survives reschedules; storage size/class match the source VM.

## 3. Model the "size it to the VM" decision with variables

The most common migration mistake is under-provisioning storage or using a slow
storage class. Re-render for a specific source VM (say a 500 GiB premium disk)
and, later, an HA 3-replica cluster:

```bash
# Match a 500Gi high-performance source disk:
terraform apply -auto-approve \
  -var 'storage_size=500Gi' -var 'storage_class=premium-ssd'
cat rendered/statefulset.yaml | grep -A4 volumeClaimTemplates

# Model an HA target (primary + 2 sync replicas):
terraform apply -auto-approve -var 'replicas=3'
grep 'replicas' rendered/statefulset.yaml
```

Because it's Terraform, `terraform plan` shows you the diff before you change
anything — useful for reviewing sizing changes in a PR.

## 4. Validate the YAML is well-formed (no cluster needed)

Any of these works; pick what you have installed:

```bash
# Pure syntax check with Python's YAML parser:
python3 -c 'import yaml,sys; [list(yaml.safe_load_all(open(f))) for f in ["rendered/service.yaml","rendered/statefulset.yaml"]]; print("YAML OK")'

# Or, if you have kubeconform / kubeval, schema-validate against the K8s API:
# kubeconform -summary rendered/*.yaml
```

## 5. (Optional) apply for real on a throwaway cluster

Only if you already have `kubectl` + a local cluster (`kind`, `minikube`, k3s):

```bash
# kind create cluster --name q211
# kubectl apply -f rendered/service.yaml -f rendered/statefulset.yaml
# kubectl get statefulset,pods,pvc
# kubectl exec -it orders-db-0 -- pg_isready -U postgres
# kind delete cluster --name q211
```

This is optional and NOT required to complete the exercise.

---

## 6. The cutover runbook (the real deliverable)

Manifests are the easy part. Write out — and be able to defend — this
sequenced runbook. This is what the interviewer actually wants.

### Phase A — Provision (no impact on the VM)
1. Apply the StatefulSet + headless Service (above). Confirm the pod is
   `Ready` and its PVC is bound with matching size/class.
2. Configure the K8s Postgres as a **streaming replica** of the VM primary
   (`primary_conninfo` pointing at the VM, replication user, `pg_basebackup`
   for the initial seed). It begins as a read-only standby.

### Phase B — Replicate & wait (still no impact)
3. Let the replica catch up. **Do not proceed** until replication lag is ~0.
   Watch `pg_stat_replication` on the VM / `pg_last_wal_receive_lsn()` vs
   `pg_last_wal_replay_lsn()` on the standby.
4. Dry-run the app's connection to the standby (read-only) from a canary pod.

### Phase C — Cutover (the brief read-only window)
5. **Put the VM primary read-only** (`ALTER SYSTEM SET default_transaction_read_only = on;`
   + reload, or stop the app's write path). This is what prevents
   **split-brain** — no new writes can land on the old primary.
6. Wait for the standby to replay the final WAL (lag hits 0).
7. **Promote** the StatefulSet pod to primary (`pg_ctl promote` / touch the
   promote trigger / `SELECT pg_promote()`).
8. **Repoint application traffic** to the headless Service DNS
   (`orders-db-0.orders-db.<ns>.svc.cluster.local` or a normal Service in
   front of it). Update the app's DB connection string / secret and roll the
   app.

### Phase D — Verify
9. Confirm writes succeed against the new primary; watch error rate, latency,
   connection counts, and replication (if you kept the VM as a downstream
   standby for safety).
10. Keep the VM around, read-only, for a defined bake period before
    decommissioning.

### Rollback (reverse of C)
- Put the **new** K8s primary read-only.
- Make the **VM** writable again (it still has all data up to the read-only
  point; if the K8s primary took writes, you must fail *back* via replication
  or reconcile — hence keep the VM as a downstream replica during the bake).
- Repoint app traffic back to the VM.
- Because writes were blocked during the window, there is **no data loss** on
  rollback as long as you roll back *before* accepting significant new writes,
  or you've set up bidirectional/downstream replication.

---

## 7. Clean up

```bash
rm -rf /tmp/q211-db-migration
```

## What you should take away

- The Kubernetes primitives are **StatefulSet + PVC template + headless
  Service**, chosen for stable identity and durable per-pod storage.
- Minimal downtime comes from **live streaming replication**, not a cold dump.
- The cutover is a **choreographed sequence**: VM read-only → drain lag →
  promote → repoint. Read-only-first is what stops split-brain.
- Rollback is the sequence in reverse, which only stays lossless if you keep
  the old primary intact (ideally as a downstream replica) during a bake period.

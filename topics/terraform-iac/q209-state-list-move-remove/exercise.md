# q209 — Hands-on lab: `state list`, `state mv`, `state rm`

This lab runs entirely locally. No cloud credentials, no network beyond the
one-time `terraform init`. We simulate "cloud resources" with the `local_file`
provider — each file on disk is a stand-in for a real resource (an EC2
instance, an S3 bucket, a database). You'll inventory state, rename and
relocate entries **without** destroying anything, and forget a resource while
proving the underlying object survives.

Works with `terraform` or `tofu` — substitute the binary name as needed. All
commands below were verified against Terraform 1.x.

---

## 0. Set up the working directory

```bash
mkdir -p /tmp/q209-state-demo && cd /tmp/q209-state-demo
```

Create `main.tf`:

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

# Each of these files stands in for a "cloud resource". We create three:
# a web server, a database, and a cache.
resource "local_file" "web" {
  filename = "${path.module}/resources/web-server.txt"
  content  = "web server config v1"
}

resource "local_file" "db" {
  filename = "${path.module}/resources/database.txt"
  content  = "database config v1"
}

resource "local_file" "cache" {
  filename = "${path.module}/resources/cache.txt"
  content  = "cache config v1"
}
```

Initialize and apply so we have a healthy baseline:

```bash
terraform init
terraform apply -auto-approve
```

Confirm the "infrastructure" and the state both exist:

```bash
ls resources/                         # cache.txt  database.txt  web-server.txt
```

---

## 1. `terraform state list` — the inventory

Print every resource address Terraform is tracking. This is read-only; it
touches nothing:

```bash
terraform state list
```

```
local_file.cache
local_file.db
local_file.web
```

These are **resource addresses** — Terraform's names for entries in the state
mapping, not cloud IDs. Everything else in this lab manipulates *these strings*
and the objects they point to, never the files themselves.

Inspect a single entry in detail with `state show`:

```bash
terraform state show local_file.web
```

```
# local_file.web:
resource "local_file" "web" {
    content              = "web server config v1"
    content_base64sha256 = "..."
    filename             = "./resources/web-server.txt"
    id                   = "..."
    ...
}
```

---

## 2. `terraform state mv` — rename a resource with NO infra change

Goal: rename `local_file.web` → `local_file.frontend`.

**First, see the wrong way.** If you just rename it in `main.tf` and plan,
Terraform has no idea the old and new addresses are the same object — it sees
one resource *gone* and one *new*:

```bash
# Rename the resource label in main.tf: local_file "web" -> local_file "frontend"
sed -i 's/resource "local_file" "web"/resource "local_file" "frontend"/' main.tf
terraform plan
```

```
Plan: 1 to add, 0 to change, 1 to destroy.
  # local_file.frontend will be created
  # local_file.web will be destroyed
```

That destroy+recreate is exactly what you must avoid on real infra. Cancel that
idea and tell Terraform the entry simply moved:

```bash
terraform state mv local_file.web local_file.frontend
terraform state list                  # now shows local_file.frontend
terraform plan
```

```
No changes. Your infrastructure matches the configuration.
```

The address changed, the plan is clean, and `web-server.txt` on disk was never
touched. **`state mv` moved the bookkeeping, not the resource.**

```bash
cat resources/web-server.txt          # still "web server config v1"
```

---

## 3. The modern alternative: a `moved {}` block

`state mv` is imperative — it runs on *your* machine and leaves no trace in the
repo, so a teammate applying from CI never learns the rename happened. Since
Terraform 1.1 the declarative replacement is a **`moved {}` block** that lives
in config, ships in the PR, and applies automatically.

Let's rename `local_file.db` → `local_file.primary_db` this way. Edit `main.tf`
so the resource is renamed **and** add a `moved` block:

```hcl
resource "local_file" "primary_db" {
  filename = "${path.module}/resources/database.txt"
  content  = "database config v1"
}

moved {
  from = local_file.db
  to   = local_file.primary_db
}
```

Now plan — Terraform reads the block and treats it as a move, not a
replacement:

```bash
terraform plan
```

```
  # local_file.db has moved to local_file.primary_db
Plan: 0 to add, 0 to change, 0 to destroy.
```

```bash
terraform apply -auto-approve
terraform state list                  # local_file.primary_db (no local_file.db)
```

No destroy, no `state mv` command, and the rename is now recorded in version
control. This is the preferred path for any refactor that ships through CI.
(You can delete the `moved` block a release or two later once everyone's state
has caught up.)

---

## 4. `terraform state rm` — forget a resource WITHOUT destroying it

Goal: hand `local_file.cache` off to another team's config. You want *your*
Terraform to stop managing it, but the resource must keep running.

```bash
terraform state rm local_file.cache
terraform state list                  # local_file.cache is gone from state
```

Now prove `state rm` did **not** delete anything real — the file is still on
disk:

```bash
cat resources/cache.txt               # still "cache config v1" — untouched!
```

Here's the subtle part. Your `main.tf` **still declares** `local_file.cache`,
but state no longer knows about it. So Terraform thinks it needs to be created:

```bash
terraform plan
```

```
  # local_file.cache will be created
Plan: 1 to add, 0 to change, 0 to destroy.
```

That plan is proof of the mechanic: `state rm` only removed the *mapping
entry*. The real object is orphaned (still exists, unmanaged). Because config
still references it, Terraform now wants to (re-)create it — it would happily
overwrite the existing file on the next apply.

**In the real handoff you'd also delete the resource block from your `.tf`** (or
use a `removed {}` block, next section) so Terraform doesn't try to recreate
what another config now owns. To restore a clean state here, put it back under
management:

```bash
terraform apply -auto-approve         # re-adopts cache into this state
terraform state list                  # cache, frontend, primary_db
```

---

## 5. The modern alternative: a `removed {}` block

Since Terraform 1.7, `removed {}` blocks are the declarative version of
`state rm` — you delete the `resource` block and add a `removed` block telling
Terraform to forget it *without* destroying it. To try it, remove the
`local_file.cache` resource block from `main.tf` and add:

```hcl
removed {
  from = local_file.cache

  lifecycle {
    destroy = false   # forget it, DO NOT destroy the real resource
  }
}
```

```bash
terraform plan
```

```
  # local_file.cache will no longer be managed by Terraform, but will not be
  # destroyed
Plan: 0 to add, 0 to change, 0 to destroy.
```

```bash
terraform apply -auto-approve
terraform state list                  # cache gone from state...
cat resources/cache.txt               # ...but the file still exists on disk
```

`destroy = false` is the critical line — it makes `removed` behave like
`state rm` (forget, don't delete). Set `destroy = true` and it would actually
destroy the resource. Like `moved`, this is code-reviewable and CI-friendly;
you delete the `removed` block once the apply has run everywhere.

---

## 6. Why not just hand-edit `terraform.tfstate`?

It's plain JSON, so it's tempting. Take a peek:

```bash
terraform state pull | head -20       # version, serial, lineage, resources...
```

Hand-editing risks: (1) corrupting the JSON, (2) forgetting to bump `serial`,
which the backend uses to detect concurrent writes, and (3) breaking `lineage`,
which ties a state to its history. The `state` subcommands do all of that
correctly and keep the file valid. Treat direct edits as a last resort.

---

## 7. Clean up

```bash
rm -rf /tmp/q209-state-demo
```

## What you should take away

- **`state list`** = read-only inventory of what Terraform manages. Always your
  first step before surgery.
- **`state mv`** = rename/relocate a state entry (resource rename, into/out of a
  module, `count`↔`for_each` reindex). It moves the *bookkeeping*, never the
  real resource — and skipping it after a rename causes a destroy+recreate.
- **`state rm`** = forget a resource. It **does not delete** the real object; it
  becomes orphaned/unmanaged. Contrast with `import` (adopt) and `destroy`
  (actually delete).
- Prefer **`moved {}`** (1.1+) and **`removed {}`** (1.7+) blocks for anything
  that ships through CI — declarative, reviewable, and self-applying. The CLI
  commands are the imperative escape hatch.
- These commands are far safer than hand-editing the JSON: they keep `serial`
  and `lineage` correct and the file valid.

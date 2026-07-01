# q219 — Hands-on lab: corrupt, then recover Terraform state

This lab runs entirely locally. No cloud credentials, no network beyond the
one-time `terraform init`. We simulate "cloud resources" with the `local_file`
provider (each file on disk is a stand-in for a real resource) and simulate the
"remote state corruption" by editing the local `terraform.tfstate`.

Works with `terraform` or `tofu` — substitute the binary name as needed. All
commands below were verified against Terraform 1.15.

---

## 0. Set up the working directory

```bash
mkdir -p /tmp/q219-state-recovery && cd /tmp/q219-state-recovery
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

# Each of these files stands in for a "cloud resource" (an EC2 instance,
# an S3 bucket, a database, whatever). We create three of them.
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
ls resources/                         # web-server.txt  database.txt  cache.txt
terraform state list                  # local_file.cache / .db / .web
```

---

## 1. Take a known-good backup (this is the whole game)

In the real world this is *bucket versioning doing it for you automatically*.
Here we do it by hand so you can see exactly what "the last good version" is.

```bash
cp terraform.tfstate terraform.tfstate.good-backup
```

> Note: Terraform *already* keeps a one-generation backup for you in
> `terraform.tfstate.backup` — it's the state as it was **before** the most
> recent write. We'll use that later too.

---

## 2. Trigger the disaster

**Layer 1 — the junior SRE deletes live resources.** Two of our three
"resources" get manually removed outside of Terraform:

```bash
rm resources/database.txt resources/cache.txt
ls resources/                         # only web-server.txt remains
```

**Layer 2 — CI writes a corrupt/truncated state file.** We clobber the state
so it's no longer valid JSON:

```bash
echo '{ "version": 4, "TRUNCATED' > terraform.tfstate
```

Now watch Terraform fail — it can't even parse state:

```bash
terraform plan
```

You get an error that looks like this (Terraform wraps parse failures in the
state-lock message, but the real cause is the two bullet points):

```
Error: Error acquiring the state lock
...
- Unsupported state file format: The state file could not be parsed as JSON:
  syntax error at byte offset 27.
- Unsupported state file format: The state file does not have a "version"
  attribute, which is required to identify the format version.
```

This is the two-layer failure: infra is partly gone **and** state is unusable.

---

## 3. Recover the state (Path A: you have a backup — the normal case)

Restore the last known-good state. In production this is
"restore the previous S3 object version" or `terraform state push` of a pulled
snapshot; here it's a file copy:

```bash
cp terraform.tfstate.good-backup terraform.tfstate
```

Sanity-check that state parses again and describes all three resources:

```bash
terraform state list                  # cache / db / web all listed again
```

Now let **Terraform tell you the damage**. Run a plan — the refresh step
reconciles state against the real world and shows what's missing:

```bash
terraform plan
```

Terraform reports it will **create** `local_file.db` and `local_file.cache`
(the two we deleted) and leave `local_file.web` untouched:

```
  # local_file.cache will be created
  # local_file.db will be created
Plan: 2 to add, 0 to change, 0 to destroy.
```

That plan is your damage report. Recreate the deleted resources:

```bash
terraform apply -auto-approve
ls resources/                         # all three files are back
terraform plan                        # "No changes." — fully reconciled
```

Recovery complete. Total time in a real incident: minutes — *because the state
backup existed*.

---

## 4. Recover with NO backup (Path B: `terraform import` fallback)

Now the painful path — nobody set up versioning and there is **no** usable
state anywhere. You must rebuild state from the surviving real infrastructure
with `terraform import`.

**Important real-world gotcha you'll hit immediately:** `import` only works for
resource types whose provider *implements* it, and you must supply the real
object's ID. The `local_file` resource we used above **does not implement
import** — so this section uses `terraform_data` (a built-in resource that
does) to demonstrate the mechanic. Not every resource is importable; know this
before an incident.

Set up a fresh directory:

```bash
mkdir -p /tmp/q219-import && cd /tmp/q219-import
cat > main.tf <<'EOF'
terraform {
  required_version = ">= 1.5"
}

# terraform_data is a built-in resource (no provider download needed) that
# supports import. Each one stands in for a real cloud resource.
resource "terraform_data" "web"   {}
resource "terraform_data" "db"    {}
resource "terraform_data" "cache" {}
EOF

terraform init
terraform apply -auto-approve
terraform state list                  # web / db / cache
```

Now simulate the catastrophe: **all state is gone**, and `cache` was truly
deleted for good, but `web` and `db` still exist in the real world:

```bash
rm -f terraform.tfstate terraform.tfstate.backup
terraform state list                  # empty — no state at all
```

Rebuild state for the **survivors** by importing them with their real IDs
(in a real provider these are ARNs, instance IDs, etc.):

```bash
terraform import terraform_data.web "web-real-id-12345"
terraform import terraform_data.db  "db-real-id-67890"
```

Because `cache` genuinely no longer exists, we do **not** import it — we let
Terraform recreate it:

```bash
terraform plan                        # plans to CREATE only terraform_data.cache
terraform apply -auto-approve
terraform state list                  # all three back under management
```

> Real-world caveat: `import` only rebuilds *state*. Your `.tf` config must
> already describe each resource you import, or the next plan will try to
> "correct" the imported object toward the HCL. Importing dozens of resources
> by hand is exactly the tedium that versioned remote state exists to prevent.

---

## 5. Bonus: the built-in `.tfstate.backup` and remote `state pull`/`push`

Terraform writes `terraform.tfstate.backup` automatically on every state
change — it's the previous generation, your zero-effort local safety net:

```bash
cd /tmp/q219-state-recovery
terraform apply -auto-approve         # make any change
ls -la terraform.tfstate.backup       # exists, one generation behind
```

With a *remote* backend you'd pull the authoritative copy to archive it before
touching anything, and push a corrected copy back if needed:

```bash
# terraform state pull > /tmp/state-snapshot-$(date +%s).json   # download
# ... inspect the snapshot, and only if you must overwrite remote state:
# terraform state push /tmp/state-snapshot-<ts>.json            # upload
```

`state pull`/`push` are the manual escape hatch for remote backends; treat
`push` as a loaded gun — it overwrites the remote state unconditionally.

---

## 6. Clean up

```bash
rm -rf /tmp/q219-state-recovery /tmp/q219-import
```

## What you should take away

- **State and infrastructure are two separate things** that failed together.
- Recovery order is **state first** (restore a good version), then **let
  `plan` compute the damage**, then **`apply`** to rebuild.
- `import` is the fallback when no state backup exists — slow, manual, and only
  available for resource types the provider supports.
- Everything easy about Path A comes from **versioning + locking** being set up
  *before* the incident.

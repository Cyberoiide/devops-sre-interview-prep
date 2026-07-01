# q201 — Hands-on lab: making a newly-added output actually show up

This lab runs entirely locally. No cloud credentials, no network beyond the
one-time `terraform init`. We use the `random` and `local_file` providers to
create a resource, then add a new `output` block that references that
already-existing resource — **without changing any infrastructure** — and watch
exactly which commands surface the new value and which don't.

Works with `terraform` or `tofu` — substitute the binary name as needed. All
commands below were verified against Terraform 1.x.

---

## 0. Set up the working directory

```bash
mkdir -p /tmp/q201-outputs-demo && cd /tmp/q201-outputs-demo
```

Create `main.tf`:

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
    local = {
      source  = "hashicorp/local"
      version = "~> 2.5"
    }
  }
}

# A resource that stands in for "some real cloud resource" (an instance, a
# bucket, a DB). It has attributes we'll want to expose as outputs later.
resource "random_pet" "server" {
  length    = 2
  separator = "-"
}

resource "local_file" "config" {
  filename = "${path.module}/server-config.txt"
  content  = "server name: ${random_pet.server.id}\n"
}
```

Now create `outputs.tf` with **one** output to start:

```hcl
output "server_name" {
  description = "The generated server name"
  value       = random_pet.server.id
}
```

Initialize and apply so we have a healthy, fully-applied baseline:

```bash
terraform init
terraform apply -auto-approve
```

At the end of that apply you'll see the first output printed:

```
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

Outputs:

server_name = "helped-mongoose"
```

Confirm it's readable from state:

```bash
terraform output                      # server_name = "helped-mongoose"
terraform output server_name          # "helped-mongoose"
```

---

## 1. Add a new output that references an EXISTING resource

Open `outputs.tf` and add a second output block. It references
`random_pet.server` and `local_file.config` — both of which **already exist**.
We are changing **no** resources, only exposing more of what's already there:

```hcl
output "server_name" {
  description = "The generated server name"
  value       = random_pet.server.id
}

# NEW — references resources that already exist. No infra change.
output "config_path" {
  description = "Absolute path to the rendered config file"
  value       = local_file.config.filename
}
```

Save the file. The infrastructure is untouched; we only edited config.

---

## 2. The gotcha: `terraform output` does NOT show it yet

Try to read the new output straight away:

```bash
terraform output config_path
```

You get an error — the output **isn't in state yet**:

```
╷
│ Error: Output "config_path" not found
│
│ The output variable requested could not be found in the state file. If you
│ recently added this to your configuration, be sure to run `terraform apply`,
│ since the state won't be updated with new output values until then.
╵
```

And the bare command still lists only the old one:

```bash
terraform output                      # only server_name — config_path missing
```

**This is the key lesson.** `terraform output` reads outputs *from the state
file*. Terraform even tells you the fix in the error message: outputs aren't
written to state until an apply. Editing `outputs.tf` alone does nothing to
state.

---

## 3. `terraform plan` previews it — but does NOT persist it

```bash
terraform plan
```

Terraform reports **no resource changes**, but flags the new output under a
dedicated section:

```
No changes. Your infrastructure matches the configuration.

Changes to Outputs:
  + config_path = "/tmp/q201-outputs-demo/server-config.txt"
```

That `+ config_path` line is a **preview**. Plan computed the value but wrote
nothing to state. Prove it — `terraform output` is still empty of the new one:

```bash
terraform output config_path          # still: Error: Output "config_path" not found
```

---

## 4. `terraform apply` persists and prints it (0 resource changes)

Run the normal apply. Because you only added an output referencing existing
resources, there are **zero** resource changes:

```bash
terraform apply -auto-approve
```

```
No changes. Your infrastructure matches the configuration.

Changes to Outputs:
  + config_path = "/tmp/q201-outputs-demo/server-config.txt"

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Outputs:

config_path = "/tmp/q201-outputs-demo/server-config.txt"
server_name = "helped-mongoose"
```

Note `0 added, 0 changed, 0 destroyed` — no infrastructure moved — yet the
apply **evaluated the new output, wrote it to state, and printed it**. Now it's
readable every way:

```bash
terraform output                      # both server_name AND config_path
terraform output config_path          # "/tmp/q201-outputs-demo/server-config.txt"
terraform output -json                # machine-readable, both outputs present
```

The `-json` form is what you'd use in scripts/CI:

```bash
terraform output -json config_path    # "/tmp/q201-outputs-demo/server-config.txt"
```

---

## 5. The alternative: `terraform apply -refresh-only`

You don't strictly need a full plan/apply cycle to write outputs to state. A
refresh-only apply reconciles state with real infrastructure and re-evaluates
outputs **without** planning any resource change. Add a *third* output to
demonstrate:

```hcl
output "server_id" {
  description = "The random_pet resource ID"
  value       = random_pet.server.id
}
```

Then:

```bash
terraform apply -refresh-only -auto-approve
terraform output server_id            # now present in state
```

`-refresh-only` explicitly promises it won't create, change, or destroy
resources — it only updates state (including outputs) to match reality. That
makes it a very safe way to get new outputs written when you're nervous about a
normal apply doing anything unexpected.

> Deprecated form: `terraform refresh` does the same thing, but as of Terraform
> 0.15.4 it's **deprecated** and is now just an alias for
> `terraform apply -refresh-only`. Prefer the explicit
> `apply -refresh-only`; reach for `terraform refresh` only in old docs/muscle
> memory, and know it's on the way out.

---

## 6. Clean up

```bash
rm -rf /tmp/q201-outputs-demo
```

## What you should take away

- **Outputs are persisted in the state file.** `terraform output` only *reads*
  that persisted copy — a brand-new `output` block is invisible to it until
  something writes it to state.
- **`terraform plan`** previews new outputs under `Changes to Outputs:` but
  persists nothing.
- **`terraform apply`** is the clean answer: adding an output that references
  existing resources yields `0 added, 0 changed, 0 destroyed`, yet apply still
  evaluates, writes, and prints the outputs.
- **`terraform apply -refresh-only`** is the safe alternative that updates
  state (including outputs) without touching resources. `terraform refresh` is
  the deprecated alias for it.

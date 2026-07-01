# q205 — Hands-on lab: force a resource to be recreated

This lab runs entirely locally. No cloud credentials, no network beyond the
one-time `terraform init`. We use the `random` provider to create a resource
whose value **visibly changes every time it's recreated**, and a `local_file`
that records that value to disk — so you can *prove* the resource was actually
rebuilt, not just re-read.

Works with `terraform` or `tofu` — substitute the binary name as needed. All
commands below were verified against Terraform 1.x.

---

## 0. Set up the working directory

```bash
mkdir -p /tmp/q205-taint-demo && cd /tmp/q205-taint-demo
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

# Stand-in for a "server". random_id generates a value ONCE at create time and
# keeps it stable across applies — so if its hex value changes, we KNOW the
# resource was destroyed and recreated (not just refreshed).
resource "random_id" "server" {
  byte_length = 8
}

# A file that records the server's identity. Because its content interpolates
# random_id.server.hex, recreating the server cascades a new value into here —
# a second, on-disk witness that recreation happened.
resource "local_file" "manifest" {
  filename = "${path.module}/server-manifest.txt"
  content  = "server id = ${random_id.server.hex}\n"
}
```

Initialize and apply to get a healthy baseline:

```bash
terraform init
terraform apply -auto-approve
```

Record the current server id — this is the value we'll watch:

```bash
terraform output 2>/dev/null; cat server-manifest.txt   # e.g. server id = 3f9a1c...
terraform state list                                    # local_file.manifest / random_id.server
```

---

## 1. The modern way: `terraform apply -replace=<addr>`

This is the recommended approach on any current Terraform. First **preview** it
— `-replace` is a plan-time flag, so you see exactly what will happen before
committing:

```bash
terraform plan -replace="random_id.server"
```

Terraform shows the resource being destroyed and recreated (`-/+`), with the
tell-tale `forces replacement`, and the manifest updating as a knock-on effect:

```
  # random_id.server must be replaced
-/+ resource "random_id" "server" {
      ~ hex         = "3f9a1c..." -> (known after apply)
        # (forces replacement)
    }
  # local_file.manifest will be updated in-place
Plan: 1 to add, 1 to change, 1 to destroy.
```

Note it did **not** modify state or the resource yet — this was just a plan.
Now actually do it:

```bash
terraform apply -replace="random_id.server" -auto-approve
cat server-manifest.txt                                 # server id = <A DIFFERENT VALUE>
```

The server id changed — proof the resource was genuinely destroyed and
recreated even though **nothing in `main.tf` changed**. A plain
`terraform plan` right after is clean:

```bash
terraform plan                                          # No changes.
```

---

## 2. The deprecated way: `terraform taint` / `terraform untaint`

`taint` gets you the same end result, but as a separate, state-mutating step.
Mark the resource:

```bash
terraform taint random_id.server
```

Terraform flips a flag in **state** and prints a deprecation warning like:

```
Resource instance random_id.server has been marked as tainted.
Warning: "terraform taint" is deprecated. Use the -replace option with
"terraform plan" or "terraform apply" instead.
```

Nothing in `main.tf` changed and nothing in the "cloud" (our files) changed yet
— only state was touched. But the *next* plan now shows the forced recreate:

```bash
terraform plan
```

```
  # random_id.server is tainted, so must be replaced
-/+ resource "random_id" "server" {
      ~ hex = "..." -> (known after apply)
    }
Plan: 1 to add, 1 to change, 1 to destroy.
```

### Undo it without recreating — `terraform untaint`

Suppose you tainted it by mistake. Clear the flag *before* applying and the
plan goes back to clean — no recreation happens:

```bash
terraform untaint random_id.server
terraform plan                                          # No changes.
```

Confirm the server id is still the same as after step 1:

```bash
cat server-manifest.txt                                 # unchanged — untaint saved us
```

### Or let the taint ride

Taint it again and apply to see it actually rebuild (the equivalent of step 1,
just via the deprecated two-step dance):

```bash
terraform taint random_id.server
terraform apply -auto-approve
cat server-manifest.txt                                 # server id changed again
```

---

## 3. Targeting one instance of a `count` / `for_each` resource

`-replace` (and `taint`) can target a single instance by its indexed address.
Add a fleet to `main.tf`:

```hcl
resource "random_id" "fleet" {
  count       = 3
  byte_length = 4
}
```

```bash
terraform apply -auto-approve
terraform state list        # ...random_id.fleet[0] / [1] / [2]
```

Replace only instance `[1]` and watch the other two stay put:

```bash
terraform apply -replace='random_id.fleet[1]' -auto-approve
```

```
Plan: 1 to add, 0 to change, 1 to destroy.   # only [1] is rebuilt
```

For a `for_each` resource the address uses the key instead of the index, e.g.
`terraform apply -replace='random_id.fleet["blue"]'`. Quote the address so your
shell doesn't eat the brackets or quotes.

---

## 4. Clean up

```bash
rm -rf /tmp/q205-taint-demo
```

## What you should take away

- **`taint` only writes to state.** It changes nothing in your HCL and nothing
  in the real world — it just flags the object so the *next* apply recreates it.
- **`terraform apply -replace=<addr>` is the modern, recommended replacement.**
  It's a plan-time flag: you *see* the `-/+ forces replacement` and approve it
  in one step, with no separate state-mutating command — much safer in CI.
- `terraform taint` / `untaint` are **deprecated since 0.15.2** and print a
  warning pointing you at `-replace`; `untaint` is still handy to *undo* a taint
  before you apply.
- Use an **indexed address** (`[1]`, `["blue"]`) to replace exactly one
  instance of a `count`/`for_each` resource.

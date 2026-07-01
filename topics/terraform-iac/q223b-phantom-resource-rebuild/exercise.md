# q223b — Hands-on lab: create a perpetual diff, then fix it

You'll manufacture a real "phantom rebuild" with the `null` and `time`
providers, watch Terraform refuse to converge, and then fix it two ways:
`ignore_changes` for the computed-value case and `set`-vs-`list` for the
ordering case.

Runs entirely locally, no cloud credentials. Verified against Terraform 1.15.
Substitute `tofu` for `terraform` if you use OpenTofu.

---

## 0. Set up

```bash
mkdir -p /tmp/q223b-phantom && cd /tmp/q223b-phantom
```

---

## Part 1 — Cause #1: a computed value that changes every plan

Create `main.tf`:

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    null = {
      source  = "hashicorp/null"
      version = "~> 3.2"
    }
  }
}

# BAD: timestamp() is re-evaluated on every plan, so this trigger is never
# equal to the value stored in state. Terraform will want to replace the
# resource on EVERY apply, forever.
resource "null_resource" "with_timestamp" {
  triggers = {
    always = timestamp()
  }
}
```

Apply it, then plan again **without changing anything**:

```bash
terraform init
terraform apply -auto-approve
terraform plan          # <-- watch this
```

You'll see the phantom rebuild even though you touched nothing:

```
  # null_resource.with_timestamp must be replaced
      ~ triggers = { # forces replacement
Plan: 1 to add, 0 to change, 1 to destroy.
```

Apply again and re-plan — it comes right back. **This is the perpetual diff.**
Read the diff carefully: the `~ triggers` line names the exact attribute
driving the churn. That reading skill is the whole point.

### Fix #1 — `ignore_changes`

Tell Terraform to stop reacting to changes on that attribute after creation:

```hcl
resource "null_resource" "with_timestamp" {
  triggers = {
    always = timestamp()
  }

  lifecycle {
    # After creation, ignore drift on triggers so the recomputed
    # timestamp no longer forces a replace.
    ignore_changes = [triggers]
  }
}
```

```bash
terraform apply -auto-approve
terraform plan          # -> "No changes. Your infrastructure matches..."
```

The diff is gone. (In a real case where you simply shouldn't be generating a
value non-deterministically, the *better* fix is to remove `timestamp()`
entirely or replace it with a `time_static` resource that only changes when you
want it to — `ignore_changes` is the right tool when the churn comes from the
*provider* returning a value you can't control.)

---

## Part 2 — Cause #3: list ordering churn

Reset to a fresh file. This time we show why an order-sensitive **list**
produces a phantom diff while a **set** does not. We use `terraform console`
so you can see the comparison semantics directly — no cloud resource needed:

```bash
# Note: piped `terraform console` reliably evaluates ONE expression per call,
# so run these as two separate commands (or type them in interactive console).
echo 'tolist(["rule-a","rule-c","rule-b"]) == tolist(["rule-a","rule-b","rule-c"])' | terraform console
echo 'toset(["rule-a","rule-c","rule-b"]) == toset(["rule-a","rule-b","rule-c"])'   | terraform console
```

Output:

```
false   # lists compare element-by-element IN ORDER -> "changed"
true    # sets are unordered -> identical contents compare equal
```

That single `false` is the entire mechanism behind "the security-group rules
keep showing a diff": the cloud API returned the rules in a different order
than your `.tf` lists them, and because the attribute is a **list**, Terraform
reports a change. Prove the normalization a set gives you:

```bash
echo 'jsonencode(["rule-a","rule-c","rule-b"])'              | terraform console
echo 'jsonencode(tolist(toset(["rule-a","rule-c","rule-b"])))' | terraform console
```

```
"[\"rule-a\",\"rule-c\",\"rule-b\"]"   # list keeps your (arbitrary) order
"[\"rule-a\",\"rule-b\",\"rule-c\"]"   # set normalizes -> stable order
```

### Fix #2 — use a set (or sort) so order stops mattering

When *you* control the type (e.g. a variable or a module input), declare it as
a set:

```hcl
variable "ingress_rules" {
  # type = list(string)   # BAD for order-insensitive data: churns on reorder
  type = set(string)      # GOOD: order is irrelevant, no phantom diff
  default = ["rule-a", "rule-c", "rule-b"]
}
```

When you *don't* control the type (a provider models the attribute as a list
but the API reorders it), the equivalents are:

- `sort(...)` the value so both sides land in the same deterministic order, or
- `lifecycle { ignore_changes = [that_attribute] }` if reordering is
  cosmetic and you manage the contents elsewhere.

---

## Part 3 — Separating "real drift" from "config problem" with `-refresh-only`

When you're not sure whether the real world actually changed or your config is
just wrong, `-refresh-only` updates state from reality and shows you **pure
drift** without proposing to change any infrastructure:

```bash
# Back in the Part 1 directory (or any applied config):
terraform plan  -refresh-only     # shows how state differs from real world only
terraform apply -refresh-only     # reconciles STATE to reality, no infra change
```

If a resource *stops* showing a diff after `apply -refresh-only`, the diff was
stale state, not a real change. If it *still* diffs afterward, the difference is
between your **config** and reality — a genuine config/normalization issue to
fix with the techniques above.

---

## 4. Clean up

```bash
rm -rf /tmp/q223b-phantom
```

## What you should take away

- A phantom diff is always **one of three moving parts**: a computed/dynamic
  value, provider normalization/defaults, or list ordering.
- **Read the `~`/`# forces replacement` lines** to find the exact attribute
  before you fix anything.
- Fix by cause: `ignore_changes` for uncontrollable computed values, `set`/
  `sort` for ordering, deliberate provider pinning for normalization drift.
- `-refresh-only` is the diagnostic that tells drift apart from config error.
- `ignore_changes` is a scalpel, not a mute button — it also stops Terraform
  from correcting *real* unwanted changes to that attribute, so scope it
  tightly.

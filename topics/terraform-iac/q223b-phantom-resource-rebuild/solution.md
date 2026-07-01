# q223b — Solution & walkthrough

## The mental model: a plan is a three-way diff

Terraform decides what to do by comparing three things for every attribute:

1. **Config** — what your `.tf` says the value should be.
2. **Prior state** — what Terraform recorded last time.
3. **Refreshed real value** — what the provider reports the object *currently*
   looks like (the refresh step reads the live API).

A diff appears when these disagree. A **phantom** diff — one that recurs even
though no human changed anything — means one of those three moved on its own.
Nailing "which one moved, and why" is the whole skill. There are three classic
culprits.

## Cause 1 — Dynamic / computed / server-generated attributes

Some values are simply not stable across runs:

- Your config generates one non-deterministically: `timestamp()`,
  `uuid()`, a `random_*` value that isn't held stable.
- The provider recomputes one every read: a `last_modified` time, an
  auto-incrementing version, an ARN/ID suffix the cloud regenerates.

State stores the value from last apply; the next plan produces a *different*
value; Terraform dutifully proposes to change (often **replace**) the resource.
The lab's `triggers = { always = timestamp() }` is the pure form of this — the
plan even prints `# forces replacement` next to `triggers`.

**Fix:**
- If *you* introduced the non-determinism, remove it — don't put `timestamp()`
  where you didn't need a fresh value; use a `time_static` / `random_*` resource
  that only changes when its inputs change.
- If the *provider* returns a value you can't control,
  `lifecycle { ignore_changes = [attr] }` tells Terraform to stop reacting to
  post-creation changes on that attribute.

## Cause 2 — Provider-side changes and normalization

The config and state didn't change, but the **provider's behavior** did:

- A **provider upgrade** changed a default value, so the refreshed object now
  differs from what your (unchanged) config implies.
- The provider changed how it **normalizes/returns** a value: it now lowercases
  an ARN, canonicalizes/reorders keys in a JSON IAM policy, expands a CIDR
  shorthand, or adds a default field. Your config's literal string no longer
  matches the normalized form the API returns, so every plan shows a diff.

This is why "it started diffing right after we bumped the AWS provider" is such
a common report.

**Fix:**
- Treat provider versions as deliberate: pin them in `required_providers` and
  read the provider's CHANGELOG on upgrade.
- Align your config to the normalized form (e.g. write the ARN lowercased, use
  `jsonencode` so the provider and you agree on JSON shape, or use the
  provider's canonicalizing data source like `aws_iam_policy_document`).
- Where the normalization is cosmetic and unavoidable, `ignore_changes` the
  attribute.

## Cause 3 — List vs. set ordering

An attribute typed as a **list** is **order-significant**: `["a","c","b"]` is
*not* equal to `["a","b","c"]`. Many cloud APIs return collections
(security-group rules, tags, subnets, routes) in an order that differs from how
you wrote them. Because the attribute is a list, Terraform sees "position 1
changed from c to b" and reports a diff — even though the *set of rules* is
identical. Apply "fixes" the order, the API returns its own order again, and you
get a perpetual diff.

A **set** has no order, so the same contents always compare equal. The lab
proves it directly in `terraform console`:

```
tolist(["a","c","b"]) == tolist(["a","b","c"])  => false
toset(["a","c","b"])  == toset(["a","b","c"])   => true
```

**Fix:**
- When you own the type (variable, module input, or a provider attribute that
  accepts either), use `set(...)` so order is irrelevant.
- When the provider hard-codes a list you can't retype, `sort()` the value so
  both sides are deterministically ordered, or `ignore_changes` it and manage
  contents through a dedicated resource (e.g. separate
  `aws_security_group_rule` resources instead of inline `ingress` blocks).

## How to debug *which* cause you're hitting

1. **Read the plan output literally.** The `~ attribute = ... -> ...` lines and
   any `# forces replacement` tell you the *exact* attribute. Half the battle
   is just not skimming past it.
2. **Check recent changes to the provider**, not just to your code. `git log`
   the lockfile (`.terraform.lock.hcl`) and `required_providers`. A phantom diff
   that appeared "out of nowhere" is very often a provider bump.
3. **Use `-refresh-only` to isolate drift from config error:**
   - `terraform plan -refresh-only` shows only how *state* differs from the real
     world — i.e. real drift, with no proposed infra changes.
   - `terraform apply -refresh-only` reconciles state to reality. If the
     phantom diff disappears afterward, it was stale state. If it *persists*,
     the mismatch is between your **config** and reality → cause 1, 2, or 3.
4. **Turn on `TF_LOG=DEBUG`** (or `TRACE`) to see the raw values the provider
   returns during refresh when the plan output isn't specific enough.

## Choosing the fix (don't reach for one hammer)

| Symptom | Likely cause | Right fix |
|--------|--------------|-----------|
| A timestamp/uuid/version attribute always differs | Computed/dynamic value | Remove the non-determinism, or `ignore_changes` if provider-driven |
| Started after a `terraform init -upgrade` / provider bump | Provider normalization/default | Pin provider, align config to normalized form, read CHANGELOG |
| A rules/tags/subnets *collection* churns but contents are identical | List ordering | Use `set`, `sort`, or split into per-element resources |
| Not sure if reality actually changed | Any | `plan/apply -refresh-only` to isolate drift |

## Why `ignore_changes` is a scalpel, not a mute button

`ignore_changes` genuinely stops the phantom diff, but it *also* stops Terraform
from correcting **real** unwanted changes to that attribute — if someone edits
it in the console for real, Terraform will now happily ignore it. So scope it to
the single attribute (never `ignore_changes = all` unless you truly mean
"adopt whatever exists"), and prefer removing the root cause (don't generate the
value non-deterministically; use a set; align to the normalized form) when you
can. Suppress the diff only when the churn is genuinely outside your control.

## The 45-second version to say out loud

> "A plan is a three-way diff between config, state, and the refreshed real
> value, so a phantom rebuild means one of those moved on its own. Three usual
> suspects: a computed/dynamic attribute like a timestamp or auto-generated ARN;
> a provider change that altered a default or how it normalizes a value, usually
> after an upgrade; or a list attribute whose element order the API returns
> differently even though the contents match. I read the plan to find the exact
> attribute, use `plan -refresh-only` to tell real drift from a config mismatch,
> and fix by cause — `ignore_changes` for uncontrollable computed values, a
> `set` or `sort` for ordering, and deliberate provider pinning plus config
> alignment for normalization. I use `ignore_changes` sparingly because it also
> masks real changes to that field."

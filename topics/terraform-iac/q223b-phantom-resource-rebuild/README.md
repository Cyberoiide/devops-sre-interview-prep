# q223b — The phantom rebuild: `apply` wants to recreate a resource nobody touched

## The scenario (as an interviewer would pose it)

> "You run `terraform plan` on a stable environment and Terraform wants to
> **recreate** — or at least modify — a resource that you are certain nobody
> changed. The code hasn't changed in git. Nobody edited it in the cloud
> console. Yet every single plan shows the same diff, and if you apply it, the
> next plan shows it *again*. It's a perpetual diff.
>
> Why does this happen? Give me the common causes, how you'd debug which one
> you're hitting, and how you'd fix it properly rather than just applying
> repeatedly."

## What the interviewer is really probing

1. Do you understand that a Terraform diff is a **three-way comparison** —
   config (HCL) vs. prior state vs. the *refreshed* real-world value — and that
   a "phantom" diff means one of those three moved without a human?
2. Can you name the classic causes and, crucially, **tell them apart**?
3. Do you know the right tool for each fix — `ignore_changes`, `sets vs lists`,
   `-refresh-only`, provider upgrades — rather than reaching for one hammer?
4. Can you read a `terraform plan` diff carefully enough to see *which
   attribute* is driving the change?

## The three classic causes

1. **Dynamic / computed / server-generated attributes.** A value that the
   config sets to something non-deterministic (`timestamp()`, a random value)
   or that the provider recomputes each run (a version string, a
   last-modified time, an auto-generated ARN suffix). State stores one value,
   the next plan computes a different one → perpetual diff.

2. **Provider-side changes / normalization.** A provider upgrade changes a
   default, or changes how it **returns/normalizes** a value — e.g. it now
   lowercases an ARN, reorders a JSON policy's keys, or expands a shorthand.
   Your config says one form, the refreshed API returns another, so Terraform
   sees a difference it will "fix" forever.

3. **Ordering of list vs. set attributes.** An attribute modeled as a **list**
   is order-sensitive. If the API returns security-group rules (or tags, or
   subnets) in a different order than your config lists them, Terraform sees a
   diff even though the *contents* are identical. The same data in a **set**
   would compare equal.

## The short answer

Read the plan to find the *specific attribute* driving the diff, then match it
to a cause and fix:

- Non-deterministic/computed value you don't control → **`lifecycle {
  ignore_changes = [...] }`** (or stop generating it non-deterministically).
- Provider normalization/default drift → **pin/upgrade the provider
  deliberately**, align your config to the normalized form, or `ignore_changes`
  the normalized attribute.
- List-order churn → **use a set** (or sort deterministically) so order stops
  mattering.
- To separate "did the *real world* drift?" from "is my config wrong?", use
  **`terraform plan -refresh-only`** / `terraform apply -refresh-only`.

The lab **creates** a real perpetual diff using the `null` provider and the
built-in `timestamp()` function (plus a reordered list demonstrated in
`terraform console`), lets you watch it never converge, then fixes it with
`ignore_changes` and by switching a list to a set.

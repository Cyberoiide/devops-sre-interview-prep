# q223b — Interviewer follow-up questions

---

**Q: Explain the three-way comparison Terraform does during a plan.**

For each attribute Terraform compares (1) the desired value in your **config**,
(2) the value recorded in **prior state**, and (3) the **refreshed** value the
provider reports from the real world. Refresh updates the state's view of
reality; then the plan diffs config against that refreshed state. A change is
proposed wherever config and refreshed reality disagree. A phantom diff means
one of the three moved without a human editing your HCL.

---

**Q: What's the difference between `ignore_changes` and `-refresh-only`?**

`ignore_changes` is a **permanent, config-level** instruction: after a resource
is created, Terraform won't propose changes to the listed attributes ever
again. `-refresh-only` is a **one-off command mode**: it updates state to match
the current real world and shows drift *without* proposing any infrastructure
changes. Use `-refresh-only` to *diagnose* (is this real drift or a config
mismatch?); use `ignore_changes` to *suppress* an attribute you've decided
Terraform shouldn't manage.

---

**Q: Is `ignore_changes = all` ever a good idea?**

Rarely. It tells Terraform to ignore drift on *every* attribute after creation,
so the resource is effectively "create once, then never reconcile" — you lose
drift detection entirely and can't even see when someone changes it out of
band. It's occasionally used for adopt-and-forget resources or when another
system owns the object's lifecycle, but for normal resources it hides exactly
the problems Terraform exists to catch. Prefer listing specific attributes.

---

**Q: A perpetual diff appeared right after a provider upgrade. How do you confirm and fix it?**

Confirm by checking `.terraform.lock.hcl` / `required_providers` history
(`git log`) to see the version jumped, and read the provider CHANGELOG for the
attribute in the diff — look for "now normalizes/defaults/returns" notes. Fix by
either aligning your config to the new normalized form, pinning back to the old
version if the change is a regression (and reporting an issue), or
`ignore_changes` on the attribute if the difference is purely cosmetic. The key
insight is to suspect the *provider*, not your code.

---

**Q: How would you turn inline security-group rules that keep reordering into a stable config?**

Two options. (1) Model them as separate `aws_security_group_rule` /
`aws_vpc_security_group_ingress_rule` resources instead of inline `ingress`
blocks — each rule is its own resource, so order between them is irrelevant. (2)
If you must keep them inline, ensure the attribute is a set (many are) or sort
the input deterministically. Splitting into discrete resources is the more
robust, order-immune approach and is AWS's own recommendation.

---

**Q: `terraform plan` shows a diff but you believe the real resource is fine. Prove which side is wrong.**

Run `terraform apply -refresh-only`. That pulls the current real values into
state without changing infrastructure. Re-run `terraform plan`: if the diff is
now gone, the earlier diff was **stale state** (something changed in reality and
state hadn't caught up). If the diff **remains**, it's a genuine disagreement
between your **config** and reality — so it's a config/normalization/ordering
issue to fix, not stale state.

---

**Q: When is a recreate (`-/+`) unavoidable vs. an in-place update (`~`)?**

It's the provider's schema that decides: attributes marked `ForceNew` can't be
changed in place, so any diff on them forces destroy-and-recreate. In-place
`~` updates are for mutable attributes. If a phantom diff is forcing a *replace*
(as the lab's `triggers` does), you're diffing a `ForceNew` attribute — which
raises the stakes, because a phantom *replace* of a stateful resource can cause
real downtime or data loss. That urgency is why you fix the root cause rather
than repeatedly applying.

---

**Q: How can `create_before_destroy` interact with this?**

If a phantom diff forces replacement, the default order is destroy-then-create,
which causes an outage window. `lifecycle { create_before_destroy = true }`
flips that so the new resource comes up before the old is torn down. It doesn't
*fix* the phantom diff — you still want to eliminate the churn — but it reduces
the blast radius while a genuine forced replacement happens. Don't use it as a
substitute for finding out *why* the replace keeps happening.

---

**Q: Could `terraform refresh` help here, and why is the standalone command discouraged?**

`terraform refresh` updates state from reality, similar to the refresh step of a
plan. The standalone command is deprecated/discouraged because it mutates state
with **no plan and no approval** — you can't review what it's about to change.
The safe equivalent is `terraform apply -refresh-only`, which shows you the drift
and asks for confirmation before writing state. Same effect, with a review gate.

---

**Q: How do you catch phantom diffs before they hit production?**

Run `terraform plan` in CI on every PR *and* on a schedule (drift detection),
and fail/alert on unexpected non-empty plans in environments that should be
stable. Pin provider versions and gate upgrades behind a plan review. A
scheduled plan that suddenly goes non-empty with no code change is your early
warning of either real drift or a provider-behavior change — exactly the
phantom-diff signal.

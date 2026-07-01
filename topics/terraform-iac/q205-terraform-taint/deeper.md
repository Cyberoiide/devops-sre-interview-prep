# q205 — Interviewer follow-up questions

Concise, defensible answers to the questions a good interviewer asks after your
first response.

---

**Q: Precisely which of the three — HCL, state, or the real resource — does `terraform taint` modify?**

Only **state**, and only a single flag on one object instance. Your `.tf`
configuration is untouched, and the real resource is untouched at the moment you
run `taint`. The command writes "this instance is tainted" into the state file.
The actual destroy/create doesn't happen until the *next* `apply` — that's when
the real resource is finally affected. This separation (mark now, act later) is
exactly the thing `-replace` collapses back into a single reviewed step.

---

**Q: If `taint` and `-replace` produce the same result, why was `taint` deprecated?**

Because `taint` mutates state as a side effect *before* you see any plan. That's
two problems: the destroy/create is invisible until a later plan someone has to
remember to run and read, and if you never apply, state is left dirty for the
next person. `-replace` is a plan-time flag — the replacement shows up in the
plan you're already reviewing, in one step, with no lingering state change. That
makes it auditable and safe for pipelines, so HashiCorp deprecated `taint`/
`untaint` in 0.15.2.

---

**Q: How do you force recreation of just one instance of a resource created with `count` or `for_each`?**

Target the indexed address. For `count`: `terraform apply
-replace='aws_instance.web[2]'`. For `for_each`:
`terraform apply -replace='aws_instance.web["blue"]'`. Quote the whole address
so the shell doesn't interpret the brackets or double-quotes. The other
instances are left alone — the plan will show `1 to add, 1 to destroy` for just
that index.

---

**Q: `terraform taint` vs `terraform state rm` — what's the difference?**

`taint` keeps the resource under management and schedules a **destroy +
recreate** on the next apply. `state rm` **forgets** the resource entirely —
Terraform stops tracking it, the real resource is left running untouched, and
the next plan will try to *create* a new one (because config still wants it),
potentially leaving an orphan. `taint` = "rebuild this"; `state rm` = "stop
managing this." Different intents entirely.

---

**Q: Is there a purely declarative way to get the same effect, so it's in code review?**

Yes, and it's usually preferable:

- `lifecycle { replace_triggered_by = [...] }` (Terraform 1.2+) recreates a
  resource when a referenced resource/attribute changes.
- On `random_*` resources, the `keepers` map: change a value in it and the
  random resource regenerates.
- Changing any attribute the provider marks `ForceNew` (an immutable field)
  triggers replacement.

These live in HCL, so they show up in a diff and a review, unlike the imperative
`-replace`/`taint`. Reach for `-replace` only for genuine one-off "rebuild this
now" cases.

---

**Q: You ran `terraform taint` but haven't applied yet, and realize it was a mistake. How do you back out?**

`terraform untaint <address>` clears the flag from state, and the next plan is
clean again — no recreation happens. This is the one place `untaint` is still
genuinely useful. If you'd used `-replace` on a *plan* instead, there'd be
nothing to undo: you just don't apply that plan. Either way, the resource is
never actually touched until an apply runs.

---

**Q: Does `-replace` guarantee create-before-destroy so there's no downtime?**

No. `-replace` just schedules a replacement; the *ordering* of destroy vs create
is governed by the resource's `lifecycle { create_before_destroy = true }`
setting, exactly as it would be for a config-driven replacement. Without that,
Terraform destroys first, then creates — so plan for a gap. If you need
zero-downtime replacement, the resource must already be configured for
create-before-destroy (and the provider/resource has to support it).

---

**Q: Can you `-replace` several resources in one apply?**

Yes — pass the flag multiple times: `terraform apply
-replace="random_id.a" -replace="random_id.b"`. Each named address is treated as
needing replacement in that run, and the plan shows all of them. With the old
workflow you'd `taint` each one separately, mutating state once per command,
which is exactly the multi-step fragility `-replace` removes.

---

**Q: How do you wire a forced replacement into a CI/CD pipeline safely?**

Two-stage, artifact-based: `terraform plan -replace="<addr>" -out=tfplan`, gate
on human approval of the printed diff (which clearly shows the `-/+ forces
replacement`), then `terraform apply tfplan` applies exactly what was reviewed.
Never bake `taint` into a pipeline — it writes state out-of-band before anyone
sees a plan, so the actual destroy/create is invisible at approval time. The
plan-time nature of `-replace` is precisely why it's automation-safe.

---

**Q: After `-replace`, `terraform plan` still wants to replace the resource on the next run. What's wrong?**

The replacement flag itself is one-shot — it doesn't persist. If a subsequent
plan *still* wants to recreate the resource, that's not `-replace` at all;
you've got a real perpetual-diff problem: an attribute the provider treats as
`ForceNew` differs between config and state (often a provider bug, a computed
value written back oddly, or a genuinely-changed immutable field). Investigate
with `terraform plan` output showing which attribute `forces replacement`, and
`terraform state show <addr>` to compare. That's a config/provider issue to fix,
not something taint or replace should paper over.

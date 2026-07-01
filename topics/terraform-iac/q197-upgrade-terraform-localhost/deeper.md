# q197 — Interviewer follow-up questions

Concise, defensible answers to the questions a good interviewer asks after your
first response.

---

**Q: Why can't I just downgrade Terraform if the upgrade goes wrong?**

Because the state format only moves forward. The first time a newer core writes
state, it may bump the format version; an older binary then refuses to read that
file. So a "downgrade" isn't just swapping the binary — you also need the
pre-upgrade state back. That's exactly why the runbook backs up
`terraform.tfstate` (or relies on versioned remote state) *before* the first run
of the new version. With the backup you can restore the old-format state and the
old binary works again; without it, you're stuck upgrading everything forward.

---

**Q: What does `required_version` actually do, and what happens if the new binary violates it?**

It's a constraint in the `terraform {}` block, e.g. `required_version = ">=
1.5.0, < 2.0.0"`. Before doing almost anything, Terraform checks its own version
against it and **hard-errors** if it doesn't match — it won't init, plan, or
apply. So if you install a binary outside the constraint, the fix is to either
widen the constraint in config (and commit that) or install a version that
satisfies it. It's the mechanism that stops someone on the wrong core from
silently operating on a project that assumes a different one.

---

**Q: Difference between upgrading the Terraform binary and running `terraform init -upgrade`?**

Two separate things. Installing a new binary (brew / tfenv / manual) upgrades
**core** — the `terraform` program itself. `terraform init -upgrade` upgrades
**providers** — it re-resolves each provider to the newest version allowed by
your `required_providers` constraints and rewrites `.terraform.lock.hcl`. Plain
`terraform init` (no `-upgrade`) respects the existing lock file and won't move
providers. You typically do the core upgrade first, then `init -upgrade` to pull
providers along.

---

**Q: What is `.terraform.lock.hcl` and should I commit it?**

It's the dependency lock file: it pins the exact provider versions and their
cryptographic hashes (per platform) that were selected. **Yes, commit it.** It's
what makes provider resolution reproducible — teammates and CI running `init`
get byte-identical providers instead of "whatever's newest today." `init
-upgrade` is the command that intentionally updates it. If you hit hashes for a
platform your CI uses but your laptop doesn't, `terraform providers lock
-platform=...` adds them.

---

**Q: `tfenv` vs `tenv` — which and why?**

`tfenv` is the long-standing, Terraform-only version manager. `tenv` is the
modern successor written in Go: same workflow (`tenv tf install/use/list`), reads
the same `.terraform-version` files, but also manages **OpenTofu, Terragrunt, and
Atmos** from one tool and is the more actively maintained project. If you only
ever touch Terraform, `tfenv` is fine and battle-tested; if you also run OpenTofu
or Terragrunt, `tenv` consolidates everything. Either beats hand-managing
binaries.

---

**Q: How do you pin a version per project, and why bother?**

Drop a `.terraform-version` file in the repo root (`echo "1.9.8" >
.terraform-version`). Both `tfenv` and `tenv` auto-select that version when you
`cd` into the directory. You bother because it eliminates "works on my machine"
version skew: everyone on the team, plus CI, resolves the same core without any
manual coordination. It pairs with `required_version` (which *enforces* a range)
— `.terraform-version` picks the exact one to run.

---

**Q: You upgraded and now `terraform plan` shows a diff on resources nobody changed. What's going on?**

After a pure version bump, an unexpected diff is almost always one of: a
**provider behavior change** (the newer provider represents an attribute
differently, or added a default), or **drift** that existed already and the
refresh is now surfacing it. Use `terraform plan -refresh-only` to isolate pure
drift, read the provider's CHANGELOG for the versions `init -upgrade` moved
through, and inspect specific resources with `terraform state show`. Don't
`apply` a surprise diff until you understand it.

---

**Q: How do you handle a big version jump, like 1.3 to 1.13?**

Cautiously and incrementally. Rather than one leap, step through minors — 1.3 →
1.4 → 1.5 → ... — so each version's official upgrade guide applies as written
(they assume you came from the immediately prior minor). At each step you read
the notes, `init -upgrade`, and `plan` to confirm the config and state are still
coherent before moving on. It's slower but each hop is a small, understandable
diff instead of one giant unaudited change. For the state format specifically,
each hop's write may bump it — so keep backups at each step.

---

**Q: How is this different in CI versus on your laptop?**

Same runbook, but CI should **pin** rather than "upgrade": the pipeline installs
an explicit version (from `.terraform-version`, a `setup-terraform` action input,
or a container tag) so builds are reproducible and don't drift when a new release
lands. You upgrade CI deliberately by bumping that pinned version in a reviewed
PR — ideally the *same* PR that bumps `.terraform-version` and the committed
`.terraform.lock.hcl`, so local and CI move together. Remote state versioning is
also non-negotiable in CI since that's where the format-upgrade risk actually
bites the team.

---

**Q: Is upgrading OpenTofu any different?**

No, the shape is identical: record version, read notes, check `required_version`,
back up state, install, verify, `init -upgrade`, `plan`. Install options differ
slightly — `tenv tofu install`, a package manager, or the `get.opentofu.org`
script instead of HashiCorp's releases site. The one thing to watch is
**cross-tool state compatibility** if you're migrating between Terraform and
OpenTofu, but a straight OpenTofu-to-newer-OpenTofu upgrade behaves just like a
Terraform upgrade.

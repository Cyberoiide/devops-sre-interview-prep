# q197 — Solution & walkthrough

## Framing: an upgrade is a change, not a reflex

The weak answer is "`brew upgrade terraform`, done." The senior answer treats
the binary upgrade as a change with blast radius on two artifacts you care
about: your **config** (does the new core still accept it?) and your **state**
(can it still be read afterward — including by teammates and CI who haven't
upgraded yet?). The whole runbook exists to protect those two things.

The single most important sentence to say out loud:

> **Terraform upgrades the state format on write, and older binaries can't read
> the upgraded file — it's one-way. So I back up (or rely on versioned remote)
> state before the first run of any newer binary.**

## The runbook

### Step 1 — Record the current version

```bash
terraform version
```

This is your baseline and rollback target. It conveniently prints three things
at once: the **core** version, the **provider** versions in use, and an
"out of date" notice with the latest available release. Note core and providers
separately — they upgrade independently.

### Step 2 — Read the CHANGELOG and upgrade guide

Before downloading anything, read the release notes / upgrade guide for the
target minor and scan the CHANGELOG for the range between your version and the
target. You're hunting for **breaking changes**: removed or deprecated features
(e.g. `terraform taint`/`untaint` giving way to `-replace`, HCL syntax
deprecations), backend changes, and provider-protocol shifts. For a large gap,
upgrade **one minor at a time** — each minor's upgrade guide is written assuming
you came from the previous one, so stepping through them keeps the guidance
accurate and the diffs small.

### Step 3 — Check version constraints

```bash
grep required_version main.tf
terraform providers
```

The `required_version` in the `terraform {}` block is a hard gate — if the new
binary doesn't satisfy it, Terraform refuses to run until you either widen the
constraint or pick a satisfying version. Check `required_providers` constraints
too; they govern how far `init -upgrade` can move providers in step 6.

### Step 4 — Back up state first (the gotcha)

```bash
cp terraform.tfstate terraform.tfstate.pre-upgrade-backup   # local
# remote: rely on S3/GCS/Azure object versioning, which you should already have
```

This is the step juniors skip. A newer core writes state in a newer format; an
older core then errors out reading it. If the upgrade misbehaves and you need to
fall back, the backup (or a previous remote object version) is the only thing
that lets the old binary read state again. Also relevant on a team: if you push
newer-format state to a shared backend, **everyone else must upgrade too** —
the format bump is contagious.

### Step 5 — Know your providers

```bash
terraform providers        # dependency tree
terraform version          # resolved provider versions
```

So you know what you're carrying forward before you touch anything.

### Step 6 — Install the new binary

Three ways, in order of preference:

- **Version manager (recommended).** `tfenv install <ver>` then `tfenv use
  <ver>`; or the modern `tenv tf install <ver>` / `tenv tf use <ver>`. Keeps
  many versions side by side, switches per project.
- **macOS Homebrew.** `brew upgrade hashicorp/tap/terraform` (or `brew upgrade
  terraform`).
- **Manual (Linux/Windows).** Download the zip from `releases.hashicorp.com`,
  verify the checksum, `unzip`, `chmod +x`, and move it over the binary on your
  `PATH` (e.g. `/usr/local/bin/terraform`).

### Step 7 — Verify

```bash
terraform version          # shows the new core version
```

### Step 8 — Bump providers within constraints

```bash
terraform init -upgrade    # moves providers to newest allowed by constraints
                           # and rewrites .terraform.lock.hcl
```

`init -upgrade` is what actually moves **providers** (the binary upgrade in step
6 only moved core). It respects your `required_providers` constraints and
records the resolved versions + hashes in `.terraform.lock.hcl`. **Commit that
lock file** — it's how the team and CI get byte-identical provider resolution.

### Step 9 — Plan

```bash
terraform plan
```

`No changes.` means the new core parsed your config, read/upgraded state, and
found no drift. A surprise diff after a pure version bump is a signal — provider
behavior change or drift — to investigate before applying.

## Version managers: why they're the real answer

Manually swapping binaries is fine once; it's misery across five projects each
pinned to a different core. A version manager fixes that:

- **`tfenv`** — the classic, Terraform-only manager. `tfenv install`,
  `tfenv use`, `tfenv list`, `tfenv list-remote`.
- **`tenv`** — the modern, actively maintained successor. Same UX
  (`tenv tf install`, `tenv tf use`, ...) but also manages **OpenTofu,
  Terragrunt, and Atmos** from one tool, and reads the same version files.

The per-project pin is a **`.terraform-version`** file in the repo root:
`echo "1.9.8" > .terraform-version`. When you `cd` into the project, the manager
auto-selects that version. Commit it, and the whole team plus CI runs the exact
same core with no coordination.

## OpenTofu

If you're on OpenTofu, the runbook is identical in shape: `tofu version` →
read notes → check `required_version` → back up state → install (via `tenv tofu
install`, the package manager, or the `get.opentofu.org` script) → `tofu
version` → `tofu init -upgrade` → `tofu plan`. Same state-format and lock-file
cautions.

## The 30-second version to say out loud

> "First `terraform version` to record where I am, then read the CHANGELOG and
> upgrade guide for breaking changes — stepping one minor at a time for a big
> jump. I check the `required_version` and provider constraints, then — the key
> bit — back up state, because a newer core upgrades the state format one-way
> and older binaries can't read it after. I install with a version manager,
> `tfenv` or the newer `tenv`, and pin the project with a `.terraform-version`
> file so the whole team matches. Then `terraform version` to confirm core,
> `terraform init -upgrade` to move providers and rewrite the committed
> `.terraform.lock.hcl`, and finally `terraform plan` to prove there's no
> unexpected diff. OpenTofu is the same runbook."

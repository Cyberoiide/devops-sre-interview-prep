# q197 — Upgrading Terraform on your local machine

## The scenario (as an interviewer would pose it)

> "Your team is still on an older Terraform — say something in the 1.5 range —
> and you've been asked to move everyone to the current release. Start with your
> own laptop. Walk me through, step by step, how you'd upgrade Terraform locally
> **without breaking the project you're working on**. What do you check *before*
> you download anything, what's the actual install, and what's the one mistake
> that turns a routine upgrade into a genuinely bad day?"

## What the interviewer is really probing

1. Do you treat the binary upgrade as a **change with blast radius**, not a
   `brew upgrade` reflex — reading release notes and the `required_version`
   constraint first?
2. Do you know the **state-format gotcha**: newer Terraform can silently upgrade
   the state format, and older binaries then can't read it. That's a one-way
   door, and it's the answer to "the one mistake."
3. Are you fluent with the mechanics — `terraform version`, `terraform
   providers`, `terraform init -upgrade`, and what the `.terraform.lock.hcl`
   file is for?
4. Do you distinguish **the Terraform binary** from **the providers**? They
   version and upgrade independently.
5. Do you reach for a **version manager** (`tfenv`/`tenv`) rather than juggling
   binaries by hand, and can you pin per-project with `.terraform-version`?

## The short answer

Upgrading Terraform is a short **runbook**, not a one-liner:

1. **Record where you are** — `terraform version` (shows core + provider
   versions, and whether a newer release exists).
2. **Read the CHANGELOG / upgrade guide** for breaking changes between your
   version and the target. For big jumps, step one minor at a time.
3. **Check constraints** — the `required_version` in your `terraform {}` block
   must accept the new binary (or you bump it), and check provider constraints.
4. **Back up state first.** Newer Terraform *upgrades* the state format on write,
   and older binaries can't read the upgraded file — one-way. Remote state
   should be versioned; back up local `terraform.tfstate` before running
   anything new.
5. **Install the new binary** — via a version manager (`tfenv install <ver>` /
   `tenv tf install`), `brew upgrade`, or a manual download-unzip-replace on
   Linux/Windows.
6. **Verify** — `terraform version` shows the new number.
7. **Bump providers within constraints** — `terraform init -upgrade` updates
   providers and rewrites `.terraform.lock.hcl`. Commit the lock file.
8. **`terraform plan`** — confirm state still parses and there's no unexpected
   diff.

The strong recommendation is a **version manager**: `tfenv` (classic,
Terraform-only) or `tenv` (modern successor, also does OpenTofu / Terragrunt /
Atmos). A `.terraform-version` file pins each project so the whole team runs the
same core. The hands-on lab does all of this locally with no cloud — install
via `tfenv`, pin with `.terraform-version`, and prove a real config still
`init -upgrade`s and `plan`s clean.

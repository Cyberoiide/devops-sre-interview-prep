# q209 — Terraform state surgery: `state list`, `state mv`, `state rm`

## The scenario (as an interviewer would pose it)

> "You've inherited a Terraform config that's grown organically. You need to do
> three things without any downtime and without destroying live
> infrastructure: (1) get a clean inventory of everything Terraform currently
> manages, (2) **rename** a resource — `aws_instance.web` should become
> `aws_instance.frontend` — and also reorganize a couple of resources into a
> module, and (3) **hand off** a resource to a different team's config so *your*
> Terraform stops managing it, but the resource keeps running in production.
>
> Which state subcommands do you reach for, what does each one actually do to
> the real infrastructure, and — importantly — what does each one *not* do?
> And if you were shipping these changes through CI in 2026, would you still
> use the CLI commands?"

## What the interviewer is really probing

1. Do you know that **state is just Terraform's mapping** between config
   addresses and real-world objects — and that `state list`, `state mv`, and
   `state rm` edit *that mapping*, not the cloud?
2. Can you cleanly articulate the difference between **`mv`** (rename/relocate
   an entry), **`rm`** (forget an entry), and **`import`** (adopt an existing
   object into state)?
3. Do you avoid the classic exam trap — believing `state rm` **deletes** the
   real resource (it does not) or that `state mv` **moves** real infra (it does
   not)?
4. Do you know the *modern, declarative* replacements: **`moved {}`** blocks
   (Terraform 1.1+) instead of `state mv`, and **`removed {}`** blocks
   (Terraform 1.7+) instead of `state rm` — and why they're preferred for
   anything that ships through code review / CI?
5. Do you understand *why* these commands are safer than hand-editing the JSON
   (they keep `serial`/`lineage` correct and the file valid)?

## The short answer

All three operate on the **state mapping only** — never on live infrastructure:

- **`terraform state list`** — prints every resource address Terraform tracks.
  Your read-only inventory; start here.
- **`terraform state mv <src> <dst>`** — **renames or relocates** a state entry
  (rename a resource, move it in/out of a module, reindex when switching
  `count` ↔ `for_each`). The real object is untouched; you're only changing the
  address Terraform files it under. Skipping this after a rename makes Terraform
  see a *deleted* old resource and a *new* one → **destroy + recreate**.
- **`terraform state rm <addr>`** — tells Terraform to **forget** a resource.
  It stops being managed but **keeps running** in the cloud (orphaned). Use it
  to hand a resource to another config/state, or before a re-`import`.

**Modern equivalents:** put a `moved {}` block in your config instead of running
`state mv`, and a `removed {}` block instead of `state rm`. They're declarative,
reviewable in a PR, and apply automatically through CI — the CLI commands are
the imperative escape hatch for local/one-off surgery.

The hands-on lab reproduces all of this locally with `local_file` resources (no
cloud needed): you'll list state, rename via both `state mv` **and** a
`moved {}` block, and `state rm` a resource to prove the file on disk survives.

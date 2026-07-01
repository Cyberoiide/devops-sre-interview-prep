# q209 — Solution & walkthrough

## Restating the problem precisely

Three tasks, all of which must happen **without downtime and without destroying
live infrastructure**:

1. **Inventory** everything Terraform manages.
2. **Rename** `aws_instance.web` → `aws_instance.frontend`, and reorganize some
   resources into a module.
3. **Hand off** a resource to another team's config so your Terraform stops
   managing it while the resource keeps running.

The single most important idea to say out loud:

> **Terraform state is a *mapping* between the addresses in your config and the
> real objects in the cloud. `state list`, `state mv`, and `state rm` edit that
> mapping — they do not create, move, or destroy real infrastructure.**

If you internalize that, every answer falls out of it: renaming is a mapping
edit, a handoff is a mapping deletion, and neither should ever touch the running
resource.

## The three commands, precisely

### `terraform state list` — inventory (read-only)

Prints every resource address in state:

```bash
terraform state list
terraform state list module.network      # filter to a module
terraform state show aws_instance.web    # inspect one entry in full
```

This is always step one before any surgery — you can't safely move or remove
what you haven't confirmed is there. It touches nothing.

### `terraform state mv` — rename / relocate an entry

```bash
terraform state mv <source-address> <destination-address>
```

`state mv` changes the **address** an object is filed under. The real resource
is never touched. Common uses:

- **Rename a resource:** `state mv aws_instance.web aws_instance.frontend`.
- **Move into a module:** `state mv aws_instance.web module.compute.aws_instance.web`.
- **Move out of a module:** the reverse.
- **Reindex when switching `count` ↔ `for_each`:**
  `state mv 'aws_instance.web[0]' 'aws_instance.web["primary"]'`.

**Why it matters:** Terraform identifies resources by address. If you rename the
resource label in HCL but *don't* tell Terraform the address moved, it sees the
old address as deleted and the new one as new → a **destroy + recreate**. For an
EC2 instance or an RDS database that's an outage or data loss. `state mv` (or a
`moved` block) tells Terraform "same object, new name," so the plan comes back
clean.

### `terraform state rm` — forget an entry (the exam-trap command)

```bash
terraform state rm <address>
```

`state rm` removes the mapping entry. The real resource **keeps running** — it
becomes *orphaned*: it still exists in the cloud, but no Terraform config
manages it. This is the command people misremember; write it on your mental
flashcard:

> **`state rm` does NOT delete the real resource. `terraform destroy` does.**

Use cases:

- **Hand a resource off** to another config/state (this scenario's task 3).
- **A resource is now managed elsewhere** and you want to stop the duplicate
  management.
- **Before a re-`import`** — remove a stale/wrong entry, then `import` the real
  object cleanly.

**Critical follow-through:** after `state rm`, if your `.tf` *still declares* the
resource, the next `plan` will want to **create** it again (you saw this in the
lab). So a real handoff is two steps: `state rm` **and** delete the resource
block from your config (or use a `removed` block, which does both intents at
once).

## `mv` vs `rm` vs `import` — the mental model

| Command | Effect on state mapping | Effect on real infra |
|---|---|---|
| `state mv` | renames/relocates an entry | **none** |
| `state rm` | deletes an entry | **none** (resource orphaned) |
| `import` | adds an entry for an existing object | **none** (adopts what's there) |
| `destroy` / apply a delete | removes the entry after… | **destroys** the real resource |

The first three are pure bookkeeping. Only the last one touches the cloud. `mv`
and `rm` are inverses of nothing in particular; `import` is the true inverse of
`rm` (rm forgets, import adopts).

## The modern, declarative equivalents (Terraform 1.1+ / 1.7+)

The CLI commands are imperative: they run on one machine and leave no record in
the repo. In a 2026 CI-driven workflow you usually want the refactor to be
**declarative, reviewable, and self-applying**:

- **`moved {}` block (Terraform 1.1+)** — replaces `state mv`:

  ```hcl
  moved {
    from = aws_instance.web
    to   = aws_instance.frontend
  }
  ```

  Commit it with the rename; every `apply` (yours, a teammate's, CI's) performs
  the move automatically. It's visible in code review — a huge advantage over a
  `state mv` someone ran locally and forgot to mention. Also works for moving
  into/out of modules and reindexing.

- **`removed {}` block (Terraform 1.7+)** — replaces `state rm`:

  ```hcl
  removed {
    from = aws_instance.cache
    lifecycle {
      destroy = false   # forget it; do NOT destroy the real resource
    }
  }
  ```

  You delete the `resource` block and add this. `destroy = false` is what makes
  it behave like `state rm` (forget, keep the resource). `destroy = true` would
  actually destroy it.

**When to still use the CLI:** genuine one-off local surgery, recovery
scenarios, or a quick fix where round-tripping a PR is overkill. For anything
that ships through the pipeline, prefer the blocks.

## Why these commands beat hand-editing the JSON

State *is* JSON, and you *can* open it — but the `state` subcommands are far
safer because they:

- keep the file **valid** (no fat-fingered braces / truncation),
- correctly bump **`serial`** (the counter backends use to detect concurrent /
  stale writes),
- preserve **`lineage`** (the ID tying a state to its history), and
- go through **locking** on remote backends so you don't race another apply.

Hand-edit only as an absolute last resort, and pull a backup first
(`terraform state pull > backup.json`).

## Why the lab models it faithfully

- `local_file` resources = cloud resources; the files on disk = the real
  objects.
- Renaming the label + `state mv` (and the `moved` block) demonstrates the
  destroy-vs-clean-plan difference exactly as it happens on EC2/RDS.
- `state rm` then `cat`-ing the surviving file proves the resource is orphaned,
  not deleted — the core exam trap, shown empirically.
- The follow-up `plan` wanting to *create* the removed resource proves config
  still references it, motivating the "also delete the block / use `removed`"
  step.

## The 30-second version to say out loud

> "All three edit Terraform's *mapping*, never the cloud. `state list` is my
> read-only inventory. `state mv` renames or relocates an entry — resource
> rename, into/out of a module, `count`↔`for_each` reindex — with zero infra
> change; skip it after a rename and you get a destroy+recreate. `state rm`
> makes Terraform *forget* a resource without deleting it — it keeps running,
> just unmanaged — which is exactly how I'd hand one off to another team, paired
> with deleting the config block. `state rm` is not `destroy`; that's the trap.
> And in 2026 I'd ship the rename as a `moved {}` block and the handoff as a
> `removed {}` block with `destroy = false` — declarative, code-reviewed, and
> applied automatically by CI, instead of running CLI commands nobody can see in
> the diff."

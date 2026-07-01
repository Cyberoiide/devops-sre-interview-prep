# q209 — Interviewer follow-up questions

Concise, defensible answers to the questions a good interviewer asks after your
first response.

---

**Q: Does `terraform state rm` delete the real resource in the cloud?**

No — and this is the single most common misconception. `state rm` only removes
the resource's *entry* from Terraform state. The real object keeps running; it
just becomes **orphaned** (unmanaged). Terraform will no longer plan changes to
it. If you actually want to delete the resource, that's `terraform destroy` (or
removing it from config and applying). `state rm` is the tool for *stopping
management*, not for deletion.

---

**Q: I ran `state rm` but the next `terraform plan` wants to create the resource again. Why?**

Because your `.tf` config still *declares* that resource. State no longer knows
the object exists, but config says it should — so Terraform plans to create it
(and would overwrite/duplicate the orphaned real object on apply). The fix: when
you `state rm` to hand a resource off, also **delete the resource block from
your config** (or use a `removed {}` block, which expresses both intents at
once). `state rm` alone is only half the handoff.

---

**Q: What's the difference between `state rm` and `import`?**

They're inverses. `state rm` **removes** an entry from state (Terraform forgets
an object it was managing). `import` **adds** an entry for an object that already
exists in the cloud (Terraform adopts something it wasn't managing). Neither
touches the running resource. A classic combo is `state rm` a stale/wrong entry,
then `import` the real object cleanly to fix a broken mapping.

---

**Q: Why does renaming a resource in HCL without a `state mv` (or `moved` block) cause a destroy + recreate?**

Terraform tracks resources by their **address** (e.g. `aws_instance.web`). When
you change the label to `aws_instance.frontend`, Terraform sees the old address
missing from config → plans to destroy it, and a brand-new address present →
plans to create it. It has no way to know they're the same object unless you
tell it. `state mv old new` or a `moved { from = ...; to = ... }` block supplies
that link, so the plan comes back clean with zero changes.

---

**Q: When would you use `state mv` versus a `moved {}` block?**

Same effect (relocate a state entry), different ergonomics. Use a **`moved`
block** for any refactor that ships through version control / CI: it's
declarative, shows up in code review, and applies automatically for everyone —
no one has to remember to run a command. Use **`state mv`** for genuine one-off
local surgery, recovery work, or quick fixes where a PR round-trip is overkill.
The block is the default in a modern pipeline; the CLI is the escape hatch.

---

**Q: Show me the exact `state mv` syntax for reindexing a `count` resource to `for_each`.**

You move each indexed instance to its keyed equivalent, quoting the addresses so
the shell doesn't eat the brackets:

```bash
terraform state mv 'aws_instance.web[0]' 'aws_instance.web["primary"]'
terraform state mv 'aws_instance.web[1]' 'aws_instance.web["secondary"]'
```

`count` addresses use numeric indices (`[0]`); `for_each` uses string keys
(`["primary"]`). Getting this right is what lets you switch from `count` to
`for_each` — a common refactor to stop churn when a middle element is removed —
without destroying and recreating every instance.

---

**Q: How do `moved` blocks handle moving a resource into or out of a module?**

Same block, just fully-qualified addresses:

```hcl
moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.web
}
```

Terraform relocates the state entry under the module path with no infra change.
This is how you safely extract a chunk of root-module resources into a reusable
module without a destroy/recreate storm. The equivalent CLI is
`terraform state mv aws_instance.web module.compute.aws_instance.web`.

---

**Q: What does the `removed {}` block's `lifecycle { destroy = false }` actually control?**

It chooses between the two things "remove from config" could mean.
`destroy = false` = **forget only** — Terraform drops the resource from state and
leaves the real object running (equivalent to `state rm`). `destroy = true` =
**actually destroy** the real resource on apply. Because the default intent of
deleting a resource block is normally to destroy it, `removed` blocks make you
state the safe intent explicitly. For a handoff you always want
`destroy = false`.

---

**Q: Is it ever OK to just hand-edit `terraform.tfstate`?**

Almost never, and only as a last resort with a backup pulled first
(`terraform state pull > backup.json`). The risks: invalid JSON, a stale
**`serial`** (backends use it to detect concurrent/stale writes, so a wrong
value can cause lost updates or push rejections), and a broken **`lineage`**.
The `state` subcommands do all of this bookkeeping correctly and, on remote
backends, respect locking. If you think you need to hand-edit, there's usually a
`state mv`/`rm`/`import`/`replace-provider` command that does it safely instead.

---

**Q: You need to run `state mv`/`rm` against a remote backend that's shared by a team. Any precautions?**

Yes. These commands acquire the state **lock** and write a new version, so:
(1) coordinate so no one is mid-`apply`; (2) with S3/GCS/azurerm you get locking
automatically, but confirm it's configured; (3) pull a backup first
(`terraform state pull`); and (4) prefer `moved`/`removed` blocks precisely
because they go through the normal plan/apply + PR flow, which serializes the
change and records it, rather than an out-of-band local mutation the rest of the
team can't see.

---

**Q: After a `state mv`, do you leave the `moved` block in the config forever?**

You can, and it's harmless — it just becomes a no-op once every state has the
new address. Common practice is to keep it for a release or two so any stale
state (a long-lived branch, a workspace that hasn't applied yet) still gets
migrated, then delete it in a later cleanup. The same goes for `removed` blocks:
keep until every environment has applied, then remove.

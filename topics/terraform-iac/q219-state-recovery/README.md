# q219 — Disaster recovery: deleted resources + corrupted state

## The scenario (as an interviewer would pose it)

> "It's a bad afternoon. A junior SRE was cleaning up what they *thought* was a
> dev environment and manually deleted several live cloud resources from the
> console. At almost the same time, a broken CI pipeline wrote a truncated
> `terraform.tfstate` to your remote backend — so the state file is now
> corrupt. So you have a **two-layer** problem: the real infrastructure is
> partly gone, **and** Terraform's record of what should exist is unusable.
>
> Walk me through how you recover. What do you do first, what tools do you
> reach for, and how would you have set things up so this was a five-minute fix
> instead of an all-nighter?"

## What the interviewer is really probing

1. Do you understand that **state is a separate artifact from the real
   infrastructure**, and that this is *two* independent failures that happen to
   coincide?
2. Do you know that a properly configured remote backend (S3 + versioning, or
   equivalent) means the state file is almost never truly lost — you restore a
   previous version?
3. Are you fluent with the state-manipulation commands (`state pull`,
   `state push`, `import`, `force-unlock`) and do you know when each applies?
4. Do you reach for `terraform import` as the fallback when *no* backup of
   state exists?
5. Can you articulate the prevention story: versioning, locking,
   least-privilege IAM, and backups?

## The short answer

The recovery hinges on one realization: **your remote state bucket is almost
certainly versioned.** So the fix order is:

1. **Stabilize** — stop all pipelines and human `apply`s so nothing writes more
   garbage. Break any stale lock.
2. **Recover the state**, not the infrastructure, first. Restore the last
   known-good version of the state file from bucket versioning (S3 object
   versions, or `.tfstate.backup`, or `terraform state pull` from a good
   snapshot).
3. **Let Terraform tell you the damage.** Run `terraform plan`. Because state
   now describes the intended world and the real world is missing resources,
   the plan shows *exactly* what needs to be rebuilt.
4. **`terraform apply`** to recreate the deleted resources.
5. If there is **no** state backup anywhere, fall back to `terraform import` to
   rebuild state resource-by-resource from whatever real infrastructure
   survived, then plan/apply.

The hands-on lab reproduces all of this on a local backend (no cloud needed):
you'll corrupt state, recover it from a backup copy, and also practice the
no-backup `import` path.

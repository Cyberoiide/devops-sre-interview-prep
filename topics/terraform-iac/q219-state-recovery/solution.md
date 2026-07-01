# q219 — Solution & walkthrough

## Restating the problem precisely

Two independent failures happened at the same time:

1. **Infrastructure failure** — real cloud resources were manually deleted.
2. **State failure** — the `terraform.tfstate` file in the remote backend is
   corrupt (truncated/invalid).

The trap in the question is that these *feel* like one catastrophe, but they
are separate artifacts with separate recovery mechanisms. The single most
important sentence you can say in the interview is:

> **Terraform state is a description of what *should* exist; it is not the
> infrastructure itself. Recover the state first, then let Terraform compute
> and rebuild the delta.**

If you try to "fix the infrastructure" before you have a trustworthy state
file, you're flying blind — you don't even have an authoritative list of what
was supposed to exist.

## The recovery runbook

### Step 0 — Stabilize and stop the bleeding

- **Freeze all writes.** Pause CI/CD pipelines and tell humans to stop running
  `apply`. A second bad write while you're recovering turns a bad afternoon
  into a bad week.
- **Handle the lock.** With remote backends (S3 + DynamoDB, GCS, azurerm,
  Terraform Cloud) a crashed run can leave a stale lock. If a lock is genuinely
  orphaned, clear it with `terraform force-unlock <LOCK_ID>` — but only after
  you've confirmed nothing is actually running. Never `force-unlock` a live
  operation.
- **Preserve evidence.** Copy the corrupt state aside
  (`terraform state pull > corrupt-$(date +%s).json` or download the object)
  before you overwrite anything. You may need it for the postmortem, and it
  occasionally contains recoverable fragments.

### Step 1 — Recover the state (not the infra) first

This is where a properly configured backend turns disaster into a chore.

- **Restore from bucket versioning.** Production remote state on S3 should have
  **S3 Object Versioning enabled**. Every write creates a new object version;
  the previous good version is still there. List versions and restore the last
  good one:

  ```bash
  aws s3api list-object-versions --bucket my-tf-state \
    --prefix env/prod/terraform.tfstate
  aws s3api get-object --bucket my-tf-state \
    --key env/prod/terraform.tfstate \
    --version-id <LAST_GOOD_VERSION_ID> restored.tfstate
  ```

  GCS (object versioning), Azure Blob (soft delete / snapshots), and Terraform
  Cloud (automatic state version history in the UI) all have equivalents.

- **Push the good state back** (or, for local, just copy it into place):

  ```bash
  terraform state push restored.tfstate
  ```

- **Fallback within this step:** the local working copy or CI runner may still
  have `terraform.tfstate.backup` — Terraform's automatic one-generation
  backup written before the last change. That, or any developer's recent
  `state pull`, can serve as the known-good snapshot.

### Step 2 — Let Terraform compute the damage

With trustworthy state restored, run:

```bash
terraform plan
```

The refresh step reconciles the restored state (which lists all resources that
*should* exist) against the real world (where several are now missing).
Terraform prints exactly what it will **create** — that plan **is your damage
report.** You didn't have to guess which resources the junior deleted;
Terraform diffed it for you.

### Step 3 — Rebuild

```bash
terraform apply
```

Terraform recreates the deleted resources and leaves the survivors alone.
Re-run `terraform plan` afterward; a clean "No changes" means state and reality
are back in sync.

### Step 4 — The no-backup fallback: `terraform import`

If — and only if — there is genuinely **no** recoverable state anywhere (no
versioning, no `.tfstate.backup`, no `state pull` snapshot on any machine), you
rebuild state from the surviving real infrastructure:

```bash
terraform import <resource.address> <real-world-id>
```

You do this once per surviving resource, using the actual cloud ID (ARN,
instance-id, etc.). Resources that were genuinely deleted are *not* imported —
you let a subsequent `apply` recreate them. Caveats:

- Import populates **state only**; your `.tf` must already describe the
  resource or the next plan will try to "correct" it.
- Not every resource type supports import (in the lab, `local_file` does not
  but `terraform_data` does — the same unevenness exists across real
  providers).
- For many resources, use `import` blocks (Terraform 1.5+) or tools like
  `terraformer` to generate config, but it's still slow and error-prone. This
  pain is the argument *for* the prevention measures below.

## Why the lab models it faithfully

- `local_file` resources = cloud resources; deleting the files = the manual
  deletion.
- Overwriting `terraform.tfstate` with garbage = the corrupt remote state.
- The `good-backup` copy = the S3 previous object version.
- `terraform_data` + `import` = the no-backup rebuild path.

The lab proves the key behavior: after restoring state, `terraform plan`
autonomously identifies the two deleted resources and nothing else.

## Prevention (the part that separates senior from junior answers)

1. **Remote state with versioning.** S3 + `versioning { enabled = true }` (or
   GCS/Azure/TFC equivalent). This alone reduces "corrupt state" from a
   disaster to a one-command restore.
2. **State locking.** DynamoDB lock table for S3 (`dynamodb_table` /
   the newer S3-native lockfile), or the backend's built-in locking. Prevents
   two concurrent applies from racing and producing the corrupt state in the
   first place.
3. **Least-privilege IAM + guardrails.** The junior should not have had console
   delete rights on production. Use separate accounts/projects per environment,
   SCPs / deny policies, and mandatory change review. Delete protection (e.g.
   `prevent_destroy` lifecycle, RDS deletion protection, S3 MFA-delete) on
   critical resources.
4. **Backups of state and data.** Enable bucket versioning *and* periodic state
   backups to a separate location; back up the actual data stores (DB
   snapshots) so "recreate the resource" doesn't mean "lose the data."
5. **CI hygiene.** Pipelines should `plan` → review → `apply`, never write state
   from a half-failed job. Fail closed, not open.
6. **Least surprise:** small blast radius (state split per environment/service),
   so one bad actor or one bad job can't corrupt everything.

## The 30-second version to say out loud

> "Two separate failures. State first: production state lives on S3 with
> versioning, so I restore the last good object version (or `.tfstate.backup`,
> or a `state pull` snapshot) and `state push` it back. Then `terraform plan`
> shows me exactly which resources were deleted, and `apply` rebuilds them. If
> there were truly no state backup anywhere, I'd `terraform import` the
> survivors resource-by-resource and let apply recreate the rest. And it's a
> five-minute fix instead of an all-nighter *because* we'd set up versioning,
> DynamoDB locking, and least-privilege IAM beforehand."

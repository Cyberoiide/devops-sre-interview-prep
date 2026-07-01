# q219 — Interviewer follow-up questions

Concise, defensible answers to the questions a good interviewer asks after your
first response.

---

**Q: What's the difference between `terraform.tfstate.backup` and S3 object versioning?**

`terraform.tfstate.backup` is a single, local, one-generation backup that
Terraform writes automatically *before* each state-modifying operation — it
only ever holds the immediately-previous state, and it lives wherever the
command ran. S3 object versioning is server-side, keeps *every* historical
version indefinitely (until lifecycle rules prune them), and survives the loss
of any single machine. You rely on `.tfstate.backup` for "undo my last apply"
and on bucket versioning for real disaster recovery.

---

**Q: Walk me through the exact S3 + DynamoDB backend setup and what each piece does.**

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "my-tf-locks"   # lock table (classic approach)
    encrypt        = true
  }
}
```

- The **S3 bucket** stores the state object; **versioning** on that bucket is
  what makes recovery trivial. **`encrypt = true`** ensures server-side
  encryption because state can contain secrets.
- The **DynamoDB table** provides the lock: before a write, Terraform puts a
  lock item keyed on the state path; concurrent runs fail fast instead of
  racing and corrupting state. (Terraform 1.10+ / OpenTofu also support
  S3-native lockfiles via `use_lockfile = true`, letting you drop DynamoDB.)

---

**Q: The state file contains secrets. How do you protect it?**

Enable SSE (KMS-preferably) on the bucket, restrict the bucket policy to the
CI role and a small admin group, block public access, and enable versioning +
access logging. Treat the state bucket as a secrets store. Also prefer keeping
secrets out of state where possible (e.g. reference them from a secrets
manager) — but Terraform inevitably records some sensitive values, so the
backend must be locked down regardless.

---

**Q: You restored state but `terraform plan` now shows changes to resources nobody touched. What happened?**

Likely drift or a version skew: the restored state is slightly older than the
real world (someone made a legitimate change between the good version and the
corruption), or the provider version changed how attributes are represented.
Use `terraform plan -refresh-only` to see pure drift, `terraform apply
-refresh-only` to reconcile state to reality without changing infrastructure,
and inspect individual resources with `terraform state show`. See q223b for the
full "perpetual diff" treatment.

---

**Q: When would you use `terraform state rm` during recovery?**

When state references a resource that no longer exists and you do **not** want
Terraform to recreate it (e.g. it was intentionally deleted). `state rm` drops
it from state without touching any real infrastructure, so the next plan won't
try to manage it. It's also used to hand a resource off to a different state or
to break a bad reference. Contrast with `import`, which *adds* an existing real
resource into state.

---

**Q: What's the danger of `terraform state push`?**

It overwrites the remote state unconditionally with whatever you give it —
there's no merge. If you push a stale or wrong snapshot, you can make Terraform
forget resources (leading it to recreate duplicates) or think deleted resources
still exist. Always pull and inspect first, keep the current remote copy
archived, and push only a snapshot you've verified. Terraform checks lineage
and serial numbers and will warn on mismatches; don't blindly `-force`.

---

**Q: How do you prevent the junior-SRE deletion in the first place?**

Least privilege: no standing console delete permissions on prod; separate
AWS accounts per environment with SCPs; require changes to go through the
Terraform pipeline with review. Add resource-level guardrails:
`lifecycle { prevent_destroy = true }` on critical resources, RDS/EBS deletion
protection, S3 MFA-delete. And reduce blast radius by splitting state per
service/environment so one mistake can't cascade.

---

**Q: `prevent_destroy` — does it protect against the console deletion in this scenario?**

No. `prevent_destroy` only stops *Terraform* from destroying the resource
during a plan/apply; it does nothing about someone deleting via the cloud
console or API directly. It's a guardrail against fat-fingered `terraform
destroy`, not against out-of-band deletion. Protection against console deletes
comes from IAM/SCP restrictions and provider-side delete protection.

---

**Q: How would this differ with Terraform Cloud / Enterprise as the backend?**

TFC stores state with automatic, browser-visible version history and one-click
rollback, handles locking for you, and encrypts state at rest — so the
"restore last good version" step becomes a UI click. You'd also get run
history showing who applied what. The recovery *logic* is identical; the tooling
just makes step 1 (state restore) point-and-click instead of `aws s3api`.

---

**Q: How do you make sure this recovery procedure actually works before you need it?**

Game days / DR drills: periodically restore state from a version into a
scratch workspace and run `plan` to confirm it's coherent. Test that your
pipeline can detect and refuse a corrupt state. Document the runbook (the one
in solution.md) and rehearse `force-unlock`, version restore, and `import` so
that during a real incident it's muscle memory, not improvisation.

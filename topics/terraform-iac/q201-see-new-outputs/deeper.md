# q201 — Interviewer follow-up questions

Concise, defensible answers to the questions a good interviewer asks after your
first response.

---

**Q: Why are outputs stored in state at all? Why not just recompute them every time?**

Because outputs are part of the persisted contract of a stack, and other things
depend on reading them cheaply and offline. `terraform output` (and remote
state consumers like a `terraform_remote_state` data source) must be able to
read output values **without** re-evaluating the whole config or contacting
providers. Storing the computed values in state makes `terraform output` a fast,
credential-free, side-effect-free read. The cost is exactly the gotcha in this
question: a newly-added output isn't reflected in state until you apply.

---

**Q: Concretely, does `terraform output` ever touch the provider or the cloud?**

No. `terraform output` reads the outputs section of the current state file and
prints it. It does not refresh, does not call any provider, and does not need
cloud credentials. That's precisely why it can't see a brand-new output block —
it isn't looking at your config, only at stored state.

---

**Q: How does another stack consume these outputs, and does the "not in state yet" problem affect that too?**

Via a `terraform_remote_state` data source (or `tofu`'s equivalent), which reads
the *root-level* outputs from the producer stack's state backend:

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-tf-state"
    key    = "network/terraform.tfstate"
    region = "eu-west-1"
  }
}

# usage
subnet_id = data.terraform_remote_state.network.outputs.subnet_id
```

Yes, it's affected the same way: the consumer reads *persisted* state, so a new
output in the producer is only visible after the producer has been applied.
Only root outputs are exported — module outputs must be re-exposed at the root
to be readable across stacks.

---

**Q: If I `terraform plan` and it shows `Changes to Outputs:` but `No changes` to resources, is running apply risky?**

No. That combination means the only thing apply will do is update the outputs in
state — `0 to add, 0 to change, 0 to destroy` on resources. It won't recreate or
modify infrastructure. If you want an explicit guarantee, use
`terraform apply -refresh-only`, which by design cannot plan resource changes.

---

**Q: Why is `terraform refresh` deprecated, and what replaced it?**

`terraform refresh` used to update state to match real infrastructure as a
standalone command with no plan/approval step, which was considered dangerous
(it silently mutated state). Since Terraform 0.15.4 it's deprecated and is now
an alias for `terraform apply -refresh-only`, which shows you the state changes
and asks for approval before writing them. Prefer `apply -refresh-only` (or
`plan -refresh-only` to just preview drift).

---

**Q: I marked an output `sensitive = true`. How does that change what these commands show?**

`plan` and `apply` redact the value in their human-readable output, printing
`(sensitive value)` instead. But it's still stored in **plaintext in the state
file**, and `terraform output <name>` will print the real value for that
specific output (you have to name it explicitly). `terraform output -json` also
emits the real value with a `"sensitive": true` marker. So `sensitive` is about
avoiding accidental console/log leakage, not about encryption — protect the
state backend regardless.

---

**Q: Does adding an output ever cause a resource change or replacement?**

No. An `output` block is a read-only projection of already-computed values; it
is not a resource and has no lifecycle. Adding, changing, or removing an output
only ever shows up under `Changes to Outputs:` and never causes a resource to be
created, updated, or destroyed. (Removing one just deletes it from state on the
next apply.)

---

**Q: The output references a value that's only known after apply (e.g. a not-yet-created resource). What does plan show?**

`(known after apply)`. If the referenced attribute isn't yet in state — because
the resource itself is being created in the same run — plan can't compute the
output value and displays `+ my_output = (known after apply)`. Once apply
creates the resource and computes the attribute, the output is resolved and
written to state.

---

**Q: Is there any way to see an output value without writing it to state at all?**

Yes, two ways. `terraform plan` prints the computed value under `Changes to
Outputs:` without persisting it. And `terraform console` lets you evaluate the
expression interactively — e.g. type `local_file.config.filename` — reading from
current state and config without modifying anything. Both are read-only; neither
makes `terraform output` return the value afterward.

---

**Q: In CI, what's the right way to grab an output for a downstream step?**

Run an apply first (the outputs must be in state), then
`terraform output -json` and parse it, or `terraform output -raw <name>` for a
single unquoted scalar you can drop straight into a shell variable:

```bash
ENDPOINT=$(terraform output -raw db_endpoint)
```

`-raw` avoids the surrounding quotes you'd get from the default or `-json`
single-value form, which matters when feeding the value into another command.
Never scrape it out of `terraform apply` log text — read it from `output` after
apply.

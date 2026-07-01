# q201 — Solution & walkthrough

## Restating the problem precisely

You edited `outputs.tf` to add one or more `output` blocks. Each new output
references resources that **already exist** in your applied stack — you changed
no resource, variable, or module. You want to see the new output values.

The trap is the assumption that "an output is just a value I defined, so
`terraform output` should show it." It won't — not yet. The single most
important fact:

> **Terraform outputs are computed from state + config and are *persisted into
> the state file*. `terraform output` only reads that persisted copy. A
> brand-new output block is not in state until an `apply` (or refresh) writes
> it.**

So editing the config file changes what *would* be output, but not what *is*
currently stored. That's the whole question.

## What each command actually does

### `terraform output` — reads state, nothing more

`terraform output` is a pure read of the outputs section of the current state
file. It never evaluates config, never touches infrastructure, never writes.
Immediately after adding a new `output` block, `terraform output <name>` errors
with `Output "<name>" not found`, and Terraform helpfully tells you why:

```
The output variable requested could not be found in the state file. If you
recently added this to your configuration, be sure to run `terraform apply`,
since the state won't be updated with new output values until then.
```

That message *is* the answer to the interview question.

### `terraform plan` — previews, persists nothing

`plan` evaluates config against state and computes the diff, **including
outputs**. A newly-added output shows up under a dedicated block:

```
Changes to Outputs:
  + config_path = "/tmp/.../server-config.txt"
```

But plan is read-only with respect to state — it writes nothing. Run
`terraform output` after a plan and the new value is still absent. Plan lets you
*see* the value without committing it, which is useful for a sanity check, but
it's not how you make the output usable by `terraform output` or downstream
consumers.

### `terraform apply` — evaluates, persists, prints (the clean answer)

This is the normal, correct path. Because your new output only references
existing resources, the plan portion is:

```
Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
```

No infrastructure moves. But apply still evaluates the new output, **writes it
into state**, and prints the full `Outputs:` block at the end. After this,
`terraform output`, `terraform output <name>`, and `terraform output -json` all
return the value. Since there are zero resource changes, this apply is safe — it
is not going to recreate or modify anything.

### `terraform apply -refresh-only` — safe alternative

`-refresh-only` reconciles state with the real world and re-evaluates outputs
**without** planning any resource create/update/delete. It updates state
(outputs included) to match reality, so it's a very safe way to get new outputs
persisted when you want a guarantee that nothing to your resources will change.
Use it when you're nervous a normal apply might pick up unrelated drift.

### `terraform refresh` — deprecated alias

`terraform refresh` did exactly this historically, but since **Terraform
0.15.4** it is **deprecated** and is now just an alias for `terraform apply
-refresh-only`. Mentioning that you know it's deprecated (and prefer the
explicit form) is a small credibility signal in an interview.

## Why the lab models it faithfully

- `random_pet` + `local_file` = "already-existing cloud resources" with real
  attributes worth exposing.
- Adding `output "config_path"` referencing `local_file.config.filename` = the
  "new output over existing infra" edit.
- `terraform output config_path` erroring = proof outputs come from state.
- `terraform plan` showing `Changes to Outputs:` with `No changes` to resources
  = proof plan previews but doesn't persist.
- `terraform apply` reporting `0 added, 0 changed, 0 destroyed` yet printing the
  new output = proof apply persists without touching infra.
- `terraform apply -refresh-only` writing a third output = the safe alternative.

## Common ways people get this wrong

- **"Just run `terraform output`."** Wrong for a *brand-new* output — it reads
  stale state and errors. It's the right command *after* an apply.
- **"`terraform plan` will show it, so I'm done."** Plan shows it but persists
  nothing; `terraform output` still fails afterward.
- **"I need to `terraform apply` and it might change my infra."** Adding an
  output referencing existing resources is `0 to add/change/destroy`. If you're
  still uneasy, `apply -refresh-only` guarantees no resource changes.
- **"Use `terraform refresh`."** Works, but it's deprecated — say
  `apply -refresh-only`.

## The 30-second version to say out loud

> "Outputs are computed from state and config, and they're *persisted in the
> state file* — `terraform output` only reads that stored copy. So a brand-new
> output block isn't visible to `terraform output` until something writes it to
> state. `terraform plan` will *show* it under `Changes to Outputs:` but
> persists nothing. The clean answer is `terraform apply`: since I only added
> an output over existing resources, it's `0 to add, 0 to change, 0 to destroy`,
> but apply still evaluates the output, writes it to state, and prints it — then
> `terraform output` works. If I want a guarantee nothing to my resources
> changes, I use `terraform apply -refresh-only`. `terraform refresh` does the
> same but it's deprecated, now just an alias for `apply -refresh-only`."

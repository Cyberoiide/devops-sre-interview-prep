# q201 — You added new outputs but changed no resources: what surfaces them?

## The scenario (as an interviewer would pose it)

> "Say you've got a working stack — it's applied, everything's healthy. A
> teammate asks for the database endpoint and the instance ID, so you open
> `outputs.tf` and add a couple of new `output` blocks. Crucially, they just
> reference resources that **already exist** — you didn't touch a single
> resource, variable, or module.
>
> Now you want to actually *see* those new output values. What command do you
> run? And — this is the part people get wrong — if you run `terraform output`
> right after saving the file, does the new output show up? Walk me through
> what's actually happening under the hood."

## What the interviewer is really probing

1. Do you understand that **outputs are computed from state + config**, not
   from the live infrastructure directly, and that they are **persisted into
   the state file**?
2. Do you know the sharp edge: `terraform output` reads outputs **from state**,
   so a brand-new output block is **not** visible via `terraform output` until
   an `apply` (or refresh) has written it into state?
3. Can you distinguish what each command does — `plan` **previews** output
   changes, `apply` **persists** them (and prints them), `output` **reads** the
   persisted copy?
4. Do you know that adding an output referencing existing resources requires
   **no infrastructure change** — `apply` reports `0 to add, 0 to change, 0 to
   destroy` yet still updates the outputs?
5. Are you current on the tooling: that `terraform refresh` is **deprecated**
   and is now an alias for `terraform apply -refresh-only`?

## The short answer

Outputs live in the **state file**, and `terraform output` only ever reads what
is already there. A brand-new `output` block isn't in state yet, so:

- **`terraform apply`** is the clean answer. Because you only added outputs
  referencing existing resources, the plan is `0 to add, 0 to change, 0 to
  destroy` — apply changes no infrastructure, but it **evaluates the new
  outputs, writes them to state, and prints them** under `Outputs:`. Then
  `terraform output <name>` works.
- **`terraform plan`** will *show* the new output under `Changes to Outputs:`,
  but it **does not persist** anything — run it afterward and `terraform
  output` still won't have the value.
- **`terraform apply -refresh-only`** is the alternative: it reconciles state
  with reality and re-writes outputs **without** planning any resource change.
  (The old `terraform refresh` does the same thing but is deprecated — it's now
  just an alias for `apply -refresh-only`.)

So: "just added an output, changed nothing" → run `terraform apply` (safe, zero
resource changes), then `terraform output`. The hands-on lab proves each step,
including the gotcha that `terraform output` shows *nothing new* before apply.

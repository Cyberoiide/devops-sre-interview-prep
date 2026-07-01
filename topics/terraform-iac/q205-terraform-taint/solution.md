# q205 — Solution & walkthrough

## Restating the problem precisely

You have a resource that exists and whose configuration is unchanged, but you
want Terraform to **destroy and recreate** it anyway — to recover from a bad
runtime state, to rotate something, or to prove the infra rebuilds cleanly. The
question is which command does that, what it actually mutates, and which one you
should use today.

The single most important sentence you can say in the interview is:

> **`taint` doesn't change my code and doesn't change the cloud — it only flips
> a flag in state that changes what the *next* apply decides to do.**

## The three-way model (this is what the question is really testing)

Terraform is always reconciling three separate things:

1. **HCL — desired state.** What your `.tf` files say should exist.
2. **tfstate — recorded state.** Terraform's memory of what it last created,
   including each object's attributes and metadata.
3. **Real infrastructure — actual state.** What's genuinely running in the
   cloud right now.

Every `plan` is a diff across these. Normally you drive recreation by changing
**#1** (edit HCL so an immutable attribute changes → `forces replacement`).

`taint` is different: it changes **neither #1 nor #3.** It writes a single piece
of metadata into **#2** — it marks the object instance as *tainted*. That flag
means "consider this object degraded." So on the next plan, even though HCL and
the real resource are byte-for-byte what they were, Terraform's *recorded* view
says "bad object" and it schedules a destroy-and-recreate (`-/+`). Understanding
that taint lives entirely in the middle box is the whole point of the question.

## What `-replace` does differently (and why it's better)

`terraform apply -replace="<address>"` reaches the identical end state — the
resource is destroyed and recreated — but it does **not** pre-mutate state.
Instead, `-replace` is a **plan-time instruction**: "for this run, treat this
address as needing replacement." That difference matters for three reasons:

1. **You see it before you commit.** The plan output shows the `-/+ forces
   replacement` for exactly the resources you named. With `taint`, the
   state-mutating step happens first and *silently*; the destroy/create only
   surfaces on a later plan that someone has to remember to inspect.
2. **No lingering state mutation.** If you `taint` and then walk away, state is
   left dirty — the next person's plan will surprise them with a rebuild they
   didn't request. `-replace` leaves nothing behind if you don't apply.
3. **Safe in automation.** A CI pipeline can run
   `terraform plan -replace=<addr> -out=tfplan`, have a human approve the shown
   diff, then `terraform apply tfplan` — one artifact, one review, no
   out-of-band state write. That's why HashiCorp deprecated `taint` in favor of
   it.

`terraform taint` and `terraform untaint` have been **deprecated since Terraform
0.15.2**; they still work and still print a warning pointing you at `-replace`,
but you shouldn't reach for them in new work.

## Walking through the lab

- `random_id.server` is the perfect witness: it computes its value **once** at
  create time and holds it stable across applies. So if its `hex` changes, the
  only explanation is that the object was destroyed and recreated.
- `local_file.manifest` interpolates that hex into a file, giving a second,
  on-disk proof that recreation cascaded downstream.
- **Step 1** shows `plan -replace` previewing the `-/+`, then `apply -replace`
  changing the id — recreation with an unchanged config.
- **Step 2** shows `taint` flipping the state flag (with its deprecation
  warning), a subsequent `plan` revealing the forced recreate, and crucially
  `untaint` *clearing* the flag so a mistaken taint costs nothing.
- **Step 3** proves you can surgically replace `random_id.fleet[1]` while `[0]`
  and `[2]` are untouched — the indexed-address trick for `count`/`for_each`.

## When you actually want this in real life

- A VM/container drifted into a wedged runtime state that config can't express;
  fastest fix is a clean rebuild.
- **Reproducibility check** — deliberately replace a resource to confirm your
  code stands it back up cleanly (a mini DR drill).
- **Rotation** — force recreation of something whose "freshness" isn't captured
  by an attribute (e.g. a resource backing a credential you want reissued).

Prefer expressing recreation in **config** when you can (a changed input, a
`keepers` map on `random_*`, or a `replace_triggered_by` in a `lifecycle`
block), because that's declarative and reviewable. `-replace` is the imperative
escape hatch for the one-off "just rebuild this now."

## The 30-second version to say out loud

> "`taint` doesn't touch my HCL and doesn't touch the cloud — it just flips a
> flag in **state** so the *next* apply destroys and recreates that resource,
> even though nothing changed. It's been deprecated since 0.15.2. Today I use
> `terraform apply -replace=<address>` — same result, but it's a plan-time flag,
> so I see the `-/+ forces replacement` in the plan and approve it in one step
> with no separate state-mutating command. That's what makes it safe in CI. And
> I can target a single instance of a count/for_each resource with an indexed
> address like `-replace='aws_instance.web[2]'`."

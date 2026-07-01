# q205 — Forcing resource recreation: `terraform taint` vs `-replace`

## The scenario (as an interviewer would pose it)

> "One of your production VMs has drifted into a weird, half-broken state — it
> still exists, its config in Terraform hasn't changed, but something on the box
> is wedged and you just want Terraform to blow it away and stand up a fresh
> one. A teammate says 'just run `terraform taint` on it.' Talk me through what
> `taint` actually does — what does it change, in your code, in your state, or
> in the cloud? Is that still the command you'd reach for in 2024? And how would
> you do the same thing more safely, especially from a pipeline?"

## What the interviewer is really probing

1. Do you understand that `taint` changes **nothing in your HCL and nothing yet
   in the cloud** — it only flips a flag in **state** that makes the *next*
   apply destroy and recreate the resource?
2. Can you articulate the **three-way model**: HCL (desired), tfstate
   (recorded), real infrastructure (actual)? `taint` operates purely on the
   middle one.
3. Do you know `terraform taint` / `untaint` are **deprecated since 0.15.2**,
   and that the modern equivalent is `terraform apply -replace="<address>"`?
4. Can you explain *why* `-replace` is better — it's a **plan-time** flag, so
   you see the destroy/create in the plan **before** approving, with no
   separate state-mutating command; much safer in automation?
5. Do you know how to target **one instance** of a `count`/`for_each` resource
   via its indexed address?

## The short answer

`terraform taint <addr>` marks a resource as tainted **in state**. It does not
touch your configuration and does not touch the real resource — it just records
"this object is bad." The very next `plan`/`apply` then shows that resource as
`-/+` (destroy then create) and rebuilds it, even though nothing in the config
changed. `terraform untaint <addr>` clears the flag.

But `taint`/`untaint` have been **deprecated since Terraform 0.15.2**. The
recommended replacement is:

```bash
terraform plan  -replace="aws_instance.web"   # preview the destroy/create
terraform apply -replace="aws_instance.web"   # do it
```

`-replace` is strictly better because it's a **plan-time flag**: you *see* the
`-/+ forces replacement` in the plan output and approve it, in one step, with no
separate command that silently mutates state. That's what makes it safe to use
in CI. For `count`/`for_each` resources you target a single instance by its
indexed address, e.g. `-replace='aws_instance.web[2]'` or
`-replace='aws_instance.web["blue"]'`.

The hands-on lab proves all of this locally (no cloud), using `random_id` as a
stand-in resource whose value visibly changes on every recreation.

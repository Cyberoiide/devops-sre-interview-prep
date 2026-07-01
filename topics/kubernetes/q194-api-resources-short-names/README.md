# q194 — Listing resource types and their short names

## The interview question

> "Without using `kubectl explain`, how do you list **all** the resource types
> a cluster serves along with their short names — and how would you tell whether
> a given resource is namespaced?"

This looks like a trivia question but it's really probing whether you
understand where `kubectl` gets its knowledge of resource types. A weak
candidate recites a few short names from memory (`po`, `svc`, `deploy`). A
strong candidate names the command — **`kubectl api-resources`** — and explains
that it's answered by the **API server's discovery endpoints**, so it reflects
*exactly what this cluster serves*, CRDs and all — not a static list baked into
the client.

## What `kubectl api-resources` actually does

`kubectl api-resources` lists every resource **kind** the API server currently
serves. It gets this by hitting the discovery endpoints — `/api` for the core
group and `/apis` for every named group/version (e.g. `/apis/apps/v1`) — and
merging the results. That's the important part: the output is **dynamic**. If
you install a CRD, its resource shows up immediately; if an API version is
disabled or an aggregated API is registered, the list changes to match. This is
why it's the *authoritative* way to discover short names and namespacing for
the cluster in front of you, rather than trusting documentation that may target
a different version.

The default output has these columns:

| Column | Meaning |
|---|---|
| `NAME` | Plural resource name used in URLs/commands (e.g. `pods`) |
| `SHORTNAMES` | Aliases you can type instead (e.g. `po`, `svc`, `deploy`) |
| `APIVERSION` | The group/version serving it (e.g. `apps/v1`; core group shows just `v1`) |
| `NAMESPACED` | `true` if namespaced, `false` if cluster-scoped |
| `KIND` | The Kind used in YAML `kind:` (e.g. `Pod`) |

> On older `kubectl` the group column was labelled `APIGROUP` and showed only
> the group (`apps`); recent `kubectl` shows `APIVERSION` with the full
> `group/version` (`apps/v1`). Same information, slightly different framing.

## The three sibling commands (don't confuse them)

- **`kubectl api-resources`** — lists the resource **types** the cluster serves
  (this question). "What kinds exist, and what are they called?"
- **`kubectl explain <resource>`** — documents the **schema/fields** of one
  resource. "What fields does a Pod have?"
- **`kubectl get <resource>`** — lists the **instances** of a resource. "What
  pods are actually running?"

And its closest relative:

- **`kubectl api-versions`** — lists the group/versions the cluster serves
  (e.g. `apps/v1`, `networking.k8s.io/v1`), without the per-resource detail.

## What you should be able to answer

- The command is **`kubectl api-resources`**, and it's answered by the API
  server's **discovery** endpoints — so it reflects this cluster's CRDs,
  enabled API versions, and aggregated APIs, not a static client-side list.
- The columns and what each means, especially **`NAMESPACED`** and
  **`SHORTNAMES`**.
- The useful **flags**: `--namespaced=true|false`, `--api-group=<group>`
  (`--api-group=''` for core), `-o wide` (adds `VERBS` and `CATEGORIES`),
  `-o name`, `--verbs=list,get`, `--sort-by=name|kind`, `--categories`.
- The **`all` category** and why `kubectl get all` doesn't truly get everything.
- How this differs from `kubectl explain` and `kubectl get`.

## Files in this topic

- `exercise.md` — a runnable `kind` lab that reads the columns, filters by
  namespacing and group, shows short names, and **proves the dynamic claim** by
  installing a CRD and watching its short name appear.
- `solution.md` — the full explanation of the command, the discovery mechanism,
  every useful flag, the `all` category, and a one-liner summary.
- `deeper.md` — follow-up interview questions with concise answers.

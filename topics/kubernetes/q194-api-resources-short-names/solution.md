# Solution — Listing resource types and their short names

## The core answer

The command is **`kubectl api-resources`**. It lists every resource **kind** the
API server currently serves, with a row per resource and columns for its plural
name, short names, group/version, whether it's namespaced, and its Kind. That's
the direct answer to "list all resource types and their short names," and
`--namespaced=true|false` answers "is X namespaced?".

The insight that separates a strong answer: this list is **not baked into
`kubectl`**. It comes from the API server's **discovery** endpoints, so it
mirrors exactly what *this* cluster serves.

## Where the data comes from (discovery)

`kubectl api-resources` queries:

- `/api` — the legacy **core** group (`v1`: pods, services, configmaps, ...).
- `/apis` — the list of all named groups, then each group/version's discovery
  doc, e.g. `/apis/apps/v1`, `/apis/networking.k8s.io/v1`.

It merges those into the table you see. Because the source is the live API
server:

- **Installed CRDs appear immediately** — register a `CustomResourceDefinition`
  and its resource (and short names) show up on the next call.
- **Enabled/disabled API versions are reflected** — if a version isn't served,
  it isn't listed.
- **Aggregated APIs** (extension API servers registered via `APIService`, e.g.
  `metrics.k8s.io`) show up too.

This is why `api-resources` is the *authoritative* source for short names and
namespacing on the cluster in front of you — documentation can lag or target a
different Kubernetes version; discovery cannot.

## The columns

```
NAME                    SHORTNAMES   APIVERSION   NAMESPACED   KIND
pods                    po           v1           true         Pod
deployments             deploy       apps/v1      true         Deployment
nodes                   no           v1           false        Node
```

- **`NAME`** — the plural resource name used in URLs and commands (`pods`).
- **`SHORTNAMES`** — aliases you can type (`po`). Some resources have none
  (e.g. `secrets`), so the column is blank for them.
- **`APIVERSION`** — the group/version serving the resource. Core resources show
  a bare `v1`; grouped resources show `group/version` like `apps/v1`. (Older
  `kubectl` labelled this `APIGROUP` and showed only the group.)
- **`NAMESPACED`** — `true` (lives in a namespace) or `false` (cluster-scoped).
- **`KIND`** — the value you put in a manifest's `kind:` field (`Pod`).

## The useful flags

| Flag | What it does |
|---|---|
| `--namespaced=true` / `--namespaced=false` | Show only namespaced / cluster-scoped resources |
| `--api-group=apps` | Limit to one group; `--api-group=''` for the **core** group |
| `-o wide` | Add `VERBS` and (recent `kubectl`) `CATEGORIES` columns |
| `-o name` | Machine-readable `resource.group` list, one per line |
| `--verbs=list,get` | Only resources supporting the given verbs |
| `--sort-by=name` / `--sort-by=kind` | Sort the output |
| `--categories=all` | Only resources in the named category |

**`VERBS`** (via `-o wide`) lists the operations a resource supports —
`get list watch create update patch delete deletecollection`. A resource
exposing only `get`/`list` is effectively read-only (many subresources and
`componentstatuses` are), which tells you an attempt to `create`/`delete` it
will fail.

## The `all` category (and why `kubectl get all` lies)

The **`CATEGORIES`** column groups resources. The best-known category is
**`all`**. `kubectl get all` doesn't return *every* resource in the namespace —
it returns only the resources tagged with the `all` category (roughly pods,
services, deployments, replicasets, statefulsets, daemonsets, jobs, cronjobs,
etc.). Crucially it **omits** things like `secrets`, `configmaps`, `ingresses`,
`serviceaccounts`, and PVCs. So `kubectl get all` is a curated "workload-ish"
view, not a complete inventory — a common gotcha. You can see which resources
belong to `all` with:

```bash
kubectl api-resources --categories=all
```

## Don't confuse it with its siblings

- **`kubectl api-resources`** → the resource **types** (this question).
- **`kubectl explain <resource>`** → the **schema/fields** of one resource
  (`kubectl explain pod.spec.containers`). Documents structure, not instances or
  the type list.
- **`kubectl get <resource>`** → the **instances** of a resource in the cluster.
- **`kubectl api-versions`** → the group/**versions** served (e.g. `apps/v1`,
  `batch/v1`), without per-resource breakdown. The natural companion to
  `api-resources`.

## Quick recipes

```bash
kubectl api-resources                              # everything
kubectl api-resources --namespaced=false           # cluster-scoped only
kubectl api-resources --api-group=apps             # one group
kubectl api-resources --api-group=''               # core group only
kubectl api-resources -o wide                       # + VERBS + CATEGORIES
kubectl api-resources --verbs=list,watch           # watchable/listable resources
kubectl api-resources | grep -i ingress            # find a short name fast
kubectl api-versions                                # the sibling: group/versions
```

## One-liner summary for the interviewer

> "`kubectl api-resources` — it lists every kind the cluster serves with its
> short names, API version, whether it's namespaced, and its Kind. It's answered
> by the API server's discovery endpoints, so it reflects *this* cluster
> exactly, including installed CRDs and aggregated APIs — it's not a static list
> in the client. I'd filter with `--namespaced` or `--api-group`, and use
> `-o wide` to see each resource's supported VERBS and categories. That's
> distinct from `kubectl explain`, which documents a resource's fields, and
> `kubectl get`, which lists instances."

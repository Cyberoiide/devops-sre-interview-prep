# Deeper — follow-up interview questions

### 1. Where does `kubectl api-resources` get its data — is it hardcoded in `kubectl`?

No. It queries the API server's **discovery** endpoints: `/api` for the core
group and `/apis` (then each group/version doc like `/apis/apps/v1`) for named
groups. `kubectl` merges those into the table. Because the source is the live
server, the output is **dynamic** — it reflects the exact set of resources this
cluster serves, including installed CRDs, enabled/disabled API versions, and
aggregated (extension) APIs. That's why it's the authoritative source for short
names and namespacing, rather than docs that might target another version.

### 2. Why does installing a CRD immediately change the output?

A `CustomResourceDefinition` tells the API server to start serving a new
resource under its declared group/version, with the plural name, kind, and any
`shortNames` from the CRD spec. The server registers it into discovery, so the
next `kubectl api-resources` call sees it. There's nothing to restart or
reconfigure on the client — the client just re-reads discovery. (`kubectl` does
cache discovery under `~/.kube/cache/discovery`, but it invalidates quickly; a
`--cache-dir` or a fresh call reflects the change almost immediately.)

### 3. What's the difference between `api-resources` and `explain`?

`api-resources` lists the **types** the cluster serves ("what kinds exist and
what are they called?"). `kubectl explain <resource>` documents the
**schema/fields** of one type ("what fields does a Pod's spec have?"), and
`explain` itself is powered by the OpenAPI schema published by the API server —
including CRD schemas defined via `openAPIV3Schema`. So they're complementary:
one enumerates the catalogue, the other documents an entry in it. Neither lists
instances — that's `kubectl get`.

### 4. What does the `NAMESPACED` column actually control?

Whether objects of that kind live inside a namespace. Namespaced resources
(`pods`, `deployments`, `configmaps`) are addressed as
`/namespaces/<ns>/<resource>` and are shown/filtered with `-n`/`--all-namespaces`.
Cluster-scoped resources (`nodes`, `namespaces`, `persistentvolumes`,
`clusterroles`, CRDs themselves) exist once per cluster and ignore `-n`. Getting
this wrong is a common cause of "why can't I find my object in this namespace?"
— because it isn't in one. `kubectl api-resources --namespaced=false` is the
quick check.

### 5. What is the `all` category and why doesn't `kubectl get all` show everything?

`all` is a **category** — a curated grouping surfaced in the `CATEGORIES` column
(`-o wide`). `kubectl get all` returns only resources tagged `all` (pods,
services, deployments, replicasets, statefulsets, daemonsets, jobs, cronjobs,
and a few more), **not** secrets, configmaps, ingresses, serviceaccounts, PVCs,
or most CRDs. So it's a workload-oriented summary, not a full namespace
inventory. `kubectl api-resources --categories=all` shows exactly what's in it.
CRDs can opt into categories via `spec.names.categories`.

### 6. What does the `VERBS` column tell you, and when is it useful?

It lists the operations a resource supports:
`get list watch create update patch delete deletecollection`. It's useful for
spotting **read-only** resources — e.g. `componentstatuses` or many
subresources expose only `get`/`list`, so trying to `create` or `delete` them
fails. It also hints at what RBAC verbs are even meaningful for that resource.
View it with `kubectl api-resources -o wide`, or invert the query with
`--verbs=create,delete` to list only mutable resources.

### 7. `api-resources` vs `api-versions` — what's the difference?

`kubectl api-versions` lists the **group/versions** the cluster serves
(`apps/v1`, `batch/v1`, `networking.k8s.io/v1`, `v1`, ...) — one line each, no
resource detail. `kubectl api-resources` drills into the **resources** within
those versions (names, short names, kinds, namespacing). Use `api-versions` to
answer "does this cluster serve `autoscaling/v2`?" and `api-resources` to answer
"what's the short name for `horizontalpodautoscalers`, and is it namespaced?".

### 8. A resource appears under two API versions (e.g. an HPA) — how does that show up?

`api-resources` lists a row per served group/version, so a kind available at
both `autoscaling/v1` and `autoscaling/v2` can appear more than once with
different `APIVERSION` values. `kubectl` uses the **preferred** version for the
group when you run `kubectl get hpa` without specifying one. During API
migrations (e.g. an `Ingress` moving from `extensions/v1beta1` to
`networking.k8s.io/v1`), watching `api-resources` and `api-versions` is how you
confirm which versions the cluster still serves before you cut over manifests.

### 9. How do short names get resolved when you type `kubectl get po`?

`kubectl` maps the alias to a resource via the same discovery data — the
`shortNames` field each resource advertises (core ones like `po`→`pods` are
served by the API server's discovery, and CRDs declare theirs in
`spec.names.shortNames`). So short names are also **cluster-defined**, not
hardcoded in the client: a CRD author picks `ct` for `crontabs`, and `kubectl
get ct` works because discovery told the client about the alias. If two
resources ever claimed the same short name, you'd disambiguate with the fully
qualified `resource.group` form (e.g. `crontabs.stable.example.com`).

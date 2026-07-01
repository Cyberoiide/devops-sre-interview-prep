# Exercise — Discover resource types and short names on `kind`

This lab uses `kubectl api-resources` to read the resource catalogue of a real
cluster, filter it, and — the important part — **prove that the list is dynamic**
by installing a CRD and watching its short name appear in the output. Then we
use that short name in a real command.

## Prerequisites

- `docker` running
- [`kind`](https://kind.sigs.k8s.io/) installed
- `kubectl` installed

## 0. Create a cluster (or use an existing one)

A single-node cluster is plenty — we're inspecting the API surface, not
scheduling.

```bash
kind create cluster --name q194

# Confirm it's up.
kubectl get nodes
```

You should see one node, `q194-control-plane`, in `Ready` state.

---

## 1. List everything and read the columns

```bash
kubectl api-resources | head -n 20
```

Representative output (yours will have more rows):

```
NAME                    SHORTNAMES   APIVERSION   NAMESPACED   KIND
bindings                             v1           true         Binding
componentstatuses       cs           v1           false        ComponentStatus
configmaps              cm           v1           true         ConfigMap
endpoints               ep           v1           true         Endpoints
events                  ev           v1           true         Event
namespaces              ns           v1           false        Namespace
nodes                   no           v1           false        Node
persistentvolumeclaims  pvc          v1           true         PersistentVolumeClaim
persistentvolumes       pv           v1           false        PersistentVolume
pods                    po           v1           true         Pod
secrets                              v1           true         Secret
serviceaccounts         sa           v1           true         ServiceAccount
services                svc          v1           true         Service
...
```

Read the five columns: `NAME` (plural, what you type), `SHORTNAMES` (aliases),
`APIVERSION` (the serving group/version — core resources show bare `v1`),
`NAMESPACED` (true/false), and `KIND` (the YAML `kind:`). Note that `namespaces`
and `nodes` are `false` (cluster-scoped) while `pods` is `true`.

Look at a few group resources too:

```bash
kubectl api-resources --api-group=apps
```

```
NAME          SHORTNAMES   APIVERSION   NAMESPACED   KIND
daemonsets    ds           apps/v1      true         DaemonSet
deployments   deploy       apps/v1      true         Deployment
replicasets   rs           apps/v1      true         ReplicaSet
statefulsets  sts          apps/v1      true         StatefulSet
```

---

## 2. Filter: namespaced vs cluster-scoped

```bash
# Only cluster-scoped resources (nodes, namespaces, PVs, CRDs, ...).
kubectl api-resources --namespaced=false
```

```
NAME                     SHORTNAMES   APIVERSION                       NAMESPACED   KIND
componentstatuses        cs           v1                               false        ComponentStatus
namespaces               ns           v1                               false        Namespace
nodes                    no           v1                               false        Node
persistentvolumes        pv           v1                               false        PersistentVolume
customresourcedefinitions crd,crds    apiextensions.k8s.io/v1          false        CustomResourceDefinition
clusterroles                          rbac.authorization.k8s.io/v1     false        ClusterRole
storageclasses           sc           storage.k8s.io/v1                false        StorageClass
...
```

```bash
# Only namespaced resources.
kubectl api-resources --namespaced=true | head -n 8
```

This is the reliable way to answer "is X namespaced?" for *this* cluster.

---

## 3. Filter by API group

```bash
# One named group.
kubectl api-resources --api-group=networking.k8s.io
```

```
NAME              SHORTNAMES   APIVERSION            NAMESPACED   KIND
ingressclasses                 networking.k8s.io/v1  false        IngressClass
ingresses         ing          networking.k8s.io/v1  true         Ingress
networkpolicies   netpol       networking.k8s.io/v1  true         NetworkPolicy
```

```bash
# The CORE group is the empty string.
kubectl api-resources --api-group=''
```

That returns exactly the `v1` core resources (pods, services, configmaps, ...).

---

## 4. Short names, VERBS, and CATEGORIES with `-o wide`

```bash
kubectl api-resources -o wide | head -n 12
```

```
NAME          SHORTNAMES   APIVERSION   NAMESPACED   KIND         VERBS                                                        CATEGORIES
configmaps    cm           v1           true         ConfigMap    [create delete deletecollection get list patch update watch]
pods          po           v1           true         Pod          [create delete deletecollection get list patch update watch] all
services      svc          v1           true         Service      [create delete deletecollection get list patch update watch] all
deployments   deploy       apps/v1      true         Deployment   [create delete deletecollection get list patch update watch] all
...
```

`-o wide` adds two columns:

- **`VERBS`** — the operations the resource supports. A read-only resource (e.g.
  `componentstatuses`, or a subresource) shows only `get`/`list`, which is how
  you know you can't `create` or `delete` it.
- **`CATEGORIES`** — groupings like `all`. Note `pods`, `services`,
  `deployments` are in `all` but `configmaps` and `secrets` are **not** — that's
  why `kubectl get all` doesn't actually show everything.

Find resources supporting a specific verb:

```bash
# Which resources can you watch and list?
kubectl api-resources --verbs=list,watch | head
```

Sort the catalogue:

```bash
kubectl api-resources --sort-by=name | head
kubectl api-resources --sort-by=kind | head
```

`-o name` gives a clean machine-readable list (`resource.group` form):

```bash
kubectl api-resources -o name | head
# bindings
# componentstatuses
# configmaps
# ...
# deployments.apps
# ...
```

---

## 5. Prove it's dynamic — install a CRD and watch its short name appear

First confirm the resource does **not** exist yet:

```bash
kubectl api-resources --api-group=stable.example.com
# (no rows — nothing served under that group)
```

Apply a minimal CRD that defines a `CronTab` kind with short name `ct`:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
      - ct
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
EOF
```

Now re-run `api-resources` — the new type is served immediately:

```bash
kubectl api-resources --api-group=stable.example.com
```

```
NAME       SHORTNAMES   APIVERSION              NAMESPACED   KIND
crontabs   ct           stable.example.com/v1   true         CronTab
```

There it is: `crontabs`, short name `ct`, namespaced, from a group that
returned nothing seconds ago. `kubectl api-versions` shows the new group too:

```bash
kubectl api-versions | grep stable
# stable.example.com/v1
```

**Use the short name in a real command.** Create an instance and get it by its
alias:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: stable.example.com/v1
kind: CronTab
metadata:
  name: my-cron
spec:
  cronSpec: "* * * * */5"
  image: my-cron-image:latest
EOF

kubectl get ct                    # 'ct' works exactly like 'po' or 'svc'
```

```
NAME      AGE
my-cron   5s
```

And, for contrast, use a built-in short name in a real get:

```bash
kubectl get po -A                 # 'po' == pods, across all namespaces
```

---

## 6. See the sibling command

```bash
kubectl api-versions | head
```

```
admissionregistration.k8s.io/v1
apiextensions.k8s.io/v1
apps/v1
batch/v1
networking.k8s.io/v1
rbac.authorization.k8s.io/v1
storage.k8s.io/v1
stable.example.com/v1
v1
...
```

`api-versions` lists group/versions; `api-resources` drills into the resources
within them. Together they describe the entire API surface of the cluster.

---

## Teardown

```bash
kubectl delete crontab my-cron
kubectl delete crd crontabs.stable.example.com
kind delete cluster --name q194
```

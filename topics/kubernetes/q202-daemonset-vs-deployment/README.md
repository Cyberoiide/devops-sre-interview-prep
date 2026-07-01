# q202 — When to use a DaemonSet instead of a Deployment

## The interview question

> "You need to run a log-collection agent on your cluster. Would you use a
> Deployment or a DaemonSet — and why? More generally, how do you decide
> between the two, and where does a StatefulSet fit in?"

This question checks whether a candidate understands *what each controller
guarantees* rather than reaching for `Deployment` reflexively. The tell of a
strong answer is the phrase **"once per node"**: a DaemonSet's job is to run
exactly one copy of a pod on every matching node and to track the node set as
it changes, while a Deployment runs a count of interchangeable replicas placed
wherever they fit.

## What a DaemonSet actually guarantees

A DaemonSet ensures **one copy of a pod runs on every node** (or on every node
matching a `nodeSelector`/affinity). You do **not** set a replica count — the
effective count *equals the number of matching nodes*. The DaemonSet controller
watches the node set: when a **new node joins** the cluster it automatically
adds the pod there, and when a node is removed the pod goes with it. Scheduling
is driven by the controller working with the default scheduler (modern
Kubernetes injects node affinity onto each DaemonSet pod so the normal
scheduler binds it), and DaemonSet pods usually carry **broad tolerations** so
they run even on tainted or control-plane nodes.

The defining trait of DaemonSet workloads is that they **interact with the node
itself** — reading host logs through a `hostPath`, exposing node-level metrics,
or programming the node's networking/iptables/eBPF.

## What a Deployment actually guarantees

A Deployment manages a ReplicaSet of **N identical, stateless replicas** that
you scale by count (`replicas: N`) or automatically via an HPA. The scheduler
places those replicas **wherever resources fit** — it might land two on one
node and none on another. There is **no per-node guarantee**. This is exactly
what you want for a stateless application, API, or web server that you scale by
demand, not by node count.

## What you should be able to answer

- The core distinction: **DaemonSet = one pod per node (count = node count),
  Deployment = N replicas scheduled by fit**.
- The **use-case families** for a DaemonSet: log collectors (Fluent Bit,
  Fluentd, Vector), monitoring agents (Prometheus node-exporter, Datadog
  agent), CNI plugins (Calico, Cilium, Flannel), `kube-proxy`, storage/CSI node
  plugins, and security agents (Falco) — anything **node-level infrastructure**.
- Why an app/API uses a **Deployment**, and where a **StatefulSet** fits
  (stable identity + ordered rollout + per-pod storage).
- How each controller **updates**: DaemonSet `RollingUpdate`
  (`maxUnavailable`) / `OnDelete`; Deployment `RollingUpdate`
  (`maxSurge`/`maxUnavailable`) via ReplicaSets.
- How DaemonSets handle **taints** (need tolerations to run on tainted or
  control-plane nodes) and how to target a **subset** of nodes with
  `nodeSelector`/`nodeAffinity` (e.g. only GPU nodes).

## Files in this topic

- `exercise.md` — a runnable multi-node `kind` lab that proves one-DaemonSet-
  pod-per-node, shows a Deployment scattering replicas, watches a DaemonSet pod
  appear automatically when a node is labeled, and tolerates the control-plane
  taint.
- `solution.md` — the full DaemonSet-vs-Deployment decision, a comparison
  table, the use-case families, and a one-liner summary.
- `deeper.md` — follow-up interview questions with concise answers.

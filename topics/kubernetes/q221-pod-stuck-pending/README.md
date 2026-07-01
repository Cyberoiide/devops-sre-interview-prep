# q221 — Pod stuck in `Pending`

## The interview question

> "One of your pods has been sitting in `Pending` for several minutes and
> never starts. Walk me through how you diagnose *why* the scheduler cannot
> place it, and how you'd fix the common root causes."

This is one of the most common Kubernetes troubleshooting questions because
`Pending` is almost always a **scheduling** problem, and scheduling failures
have a small, well-defined set of causes. A strong candidate does not guess —
they read the `FailedScheduling` event and let the scheduler tell them the
reason.

## What `Pending` actually means

A pod is `Pending` when it has been accepted by the API server but is **not
yet running on a node**. In the overwhelming majority of cases this means the
**scheduler has not been able to bind the pod to a node**. The scheduler
records exactly why in a `FailedScheduling` event on the pod.

(Less commonly a pod can briefly show `Pending` right after being bound, before
the kubelet starts pulling images and creating containers — but once binding
happens and the kubelet takes over, the phase moves to `ContainerCreating` /
image-pull states, which are *not* the classic `Pending` covered here. See
`deeper.md`.)

## What you should be able to answer

- **First move:** `kubectl describe pod <name>` and read the **Events**
  section — the scheduler writes the exact reason there.
- The **three main root-cause families** and their tell-tale event strings:
  1. **No node matches placement constraints** — `nodeSelector`, node/pod
     affinity & anti-affinity, or **taints without a matching toleration**.
  2. **Insufficient resources** — the pod's CPU/memory/ephemeral-storage/GPU
     **requests** exceed what any node has allocatable.
  3. **Volume problems** — the pod references a PVC that is unbound/`Pending`
     (missing StorageClass, no PV, topology/zone mismatch).
- Why **requests (not limits)** drive scheduling.
- The difference between a node's **capacity** and its **allocatable**.
- How **Cluster Autoscaler / Karpenter** changes the outcome (adds a node
  instead of leaving you stuck).
- The systematic **triage flow** with the exact commands at each step.

## Files in this topic

- `exercise.md` — a runnable `kind` lab that reproduces **all three** causes,
  shows the diagnosis, and fixes each one.
- `solution.md` — the full triage flow, how to read `FailedScheduling`, the
  three root-cause families with fixes, and a decision tree.
- `deeper.md` — follow-up interview questions with concise answers.

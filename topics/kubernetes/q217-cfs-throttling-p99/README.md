# q217 — CFS throttling: P99 is bad but the CPU charts look fine

## The scenario (as an interviewer would pose it)

> "You run a multi-threaded service on Kubernetes. Users report intermittent
> latency spikes and your P99 latency SLO is being violated. But when you open
> your dashboards, CPU utilization for the pod looks perfectly healthy — well
> below the limit — even when you zoom the graph down to per-second granularity.
> The container is not OOMKilled, there are no GC pauses, the downstream
> dependencies are fast, and the node is not saturated. What is happening, how
> do you prove it, and how do you fix it?"

This is the classic **CFS bandwidth throttling** problem. The short version:
the container has a **CPU limit**, Kubernetes translates that limit into a Linux
CFS quota that is enforced in short (~100 ms) windows, and a multi-threaded app
can burn its entire quota early in the window using several cores in parallel.
The kernel then **de-schedules (throttles)** the container for the rest of that
window. The stalls are sub-100 ms, so they hide inside the averaging window of
any normal CPU chart — the graph shows a modest average while the app is
actually being repeatedly frozen.

## What you should be able to answer

- **What a CPU limit actually is** — `cpu.cfs_period_us` / `cpu.cfs_quota_us`
  (cgroup v1) or `cpu.max` (cgroup v2), and the quota math for `500m`, `1`, `2`.
- **Why the CPU chart lies** — per-period enforcement (~100 ms) vs. multi-second
  (or even 100 ms) averaging in the dashboard.
- **Why multi-threading makes it worse** — parallel threads drain the per-period
  quota in a fraction of the wall-clock window.
- **The signal you should actually trust** —
  `container_cpu_cfs_throttled_periods_total`,
  `container_cpu_cfs_throttled_seconds_total`,
  `container_cpu_cfs_periods_total`, and the raw `cpu.stat`
  (`nr_throttled`, `throttled_time`/`throttled_usec`).
- **The difference between a CPU *request* and a CPU *limit*** — requests set
  `cpu.shares`/`cpu.weight` (scheduling proportion, never throttles); only limits
  set the quota that causes throttling.
- **The full menu of fixes and their honest tradeoffs** — raise/remove the limit
  (QoS class change, noisy-neighbor risk), align thread count / `GOMAXPROCS`
  (automaxprocs) to the quota, the static CPU Manager policy, and the old-kernel
  CFS accounting bug.

## Files

- `exercise.md` — a runnable kind/minikube lab that reproduces throttling with a
  low CPU limit and a multi-threaded burner, proves it by reading `cpu.stat`,
  then fixes it by raising/removing the limit.
- `solution.md` — the full walkthrough: the averaging-vs-enforcement illusion,
  the quota math with worked numbers, the metrics to trust, and every fix with
  its tradeoffs.
- `deeper.md` — follow-up interview questions with concise correct answers.

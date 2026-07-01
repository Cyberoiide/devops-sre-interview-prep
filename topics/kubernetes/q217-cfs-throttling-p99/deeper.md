# Deeper — follow-up interview questions

Concise, correct answers to the follow-ups an interviewer is likely to drill
into after you nail the main question.

---

### 1. What exactly are `cpu.cfs_period_us` and `cpu.cfs_quota_us`?

They are the two cgroup (v1) files that program the Linux CFS **bandwidth
controller**. `cpu.cfs_period_us` is the length of each enforcement window in
microseconds (**default 100000 = 100 ms**). `cpu.cfs_quota_us` is how much
**CPU-time** the cgroup may run within one period, in microseconds (`-1` =
unlimited). A limit of `500m` sets quota `50000`; `1` sets `100000`; `2` sets
`200000` (200 ms of CPU-time per 100 ms window, only achievable across multiple
cores). When the quota is exhausted, every thread in the cgroup is throttled
until the next period boundary.

---

### 2. cgroup v1 vs. v2 — what changes for this problem?

The **mechanism and defaults are the same**; the interface differs.

- **v1:** two files, `cpu.cfs_period_us` and `cpu.cfs_quota_us`, under
  `/sys/fs/cgroup/cpu,cpuacct/`. Throttle counters in `cpu.stat` include
  `throttled_time` in **nanoseconds**. Request → `cpu.shares` (a `1024`-based
  weight).
- **v2:** one file, `cpu.max`, holding `"<quota> <period>"` (e.g. `50000 100000`,
  or `"max 100000"` for unlimited), under the unified hierarchy at
  `/sys/fs/cgroup/`. `cpu.stat` reports `throttled_usec` in **microseconds** and
  adds `nr_bursts`/`burst_usec` if CPU burst is enabled. Request → `cpu.weight`
  (a `1`–`10000` scale, not the raw 1024 shares).

For diagnosis: `nr_periods` / `nr_throttled` exist in both; just mind the
time-unit difference on the cumulative counter.

---

### 3. Does a CPU *request* cause throttling?

**No.** A request sets `cpu.shares` (v1) / `cpu.weight` (v2), which is a
**relative scheduling weight** that only matters when the CPU is **contended** —
it decides proportional shares, never an absolute cap. A container may freely
exceed its request when spare CPU exists. **Only a CPU *limit*** sets the CFS
**quota**, and only the quota throttles. So: throttling present ⇒ a limit is set.

---

### 4. Then why keep the request but drop the limit?

Because they do different jobs. The **request** is what the **scheduler** uses to
place the pod and reserve capacity (bin-packing, `kube-scheduler` fit checks),
and it feeds the QoS classification and node CPU allocation. Keeping it means the
pod still gets a fair, reserved share under contention. The **limit** is the only
thing that throttles, so dropping it removes the sub-100 ms freezes while the
request preserves sane scheduling behaviour. Dropping the request instead would
hurt scheduling and QoS and do nothing for throttling.

---

### 5. What is the QoS-class impact of removing the CPU limit?

QoS is derived from requests vs. limits across all containers:

- **Guaranteed** — every container has `requests == limits` for **both** CPU and
  memory.
- **Burstable** — at least one request/limit is set but the Guaranteed criteria
  are not met.
- **BestEffort** — no requests or limits at all.

Removing the CPU limit (while keeping requests and a memory limit) moves a pod
from **Guaranteed to Burstable**. Consequences: Burstable pods are **evicted
earlier** than Guaranteed under node memory pressure (OOM `oom_score_adj`
ranking), and they are **not eligible for exclusive pinned cores** under the
static CPU Manager policy. You trade some of that protection for the elimination
of throttling — usually a good trade for a latency-sensitive service, but state
it explicitly.

---

### 6. How does the static CPU Manager policy help?

With `--cpu-manager-policy=static` on the kubelet, **Guaranteed** pods that
request **whole integer** CPUs get **exclusive, pinned** physical cores taken out
of the shared pool. Owning whole cores means the workload isn't subject to CFS
bandwidth quota juggling on those cores and gains cache locality / reduced
context switching. Caveats: only Guaranteed + integer CPU (not `500m`), it
reduces node scheduling flexibility (cores are dedicated), and it is a node-level
setting an app team can't flip per-pod. Ideal for genuinely CPU-bound,
latency-critical services; overkill for most.

---

### 7. How would you alert on this in production?

Alert on the **throttled fraction**, not on utilization:

```promql
rate(container_cpu_cfs_throttled_periods_total{namespace="prod"}[5m])
/
rate(container_cpu_cfs_periods_total{namespace="prod"}[5m])
  > 0.25
```

Fire when a container is throttled in more than ~25% of periods sustained over a
few minutes. Pair it with your latency SLO burn-rate alerts so you can correlate
"P99 spiked" with "throttling spiked". Also surface `throttled_seconds_total`
rate as a dashboard panel next to CPU utilization so the two are seen together —
that juxtaposition is what makes the "CPU fine, latency bad" pattern obvious.

---

### 8. What are the Go and Java thread-count pitfalls specifically?

- **Go:** `GOMAXPROCS` historically defaulted to the number of **host** logical
  CPUs, ignoring the CFS quota, so a `1`-core-limited pod on a 32-core node would
  run 32 OS threads eligible to execute Go code and blow through its quota
  instantly. Fix with **`go.uber.org/automaxprocs`** (sets `GOMAXPROCS` from the
  cgroup quota) or set `GOMAXPROCS` explicitly. Go 1.25+ made the runtime
  cgroup-aware, mitigating this, but automaxprocs is still the reliable answer for
  older binaries.
- **Java:** modern JDKs (8u191+, 10+) are container-aware —
  `Runtime.availableProcessors()` reflects the CFS quota, which correctly sizes
  the common ForkJoinPool, some GC/JIT thread counts, etc. But **thread pools you
  size yourself** don't get this for free: an executor built as
  `availableProcessors() * N`, a Netty `EventLoopGroup`, a servlet-container
  worker pool, or a fixed `newFixedThreadPool(64)` can vastly over-subscribe a
  small quota and self-throttle. Audit explicit pool sizing against the limit.

---

### 9. Can `throttled_time` climb even when the app is under its quota?

Yes, on **old kernels** with the CFS **slice-accounting bug**: leftover per-CPU
runtime slices were reclaimed incorrectly, throttling cgroups that were actually
below their quota. The fix landed around **Linux 4.18** (and was refined through
the 5.x series). If you observe throttling at genuinely low utilization on a
pre-4.18 kernel, **upgrade the kernel first** — you may be fighting a bug, not
your configuration. On current kernels, sustained throttling means real
per-period quota exhaustion (usually parallelism vs. quota), which you address
with the fixes in `solution.md`.

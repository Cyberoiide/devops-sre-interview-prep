# Solution — why the CPU charts lie and how to actually fix it

## TL;DR

The container has a **CPU limit**. Kubernetes enforces that limit with the Linux
**CFS bandwidth controller**, which grants the cgroup a fixed amount of CPU-time
(**quota**) inside each short, fixed window (**period**, default 100 ms). A
multi-threaded app can burn the whole quota in a fraction of the window by
running on several cores at once, after which the kernel **throttles** (freezes)
every thread in the cgroup until the next period boundary. These stalls are
shorter than 100 ms, so they **average out** in any normal CPU dashboard: the
graph shows a modest, comfortable utilization number while the application is
actually being repeatedly starved. That intermittent starvation is your P99.

The real signal is the cgroup's throttling counters, not utilization. The usual
fix for a latency-sensitive service is to **remove the CPU limit** (keep the
request), or align the app's parallelism to the quota.

---

## 1. What a CPU limit actually is

A Kubernetes `limits.cpu` value is not a "speed limit" in any intuitive sense.
It is programmed into the container's cgroup as CFS bandwidth control:

- **`cpu.cfs_period_us`** — the length of each accounting/enforcement window, in
  microseconds. **Default: 100000 µs = 100 ms.**
- **`cpu.cfs_quota_us`** — how much **CPU-time** the cgroup may consume within
  one period, in microseconds. `-1` means unlimited.

(That is the **cgroup v1** naming. Under **cgroup v2** the same two numbers live
in a single file, `cpu.max`, written as `"<quota> <period>"`, e.g. `50000 100000`;
`"max 100000"` means unlimited quota. The concept is identical.)

The quota is CPU-*time* per period, so it can exceed one period's wall-clock by
being spread across multiple cores:

| Kubernetes limit | quota per 100 ms period | meaning |
|------------------|-------------------------|---------|
| `500m`           | 50 ms                   | half a core's worth of time per window |
| `1`              | 100 ms                  | one core fully, per window |
| `2`              | 200 ms                  | two cores fully (200 ms of CPU-time in a 100 ms window) |
| `4`              | 400 ms                  | four cores fully |

Key point: **quota is CPU-time summed across all threads/cores**, but the
**period is wall-clock**. That mismatch is the whole story.

---

## 2. Why the CPU chart lies: averaging vs. per-period enforcement

Enforcement happens on a **~100 ms** cadence. Dashboards report an **average**
over a much larger window — `kubectl top` and most Grafana panels aggregate over
seconds. Even a "per-second" panel averages over 1000 ms, i.e. **ten** CFS
periods.

Worked example. App limited to `1` (quota = 100 ms/period) but running 4 hot
threads on a 4-core node:

- In the first ~25 ms of a 100 ms period, 4 threads × ~25 ms ≈ **100 ms of
  CPU-time** — the entire quota — is consumed.
- The kernel throttles the cgroup for the remaining **~75 ms**. All four threads
  are descheduled. Any in-flight request is frozen for up to 75 ms.
- Over the full 1-second chart window: 1000 ms of CPU-time used out of a possible
  4000 ms → the panel proudly reports **"1.0 cores, 25% of the node"**. Looks
  idle. In reality the app was frozen 75% of every window.

So a healthy-looking average and terrible tail latency are **completely
consistent** under throttling. The average is real; it just answers the wrong
question. Latency is governed by the *distribution within the window*, which
averaging destroys.

Note this holds even at 100 ms-granularity charts if their samples are
period-aligned averages — you need the throttling counters, not finer CPU
sampling, to see it.

---

## 3. Why multi-threading makes it dramatically worse

A single-threaded app limited to `500m` can at worst consume 50 ms of CPU in a
period; since it only runs on one core it takes ~50 ms of wall-clock to do so and
gets throttled for the back half — bad, but bounded to one core's pace.

A multi-threaded app **parallelizes the quota burn**. `N` busy threads drain a
`Q`-millisecond quota in roughly `Q/N` milliseconds of wall-clock, then all `N`
threads are frozen for the rest of the period. The more cores the app uses, the
**earlier** in each window it hits the wall and the **larger** the frozen tail.
This is why the problem bites exactly the workloads people care about: parallel
request handlers, JVM thread pools, Go runtimes with high `GOMAXPROCS`,
connection-per-thread servers.

It also explains the counter-intuitive part: the app can be throttled while the
**node is mostly idle**. Throttling is a property of the cgroup's quota, not of
node contention.

---

## 4. The signal you should actually trust

Stop looking at utilization. Look at throttling.

Raw kernel counters in the cgroup (`cpu.stat`):

- **`nr_periods`** — enforcement windows elapsed.
- **`nr_throttled`** — windows in which the cgroup was throttled.
- **`throttled_time`** (cgroup v1, **nanoseconds**) / **`throttled_usec`**
  (cgroup v2, **microseconds**) — cumulative wall-clock time spent frozen.

The number that matters is the **throttled ratio**: `nr_throttled / nr_periods`.
Anything consistently non-trivial (say > 0.1–0.2, and certainly approaching 1.0)
on a latency-sensitive service is a red flag.

Via cAdvisor/Prometheus the same data is exposed as:

- `container_cpu_cfs_periods_total`
- `container_cpu_cfs_throttled_periods_total`
- `container_cpu_cfs_throttled_seconds_total`

Throttled fraction over time:

```promql
rate(container_cpu_cfs_throttled_periods_total[5m])
/
rate(container_cpu_cfs_periods_total[5m])
```

This is the metric that would have told you the truth from day one, and it is the
metric you should alert on.

---

## 5. Request vs. limit — a critical distinction

This trips up most candidates, so be precise:

- A **CPU request** (`requests.cpu`) sets **`cpu.shares`** (cgroup v1) /
  **`cpu.weight`** (cgroup v2). Shares are a **relative weight** used by the
  scheduler **only when the CPU is contended** — they decide who gets proportionally
  more time when everyone wants it. **A request never throttles anything.** With
  spare CPU, a container can freely exceed its request.
- A **CPU limit** (`limits.cpu`) sets the **CFS quota** (`cpu.cfs_quota_us` /
  `cpu.max`). **Only the limit causes throttling.**

Therefore: **if you have throttling, you have a CPU limit.** Removing the limit
removes the quota and cannot, by construction, throttle. Removing the *request*
would do nothing for throttling (and would hurt scheduling/QoS) — a common wrong
answer.

---

## 6. The full menu of fixes (with honest tradeoffs)

### Fix 1 — Raise or remove the CPU limit (keep the request)

The standard remedy for latency-sensitive, bursty, multi-threaded services.
Remove `limits.cpu` entirely and keep `requests.cpu`. No quota ⇒ no CFS
throttling; the scheduler still reserves a fair share via the request.

Tradeoffs, stated honestly:

- **QoS class changes.** A pod with `requests == limits` on all resources is
  **Guaranteed**. Drop the CPU limit and (assuming a memory limit remains) it
  becomes **Burstable**. Guaranteed pods are evicted last under node pressure and
  are eligible for exclusive cores under the static CPU Manager policy (Fix 4);
  Burstable loses some of that. Know what you are giving up.
- **Noisy-neighbor risk.** Without a limit, the container can consume every spare
  core on the node and starve co-located pods during a burst. Mitigate with sane
  **requests** (so the scheduler bin-packs correctly), node isolation, or
  `LimitRange`/policy. On dedicated nodes this risk is minimal.
- Some orgs mandate limits for cost/chargeback or admission policy. If you cannot
  remove the limit, use Fix 2/3, or set the limit generously above real parallel
  demand.

### Fix 2 — Align parallelism / `GOMAXPROCS` to the quota

If the app spins up threads based on the host's core count, it will always
over-parallelize relative to its quota and self-throttle. Make the app aware of
its quota:

- **Go:** the runtime defaults `GOMAXPROCS` to the number of host cores, not the
  CFS quota. Use **`go.uber.org/automaxprocs`** (or set `GOMAXPROCS` from the
  limit) so the runtime matches, e.g., `GOMAXPROCS=1` for a `1`-core limit.
  Go 1.25+ has since made the runtime cgroup-aware, reducing the need for the
  library, but automaxprocs remains the safe, well-known fix.
- **Java:** the JVM is container-aware for heap and for
  `Runtime.availableProcessors()` on modern JDKs, which sizes the common
  ForkJoinPool, some GC threads, etc. Still verify — thread pools you size
  yourself (e.g. an executor set to `availableProcessors() * N`, or Netty
  worker groups) can badly over-subscribe a small quota.

Tradeoff: this lowers throughput headroom (you are choosing to run less in
parallel), but it makes latency predictable and keeps the limit intact. It does
**not** help if the parallelism is inherent/unavoidable — then use Fix 1.

### Fix 3 — Confirm you are not on a buggy old kernel

Older Linux kernels had a CFS bandwidth **slice-accounting bug** that caused
**spurious throttling even when the app was well under its quota** (leftover
per-CPU slices were reclaimed incorrectly). The fix landed around **Linux 4.18**
and was refined in the **5.x** series (the `cfs_bandwidth` slice changes,
commit `de53fd7aedb1`). If you are on a pre-4.18 kernel and see throttling at low
utilization, upgrade the kernel before anything else — you may be chasing a bug,
not a config problem.

### Fix 4 — Static CPU Manager policy (exclusive pinned cores)

Set the kubelet flag **`--cpu-manager-policy=static`**. Pods in the
**Guaranteed** QoS class that request **integer** CPUs (e.g. `limits.cpu: "2"`,
equal to requests) are then given **exclusive, pinned** physical cores instead of
sharing the CFS pool. Because the app owns whole cores, it does not fight the CFS
bandwidth quota on those cores and avoids this style of throttling, while also
gaining cache locality.

Tradeoffs: only helps **Guaranteed pods with integer CPU limits** (fractional
`500m` need not apply); reduces scheduling flexibility (cores are dedicated);
best for genuinely latency-critical, CPU-bound workloads. It is a node-level
kubelet setting, not something a normal app team can toggle per-pod.

### Non-fix — "just shorten the period"

Shrinking `cpu.cfs_period_us` (say to 10 ms) reduces the size of each stall but
increases scheduling overhead, and — importantly — **vanilla Kubernetes does not
expose the CFS period as a pod-level knob**. (There is a long-standing kubelet
feature-gate discussion around a configurable global period, but it is not a
per-workload setting you can rely on.) Do not offer period tuning as the primary
answer; mention it only to show you know it exists and why it is not the lever.

---

## 7. How to summarize it in the interview

"P99 bad, CPU charts fine" on a multi-threaded pod ⇒ **check for a CPU limit,
then check the throttling counters, not utilization.** The limit is a CFS quota
enforced every ~100 ms; parallel threads burn it early and get frozen for the
rest of the window, and those sub-100 ms freezes average away on the dashboard.
Prove it with `cpu.stat` / `container_cpu_cfs_throttled_*`. Fix it by removing
the limit (keep the request, accept the Burstable/noisy-neighbor tradeoff),
aligning `GOMAXPROCS`/thread pools to the quota, and — for latency-critical
integer-CPU workloads — the static CPU Manager policy. And rule out an ancient
kernel with the known CFS accounting bug.

# Deeper — Interviewer Follow-up Questions

Concise, correct answers to the pushbacks an interviewer will use to test depth.

### Q: A job exits with code 137. What does that tell you, precisely?

137 = 128 + 9, i.e. the process was terminated by signal 9 (SIGKILL). On a
container/runner this is almost always the kernel OOM-killer firing because the
cgroup exceeded its memory limit. Confirm with `dmesg`/`journalctl -k` showing
`Out of memory: Killed process`, or the cgroup's `memory.events` `oom_kill`
counter. It is *not* a graceful shutdown — the process got no chance to clean up.

### Q: How is 137 different from 143, and from 124?

- **143** = 128 + 15 = SIGTERM: a graceful termination request. CI often sends
  SIGTERM to cancel a job (user cancel, `fail-fast`, or a soft timeout) and only
  escalates to SIGKILL if the process ignores it.
- **124** is the exit code GNU `timeout(1)` returns when *it* killed the command
  for exceeding the time budget.
- **137** is a hard kill, no cleanup. Different cause, different fix.

### Q: The same test fails only sometimes. How do you decide if it's a test bug or infra?

Re-run the identical commit several times. If failures move between tests or
disappear, suspect nondeterminism (resource pressure or shared state). If it is
always the *same* test, run it in isolation and under randomized order
(`pytest -p randomly`, `go test -shuffle=on`). If it fails in isolation, it is a
genuine test bug (input scaling, unhermetic setup). If it only fails alongside
other jobs, it is contention.

### Q: Sharding fixed the timeout. When does sharding NOT help?

When the slowness is *algorithmic* per shard rather than volume-based. If a
single test is O(n²) in the number of files and a large PR touches many files,
each shard still runs the quadratic test on its portion, and beyond a point no
amount of sharding saves you. Sharding parallelizes independent work; it cannot
fix a slow individual unit or a serial bottleneck (Amdahl's law). It also adds
fixed per-shard overhead (spin-up, checkout, dependency restore), so past a point
more shards make the pipeline slower, not faster.

### Q: Why not just add automatic retries and move on?

Retries convert an intermittent red into an eventual green, which hides the real
bug — usually shared-state contention that will bite worse later (and can cause
real production incidents from the same non-isolation). They also multiply CI
cost and wall time, and they erode trust in the signal ("just hit re-run" becomes
the culture). Retries are acceptable as a *temporary* guard on a specifically
quarantined flaky test while it is being fixed, not as a blanket policy.

### Q: A large PR OOMs. Do you just give the runner more RAM?

That is the fast fix and sometimes the right one, but it only raises the
threshold — the next bigger PR hits it again, and you are now paying for bigger
runners on every job. Prefer reducing *peak* memory: stream data instead of
loading it all, lower per-worker heap, reduce intra-job parallelism (each worker
has its own footprint), and shard so each job's working set is smaller. Size up
when the work is genuinely irreducible or as a stopgap.

### Q: What are "ephemeral environments" and why do they fix intermittent failures?

An ephemeral environment is a fresh, isolated set of dependencies (database,
message broker, service instances) created for a single job/shard and destroyed
after. Because nothing is shared between concurrent jobs, there is no state to
collide on — the timing-dependent, order-dependent failures vanish. In GitHub
Actions this is `services:` containers per job; in Kubernetes-based runners it is
a fresh pod per job; in general it means unique DB names/schemas, dynamic ports,
and `mktemp` dirs rather than fixed shared resources.

### Q: How would you measure whether the problem is memory, CPU, disk, or fds?

On self-hosted runners, scrape cgroup stats: `memory.peak` / `memory.events` for
OOM, `cpu.stat` throttling for CPU starvation, disk usage on the work volume, and
`/proc/<pid>/fd` count or `ss`/`lsof` for file-descriptor/socket exhaustion. On
hosted runners where you cannot see the host, reproduce locally under a container
with explicit `--memory`, `--cpus`, and `--pids-limit` and bisect which limit
reproduces the failure — exactly what the exercise does.

### Q: Large PRs run more jobs in parallel. Where else can that bite besides shared state?

Rate limits and quotas: more parallel jobs hammering a package registry, a
container registry, an external API, or the CI provider's own concurrency limits
can cause throttling (HTTP 429) or queueing that looks like a timeout. Also
network/disk saturation on a shared self-hosted runner host, and cache stampedes
where many jobs try to populate the same cache key at once. The fix is
concurrency controls (`concurrency:` groups, job-count caps) plus caching and
backoff.

### Q: How would you stop this class of problem from recurring, not just fix today's PR?

Make limits explicit and failures legible: set `timeout-minutes` at step and job
level, set explicit CPU/memory requests and limits, and shard by default. Enforce
test hermeticity and run tests shuffled in CI so order dependence is caught early.
Track flaky tests in a quarantine list with owners. Watch for the "just slow"
signal as an early warning before it becomes a timeout. And keep PRs smaller where
possible — smaller PRs are both a cause fix (less stress per pipeline run) and
better engineering practice.

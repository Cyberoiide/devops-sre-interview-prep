# Solution — Diagnosing and Fixing Large-PR Pipeline Failures

## The mindset: turn "it's flaky" into a triage tree

The failure description mixes four different symptoms (test fails, OOM, timeout,
just slow) and one clue (correlates with PR size). The single most important move
is to **stop treating it as one problem**. PR size is a stress multiplier that
pushes several independent limits toward their breaking point at once. You
diagnose by classifying the failure, not by guessing at a fix.

A structured triage:

```
Pipeline fails on large PR
        |
        +-- What is the exit code / signal of the failing step?
        |     137  -> killed by SIGKILL, almost always OOM         -> Section 2
        |     143  -> SIGTERM (often a graceful cancel / timeout)  -> Section 2
        |     124  -> GNU `timeout` killed it (ran too long)       -> Section 2
        |     1    -> the test's own assertion failed              -> Section 1
        |
        +-- Is it ALWAYS the same test / job that fails?
        |     yes -> test-specific bug, dig into that test          -> Section 1
        |     no, moves around / disappears on re-run -> contention -> Section 3
        |
        +-- Does re-running (same code) change the outcome?
              yes -> nondeterminism: resource OR shared-state       -> Sections 2,3
              no  -> deterministic: it genuinely exceeds a limit    -> Section 2
```

Everything below hangs off this tree.

## Step 0 — Reproduce and gather evidence before changing anything

- **Read the raw logs, not the summary.** CI UIs hide the exit code. Find the
  actual `Process completed with exit code N` line, or the `Killed` message, or
  the "exceeded the maximum execution time" banner.
- **Check the runner's resource graphs** if you have them (self-hosted: node
  exporter / cgroup memory.peak; GitHub-hosted: you often only get the exit
  code, so reproduce locally under a memory limit as in the exercise).
- **Re-run the exact same commit.** If the result changes, you have
  nondeterminism (resource pressure or shared state). If it fails identically
  every time, it is a deterministic limit or a real test bug.
- **Bisect on size if you can.** If PRs under ~N files pass and above ~N fail,
  that threshold itself is a strong hint you are hitting a fixed limit (RAM,
  time, file descriptors, disk).

## Section 1 — Test-specific failures ("is it always the same test?")

This is the first question to ask because it can short-circuit the whole
infrastructure investigation. If one specific test is the one that fails, PR
size is only *exposing* a latent bug in that test.

Common reasons a single test breaks on large PRs:

- **The test's input scales with the PR.** A test that "lints every changed
  file" or "diffs the whole tree" does more work as the PR grows, so it is the
  first to hit a memory or time limit while every other test is fine.
- **Order dependence.** Larger PRs change which tests run and in what order
  (especially with test parallelism). A test that secretly depends on state left
  by another test will pass in one order and fail in another. Confirm with a
  randomized-order run (`pytest -p randomly`, `go test -shuffle=on`,
  `jest --shuffle`).
- **Hidden global state / caches** the test reads that a big PR happens to
  populate differently.

**Fixes:** make the test independent and hermetic (set up and tear down its own
fixtures, no reliance on execution order), cap or stream its input, and if it is
genuinely flaky, **quarantine it** (mark it as known-flaky so it runs but does
not block the merge) while it is fixed — do *not* paper over it with blanket
retries across the whole suite.

## Section 2 — Resource contention and exhaustion (OOM / timeout / slow)

This is the "the runner is too small for the work a large PR generates" family.
Large diffs mean more to compile, more files to lint, larger test matrices,
bigger build artifacts, more Docker layers, larger caches to restore. All of it
pushes CPU, RAM, disk, file descriptors, and wall-clock toward hard limits.

**OOM (exit 137).** The Linux OOM-killer sends SIGKILL when a cgroup exceeds its
memory limit (a container `--memory`, a Kubernetes pod limit, or physical RAM on
a bare runner). Signatures: exit 137, `Killed` in output, `dmesg` shows
`Out of memory: Killed process`. Fixes, cheapest first:
  - **Size the runner up** (bigger instance / larger `resources.limits.memory`).
    Fast, costs money, and only pushes the threshold higher.
  - **Reduce peak memory**: stream instead of loading everything, lower per-JVM
    heap, reduce intra-job parallelism (each parallel worker has its own memory
    footprint — 8 workers on a 2-core runner can OOM where 2 workers would not).
  - **Shard across more jobs** so each job's working set is smaller.

**Timeout (exit 124 / cancelled after N minutes).** The job did not crash; it was
cut off. Either the work genuinely got slower (bigger PR = more to do) or it
stalled (a hung network call, a deadlock, waiting on a resource that never frees
— which overlaps with Section 3). Fixes:
  - **Shard/parallelize** so each job does a fraction of the work.
  - **Cache aggressively** (dependencies, build outputs, Docker layers) so large
    PRs are not rebuilding the world.
  - **Profile the slow step.** If the suite is O(n²) in the number of files, more
    runners only delays the wall; the algorithm needs fixing.
  - **Set `timeout-minutes` at the step and job level** so a hang fails fast with
    a clear signal instead of burning the whole job budget.

**Just slow (passes eventually).** Same root cause as timeout, below the cutoff.
Treat it as an early warning: it will become a timeout on the next bigger PR. Add
caching, sharding, and right-size the runner before it tips over.

**Runner sizing and parallelism, stated cleanly:**
- Give jobs *explicit* resource requests/limits so behavior is predictable
  instead of "whatever the host had free."
- Match intra-job worker count to the runner's cores/RAM; oversubscription causes
  both OOM and thrash-induced slowness.
- Prefer **more small jobs (sharding)** over one giant job — it parallelizes wall
  time and keeps each job's footprint small.

## Section 3 — State and concurrency contention (the intermittent killer)

This is the family that produces the maddening "fails sometimes, passes on
re-run, moves to a different job" behavior. Large PRs trigger more jobs and more
parallel shards, so the probability of a collision on shared mutable state rises.

Shared-state offenders:
- A **single shared test database** that parallel jobs read and write, so one
  job sees another's rows.
- **Hard-coded ports** (`localhost:8080`) so two jobs on the same runner fight
  over the socket.
- **Fixed temp paths** (`/tmp/testdata`, a fixed fixture dir) that jobs clobber.
- **Shared cache directories** written concurrently, causing corrupt/partial
  reads.
- **Order-dependent tests** (also in Section 1) whose interleaving changes under
  parallelism.

Why it is intermittent: the outcome depends on the *timing* between concurrent
jobs, which is nondeterministic. Small PRs run few jobs and rarely collide; large
PRs run many and collide often. Re-running reshuffles the timing, so the failure
appears to "go away."

**The fix is isolation, not retries:**
- **Ephemeral per-job environments.** Spin up a fresh database/service container
  per job (or per shard) and tear it down after. In Actions, `services:` gives
  each job its own containers; in Kubernetes-based runners, a fresh pod per job.
- **Dynamic ports** (bind to port 0 and read back the assigned port) instead of
  fixed ones.
- **Unique temp dirs** (`mktemp -d`) and **namespaced cache keys** (include the
  job/shard id).
- **Make tests hermetic** so order does not matter, and run them shuffled in CI
  to catch order dependence early.

Retrying a flaky job masks the collision and lets the bug rot; it also inflates
CI time and cost. Retries are a stopgap while you build isolation, not the cure.

## Bringing it together — the answer you give out loud

"First I'd classify the failure instead of treating it as one bug. I'd pull the
raw exit code: 137 means OOM, 124 or a 'max execution time' banner means timeout,
1 means the test itself failed. Then I'd ask whether it's always the same test —
if it is, it's a test bug that big PRs expose, and I'd look at that test's input
scaling and order dependence. If the failure moves around or disappears on
re-run, that's nondeterminism: either the runner is resource-starved (fix with
right-sizing, reduced peak memory, caching, and sharding) or jobs are colliding
on shared state like a common DB, fixed ports, or temp paths (fix with ephemeral
per-job environments and unique resources). Sharding buys headroom on both time
and memory. I'd treat retry-until-green as a smell — it hides the shared-state
bug and inflates cost. And I'd add explicit resource limits and `timeout-minutes`
so failures are fast and legible instead of mysterious."

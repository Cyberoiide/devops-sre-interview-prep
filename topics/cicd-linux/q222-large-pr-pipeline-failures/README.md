# Q222 — The CI/CD Pipeline That Falls Apart on Large PRs

## Scenario (as posed in the interview)

> "Your team's CI/CD pipeline works reliably for small pull requests. But as PRs
> get larger — more files changed, more commits, bigger diffs — the pipeline
> starts failing *inconsistently*. Sometimes a test fails. Sometimes a runner
> runs out of memory and gets killed. Sometimes a job just times out. Sometimes
> the whole thing is simply slow but eventually passes. Re-running the pipeline
> often makes the failure go away, or move to a different job.
>
> Walk me through how you'd investigate this. What are the likely root causes,
> and how would you fix them?"

## Why this question is deliberately vague

The interviewer is not looking for one memorized answer. They are testing whether
you can take an ambiguous, "it's flaky and I don't know why" production complaint
and turn it into a *structured investigation*. The symptom ("fails on large PRs")
correlates with PR size, but correlation is the starting clue, not the diagnosis.

Large PRs are a stress multiplier. They push every part of the pipeline closer to
some limit. Your job is to figure out *which* limit is being hit, because the fix
is completely different depending on the cause.

## The three families of root cause you should reach for

Strong answers organize the chaos into a small number of hypotheses:

1. **Test-specific failures.**
   Is it always the *same* test that fails? If so, PR size is a red herring — the
   real question is what about that test breaks when it sees more input, more
   files, or a different execution order. This is a correctness/flakiness bug in
   one test, not an infrastructure problem.

2. **Resource contention / exhaustion on runners.**
   Larger PRs mean more code to compile, more files to lint, bigger test
   matrices, larger artifacts, more Docker layers. That drives up CPU, RAM, disk,
   and wall-clock time. When a GitHub Actions runner or Jenkins agent hits its
   memory ceiling the kernel OOM-kills a process (exit 137); when it hits the job
   time limit the job is cancelled (timeout); when it is just under-provisioned
   everything is slow. These are *provisioning and limits* problems.

3. **State / concurrency contention.**
   Bigger PRs touch more of the system, so they trigger more jobs, more parallel
   test shards, and more code paths at once. If those jobs share mutable state —
   a single database, fixed TCP ports, temp files at hard-coded paths, a shared
   cache directory — they collide. Test order changes between runs, so the
   collision is intermittent. These are *isolation* problems.

The rest of this exercise walks through a triage methodology that separates these
three, then the concrete fixes: runner sizing, sharding/parallelism, resource
limits (cgroups/`ulimit`/container memory), ephemeral per-job environments, and
flaky-test isolation.

## What a good answer demonstrates

- A **triage method**: reproduce, read the exit codes/logs, classify the failure
  before guessing at fixes.
- Knowing that **exit 137 = SIGKILL (usually OOM)** and a job "cancelled after N
  minutes" = timeout — different signatures, different fixes.
- The instinct to ask **"is it always the same test?"** before blaming
  infrastructure.
- Understanding that **shared state + parallelism = intermittent failures**, and
  that the durable fix is *isolation*, not retries.
- Treating **retry-until-green as a smell**, not a solution.

Work through `exercise.md` to see all three failure modes reproduced in a tiny
local pipeline, then read `solution.md` for the full walkthrough and `deeper.md`
for the follow-up questions an interviewer will push on.

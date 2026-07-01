# q195 — Working around Lambda cold starts

## The interview question

> "Your Lambda function is usually fast — single-digit milliseconds — but every
> so often a request takes over a second, and it's worse right after a deploy or
> a traffic spike, and it wrecks your P99. What's happening, and what levers do
> you have to reduce it? Assume I'll ask you to distinguish 'made it rarer' from
> 'made it not matter.'"

This tests whether you actually understand the Lambda **execution model** — not
just that "cold starts are a thing." The strong answer defines precisely what a
cold start *is*, breaks the latency into its phases, and then gives levers in
two buckets: **make init cheaper/faster** and **eliminate the cold start from
the hot path** (Provisioned Concurrency / SnapStart).

## What a cold start actually is

When a request arrives and there's **no warm execution environment** available,
Lambda must **create a new one**: download your code/layers, start the runtime
(JVM, Node, Python interpreter…), and run your **initialization code** — the
stuff *outside* the handler (imports, SDK clients, DB connection setup, config
loading) — **before** it can invoke your handler for the first time. That
one-time setup is the **cold start**. Once the environment exists, subsequent
invocations reuse it (a **warm start**) and skip all of that.

Cold starts cluster exactly where the scenario says: **after a deploy** (all
environments are new), during **scale-up** (bursts spin up fresh environments),
and after idle **scale-to-zero** (Lambda reclaims idle environments). They hit
**P99/tail latency**, rarely the median.

## The two buckets of fixes

**A. Make the cold start cheaper (reduce init cost)**

- **Do less before the handler.** Lazy-load heavy deps, don't import what a given
  path won't use, defer expensive work until first actually needed.
- **Reuse across invocations.** Create SDK/DB clients **once at init**, in the
  global scope, so warm invocations reuse them. For relational DBs behind
  connection limits, use **RDS Proxy** to pool connections.
- **Shrink the package.** Smaller deployment/layers = faster download & init.
  Trim dependencies, tree-shake, avoid giant monolithic bundles.
- **Pick a faster runtime.** Interpreted/lightweight runtimes (Node, Python, Go)
  start far faster than a cold JVM/.NET.

**B. Take the cold start off the hot path**

- **Provisioned Concurrency** — keep **N environments pre-initialized and warm**
  so those invocations have **no cold start at all**. Pair with **scheduled
  scaling** (Application Auto Scaling) to raise it *before* a known peak.
- **SnapStart** (Java, and now other runtimes) — Lambda snapshots the
  initialized environment and **restores from the snapshot** instead of
  re-running init, cutting Java cold starts dramatically at **no extra
  steady-state cost** (unlike Provisioned Concurrency, which you pay for).
- **Keep-warm pings** — a scheduled invocation to stop scale-to-zero. Crude,
  doesn't help bursts, and largely superseded by the above; mention it, don't
  lead with it.

## What you should be able to answer

- A precise definition of a cold start and its **phases** (download → runtime
  init → **your init code** → handler).
- **Init code vs handler code**, and why work in the global scope is the
  expensive part you control.
- Provisioned Concurrency **vs** SnapStart — what each does and their cost trade.
- Why **requests → new environments** (concurrency model) is what generates cold
  starts, and how bursts/deploys/idle trigger them.

## Files in this topic

- `exercise.md` — a tiny Python Lambda (runnable locally, or on LocalStack, or
  documented for real AWS) that shows init-time work in the global scope vs the
  handler, **prints the cold-vs-warm marker in the logs**, and lets you measure
  the difference; plus notes on Provisioned Concurrency config.
- `solution.md` — the full model, the phase breakdown, and every lever with the
  trade-offs.
- `deeper.md` — follow-up interview questions with concise answers.

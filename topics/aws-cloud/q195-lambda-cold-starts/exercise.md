# Exercise — See a Lambda cold start, and move work off the hot path

This lab uses a tiny Python Lambda that makes the **cold vs warm** distinction
visible: it does deliberate init-time work in the **global scope** and logs a
marker showing whether the invocation reused a warm environment. You'll (A) run
the exact same handler logic **locally** with a trivial harness to internalize
init-vs-handler, (B) deploy it to **LocalStack** and watch the cold/warm marker
in real invocations, and (C) read the notes on **Provisioned Concurrency** for
the real-AWS lever.

## What's honest about this lab

- **The init-vs-handler concept runs 100% locally** (Part A) — pure Python, no
  cloud, no cost. This is where the *understanding* lives.
- **LocalStack Community runs real Lambda invocations** (Part B) and will show a
  genuine cold start on the first call and warm reuse after — free.
- **Provisioned Concurrency and SnapStart are real-AWS features** —
  Provisioned Concurrency is **not** in LocalStack Community and **costs money**
  on real AWS (you pay for the pre-warmed capacity). Part C documents the config;
  don't leave it enabled.

## Prerequisites

- Python 3.x for Part A (no packages needed).
- `docker` + [LocalStack](https://docs.localstack.dev/getting-started/) + the
  `awslocal` wrapper for Part B.

---

## The function

`handler.py`:

```python
import time
import os

# ------------------------------------------------------------------
# INIT / GLOBAL SCOPE — runs ONCE per execution environment (the cold
# start). Anything expensive here is paid on every cold start, not per
# request. This is the code you optimize.
# ------------------------------------------------------------------
COLD = True   # flipped to False after the first invocation in this env

def _expensive_init():
    # Simulate heavy init: importing big libs, building SDK/DB clients,
    # loading config. Reuse the RESULT across warm invocations.
    time.sleep(0.5)          # pretend this is 500ms of client/connection setup
    return {"db_client": "connected", "pid": os.getpid()}

RESOURCES = _expensive_init()   # <-- created once, reused by warm invocations


# ------------------------------------------------------------------
# HANDLER — runs on EVERY invocation. Keep it lean; reuse RESOURCES.
# ------------------------------------------------------------------
def handler(event, context):
    global COLD
    was_cold = COLD
    COLD = False

    start = time.time()
    # ... real per-request work would go here; it reuses RESOURCES ...
    result = {"reused": RESOURCES["db_client"], "pid": RESOURCES["pid"]}
    took_ms = round((time.time() - start) * 1000, 2)

    marker = "COLD START" if was_cold else "warm"
    print(f"[{marker}] pid={result['pid']} handler_ms={took_ms}")
    return {"cold_start": was_cold, "handler_ms": took_ms, **result}
```

The teaching point is the placement: `_expensive_init()` runs in the **global
scope** (paid once, on the cold start), and the handler **reuses `RESOURCES`**
instead of rebuilding the connection every request.

---

## Part A — Run it locally to internalize init vs handler (free, no cloud)

`local_run.py`:

```python
import time, handler   # importing handler.py runs its global/init code NOW

print("--- module imported (this was the 'cold start' init) ---")
for i in range(3):
    t0 = time.time()
    out = handler.handler({}, None)
    print(f"invocation {i}: {out}  wall_ms={round((time.time()-t0)*1000,2)}")
    time.sleep(0.2)
```

```bash
python3 local_run.py
```

**What you should see:** the `import handler` line pays the 500ms init **once**
(that's the cold start). Then invocation 0 logs `[COLD START]`, and invocations
1–2 log `[warm]` with the **same pid** and a tiny `handler_ms` — because they
reuse `RESOURCES` instead of re-running `_expensive_init()`. Move
`_expensive_init()` *inside* `handler()` and re-run: now every invocation pays
500ms. That is exactly the mistake cold-start tuning is about — **keep reusable
setup in the global scope, out of the per-request path.**

---

## Part B — Deploy to LocalStack and watch a real cold/warm marker (free)

```bash
localstack start -d

# Package and create the function.
zip -j /tmp/fn.zip handler.py
awslocal lambda create-function \
  --function-name coldstart-demo \
  --runtime python3.12 \
  --handler handler.handler \
  --zip-file fileb:///tmp/fn.zip \
  --role arn:aws:iam::000000000000:role/irrelevant     # LocalStack ignores the role

# First invoke = COLD (new environment must be created + init run).
awslocal lambda invoke --function-name coldstart-demo /tmp/out1.json >/dev/null
cat /tmp/out1.json      # "cold_start": true

# Immediately invoke again = warm (environment reused).
awslocal lambda invoke --function-name coldstart-demo /tmp/out2.json >/dev/null
cat /tmp/out2.json      # "cold_start": false
```

The first response has `"cold_start": true`; the second reuses the environment
and reports `false`. That's the phenomenon end to end. (Idle environments get
reclaimed, so if you wait a while and invoke again you may see another cold
start — the same thing that happens after real scale-to-zero.)

Teardown:

```bash
awslocal lambda delete-function --function-name coldstart-demo
localstack stop
```

---

## Part C — Provisioned Concurrency & SnapStart (real AWS, documented)

These are the "take the cold start off the hot path" levers. They need a real
account (Provisioned Concurrency **costs money** and isn't in LocalStack
Community). Config you'd use:

**Provisioned Concurrency** keeps N environments **pre-initialized and warm** on
a published **version/alias** (you can't set it on `$LATEST`):

```bash
# Publish a version and point an alias at it.
aws lambda publish-version --function-name coldstart-demo
aws lambda create-alias --function-name coldstart-demo \
  --name prod --function-version 1

# Keep 5 environments warm — those invocations have NO cold start.
aws lambda put-provisioned-concurrency-config \
  --function-name coldstart-demo --qualifier prod \
  --provisioned-concurrent-executions 5
```

Pair it with **scheduled scaling** (Application Auto Scaling) to raise the
provisioned count *before* a known daily peak and lower it after, so you're not
paying for warm capacity around the clock:

```bash
aws application-autoscaling register-scalable-target \
  --service-namespace lambda \
  --resource-id function:coldstart-demo:prod \
  --scalable-dimension lambda:function:ProvisionedConcurrency \
  --min-capacity 1 --max-capacity 20
# ...then put-scheduled-action to bump min-capacity ahead of the peak.
```

**SnapStart** (Java, and expanded to other runtimes) is the cheaper alternative
for slow-initializing runtimes: Lambda snapshots the **post-init** environment
and **restores from the snapshot** instead of re-running init, so you get much
faster cold starts with **no charge for idle warm capacity**. Enable it on the
function config (`SnapStart: {ApplyOn: PublishedVersions}`) — note the
uniqueness caveats (don't cache randomness/connections that must be per-instance;
use the runtime hooks to re-establish them after restore).

- [ ] Real-AWS check: enable Provisioned Concurrency on the `prod` alias, invoke
      via the alias, and confirm `cold_start` is `false` even on the first call
      after a deploy. **Delete the provisioned-concurrency config afterward** —
      it bills for the warm capacity continuously.

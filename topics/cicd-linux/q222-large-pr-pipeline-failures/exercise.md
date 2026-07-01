# Exercise — Reproduce and Fix the Three Failure Modes

This lab builds a tiny "CI job" as a shell script whose behavior degrades as the
input (the simulated PR size) grows. You will reproduce an OOM kill, a timeout,
and a shared-state collision, then fix each one. Everything runs locally with
bash and Docker. No cloud CI account needed.

Estimated time: 30–45 minutes.

## Prerequisites

- Linux (or WSL2 / a Linux VM). The OOM and cgroup parts need a real Linux kernel.
- `docker` installed and runnable by your user.
- `bash`, `awk`, `python3` available.

Create a working directory:

```bash
mkdir -p ~/ci-lab && cd ~/ci-lab
```

---

## Part 1 — The OOM kill (exit 137)

Create a "test" that allocates memory proportional to PR size.

```bash
cat > mem_test.py <<'EOF'
import sys

# "PR size" in MB of memory the test tries to hold at once.
pr_size_mb = int(sys.argv[1]) if len(sys.argv) > 1 else 50

print(f"[mem_test] simulating a PR that needs {pr_size_mb} MB", flush=True)
chunk = bytearray(1024 * 1024)          # 1 MB
held = []
for i in range(pr_size_mb):
    held.append(bytearray(chunk))       # keep a reference so it is not freed
print(f"[mem_test] allocated {pr_size_mb} MB, test passed", flush=True)
EOF
```

Run it in a container with a hard 128 MB memory limit — this mimics a runner with
a fixed RAM ceiling:

```bash
# Small PR: fits comfortably. Exit code 0.
docker run --rm --memory=128m -v "$PWD":/w -w /w python:3.12-slim \
  python mem_test.py 50
echo "exit=$?"

# Large PR: exceeds the limit. The kernel OOM-killer fires. Exit code 137.
docker run --rm --memory=128m -v "$PWD":/w -w /w python:3.12-slim \
  python mem_test.py 300
echo "exit=$?"
```

**Observe:** the large run dies with `exit=137`. That is `128 + 9` = killed by
signal 9 (SIGKILL), which the OOM-killer sends. Confirm on the host that the
kernel logged it:

```bash
dmesg | grep -i -E 'killed process|oom' | tail -5   # may need sudo
```

This is failure mode #2. The "test" is fine; the *runner is too small for the
work the large PR generates*.

**Fix A — give the job more memory (runner sizing):**

```bash
docker run --rm --memory=512m -v "$PWD":/w -w /w python:3.12-slim \
  python mem_test.py 300
echo "exit=$?"   # now 0
```

**Fix B — make the work smaller per job (streaming instead of holding it all).**
The better fix is often to not hold everything in memory at once. Edit
`mem_test.py` so it does not accumulate `held` — process and release each chunk.
Then even 300 "MB of PR" passes under the 128 MB limit because peak usage stays
flat. This is the real-world lesson: throwing RAM at the problem is the quick
fix; reducing peak memory is the durable one.

---

## Part 2 — The timeout

Create a job whose runtime grows super-linearly with PR size (a stand-in for an
O(n²) test suite, an unsharded lint over every file, etc.):

```bash
cat > slow_test.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
pr_files="${1:-10}"
echo "[slow_test] processing $pr_files files"
# Simulate per-pair work: quadratic in the number of files.
for ((i=0; i<pr_files; i++)); do
  for ((j=0; j<pr_files; j++)); do
    : # pretend to compare file i against file j
  done
  sleep 0.05   # per-file fixed cost (I/O, spawning a process, etc.)
done
echo "[slow_test] done"
EOF
chmod +x slow_test.sh
```

Now enforce a job timeout the way CI does, with `timeout(1)`:

```bash
# Small PR (10 files): finishes within the 5s job limit.
timeout 5s ./slow_test.sh 10 ; echo "exit=$?"

# Large PR (200 files): blows past the 5s limit and is killed. Exit 124.
timeout 5s ./slow_test.sh 200 ; echo "exit=$?"
```

**Observe:** exit code `124` is `timeout`'s signal that it killed the command for
running too long. This is failure mode #2 again but in its *time* dimension: the
job did not crash, it was cut off. In GitHub Actions this shows as
"The job running on runner ... has exceeded the maximum execution time" or a
step `timeout-minutes` cancellation.

**Fix — sharding.** Split the work across N parallel jobs so each shard sees only
a fraction of the files:

```bash
cat > sharded.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
total_files="${1:-200}"
shards="${2:-4}"
pids=()
for ((s=0; s<shards; s++)); do
  # Each shard handles total/shards files.
  per=$(( total_files / shards ))
  ( ./slow_test.sh "$per" >/dev/null 2>&1 ) &
  pids+=($!)
done
for p in "${pids[@]}"; do wait "$p"; done
echo "all shards done"
EOF
chmod +x sharded.sh

# 200 files across 4 shards → each shard does 50, finishes well under 5s.
timeout 5s ./sharded.sh 200 4 ; echo "exit=$?"
```

The lesson: parallelism/sharding turns one job that exceeds the limit into N jobs
that each fit. (Note the quadratic term still bites eventually — sharding buys
headroom, it does not fix an algorithmically slow test.)

---

## Part 3 — Shared-state contention (the intermittent one)

This is the sneaky failure: it only shows up *sometimes*, because it depends on
timing between parallel jobs.

Simulate two test jobs that both write to the same "shared resource" (a fixed
file path — think a shared test DB, a hard-coded port, or `/tmp/testdata`):

```bash
cat > shared_state_test.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
name="$1"
shared="/tmp/ci_shared_resource"      # BUG: hard-coded shared path

echo "$name" > "$shared"              # each job claims the resource
sleep 0.2                             # do some "work"
owner="$(cat "$shared")"              # expect to still own it

if [[ "$owner" != "$name" ]]; then
  echo "[$name] FAIL: resource was clobbered by '$owner'"
  exit 1
fi
echo "[$name] OK"
EOF
chmod +x shared_state_test.sh
```

Run the two jobs in parallel, several times. Because ordering is nondeterministic,
you will see it pass sometimes and fail sometimes:

```bash
for run in 1 2 3 4 5; do
  echo "=== run $run ==="
  ./shared_state_test.sh jobA & ./shared_state_test.sh jobB &
  wait
done
```

**Observe:** intermittent `FAIL: resource was clobbered`. Nothing about the tests
changed between runs — only the interleaving did. On a small PR you might run one
job and never see it; on a large PR you run many jobs in parallel and the
collision surfaces. This is failure mode #3.

**Fix — isolation.** Give each job its own resource instead of a shared one:

```bash
cat > isolated_test.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
name="$1"
shared="$(mktemp -d)/resource"        # FIX: per-job unique path
trap 'rm -rf "$(dirname "$shared")"' EXIT

echo "$name" > "$shared"
sleep 0.2
owner="$(cat "$shared")"
[[ "$owner" == "$name" ]] && echo "[$name] OK" || { echo "[$name] FAIL"; exit 1; }
EOF
chmod +x isolated_test.sh

for run in 1 2 3 4 5; do
  echo "=== run $run ==="
  ./isolated_test.sh jobA & ./isolated_test.sh jobB &
  wait
done
```

Now it passes every time. The same principle scales up: ephemeral per-job
databases (a fresh container per shard), randomized ports, unique temp dirs,
namespaced cache keys.

---

## Part 4 (optional) — Put it in a real GitHub Actions workflow

If you want to see these in a real pipeline, drop this in
`.github/workflows/large-pr-demo.yml` in a scratch repo. It shows a matrix that
sits under the resource limit and a `timeout-minutes` guard.

```yaml
name: large-pr-demo
on: [workflow_dispatch]
jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 5           # job-level timeout guard
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4]      # 4-way sharding
    steps:
      - uses: actions/checkout@v4
      - name: Run this shard only
        run: |
          echo "Running shard ${{ matrix.shard }} of 4"
          # In a real suite: pass --shard to your test runner, e.g.
          # pytest --splits 4 --group ${{ matrix.shard }}
          ./slow_test.sh 50 || true
```

To watch an OOM in Actions, you would allocate more than the runner's ~7 GB (or
use a container with `--memory`). The exit-137 signature is identical to Part 1.

---

## What you should be able to explain after this lab

- The **exit-code fingerprints**: `137` = OOM/SIGKILL, `124` = `timeout` killed
  it, `1` = the test's own assertion failed.
- Why **shared state produces intermittent, order-dependent failures** and why
  isolation (not retries) is the fix.
- Why **sharding** converts one over-limit job into N jobs that fit, and its
  limit (it does not fix algorithmic blowup).
- The trade-off between **sizing up the runner** (fast, costs money) and
  **reducing the work** (slower to build, durable).

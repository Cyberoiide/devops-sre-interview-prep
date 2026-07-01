# Exercise — reproduce and fix CFS throttling

Goal: deploy a multi-threaded CPU burner with a **low CPU limit**, watch it get
throttled (while average CPU looks modest), prove it by reading the cgroup
`cpu.stat`, then **remove/raise the limit** and watch throttling stop.

Everything below uses public images (`polinux/stress`, which ships `stress`) so
there is nothing to build.

---

## 0. Prerequisites

- `docker` (or podman) running
- `kubectl`
- `kind` **or** `minikube`
- A machine with **at least 2 physical CPUs** — the whole point is parallelism.
  Check: `nproc` (should be >= 2, ideally 4+).

Create a cluster:

```bash
# Option A: kind
kind create cluster --name cfs-lab

# Option B: minikube (give it real cores!)
minikube start --cpus=4
```

Confirm the node has multiple CPUs, otherwise you cannot demonstrate the
"burn quota in parallel" effect:

```bash
kubectl get nodes -o jsonpath='{.items[0].status.capacity.cpu}{"\n"}'
```

---

## 1. Deploy the throttled pod

We limit the container to **500m (half a core)** but run **4 CPU workers**.
Four threads want 4 cores of work; the quota only allows the equivalent of
0.5 cores per 100 ms window, so the container will spend most of every window
throttled.

`throttled-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cfs-throttle
  labels:
    app: cfs-throttle
spec:
  containers:
    - name: burner
      image: polinux/stress:latest
      # 4 workers, each a busy CPU loop. sqrt() keeps them hot.
      command: ["stress"]
      args: ["--cpu", "4", "--timeout", "600s"]
      resources:
        requests:
          cpu: "250m"
          memory: "64Mi"
        limits:
          cpu: "500m"       # <-- the culprit: 50ms quota per 100ms period
          memory: "128Mi"
```

```bash
kubectl apply -f throttled-pod.yaml
kubectl wait --for=condition=Ready pod/cfs-throttle --timeout=60s
```

---

## 2. Observe: CPU looks "fine" but latency would be terrible

If `metrics-server` is installed (`minikube addons enable metrics-server`, or
install it on kind), the average will read at or just under the limit:

```bash
kubectl top pod cfs-throttle
# NAME           CPU(cores)   MEMORY(bytes)
# cfs-throttle   500m         3Mi
```

`kubectl top` reports an **average over its sampling window** — it cannot show
you the sub-100 ms freezes. The average being pinned at ~500m is itself a hint,
but on a real service that only spikes occasionally you would see a comfortable
number like "600m used of a 1000m limit" and conclude everything is fine. It is
not. The next step shows the truth.

---

## 3. Prove throttling: read the cgroup `cpu.stat`

This is the reliable demo. The kernel exposes throttling counters directly.
The path differs between cgroup v2 (modern: kind on a recent kernel, most 2023+
distros) and cgroup v1 (older).

First, figure out which cgroup version the node uses:

```bash
kubectl exec cfs-throttle -- sh -c 'test -f /sys/fs/cgroup/cpu.stat && echo "cgroup v2" || echo "cgroup v1"'
```

### cgroup v2 (unified hierarchy)

```bash
kubectl exec cfs-throttle -- cat /sys/fs/cgroup/cpu.stat
```

Look for these lines:

```
usage_usec 12345678
nr_periods 500
nr_throttled 470        <-- number of enforcement windows where we hit the cap
throttled_usec 24500000 <-- total wall-clock time spent frozen (microseconds)
```

The quota/period is in `cpu.max`:

```bash
kubectl exec cfs-throttle -- cat /sys/fs/cgroup/cpu.max
# 50000 100000   <-- quota=50000us (50ms), period=100000us (100ms) == 500m
```

### cgroup v1 (legacy)

```bash
kubectl exec cfs-throttle -- cat /sys/fs/cgroup/cpu,cpuacct/cpu.stat
```

Look for:

```
nr_periods 500
nr_throttled 470
throttled_time 24500000000   <-- NOTE: nanoseconds in cgroup v1
```

And the quota/period:

```bash
kubectl exec cfs-throttle -- sh -c 'cat /sys/fs/cgroup/cpu,cpuacct/cpu.cfs_quota_us /sys/fs/cgroup/cpu,cpuacct/cpu.cfs_period_us'
# 50000    <-- cpu.cfs_quota_us  (50ms)
# 100000   <-- cpu.cfs_period_us (100ms)
```

### Watch it climb

Run the read twice, ~10 seconds apart, and diff the throttled counters:

```bash
# cgroup v2 example
kubectl exec cfs-throttle -- sh -c 'grep -E "nr_periods|nr_throttled|throttled_usec" /sys/fs/cgroup/cpu.stat'
sleep 10
kubectl exec cfs-throttle -- sh -c 'grep -E "nr_periods|nr_throttled|throttled_usec" /sys/fs/cgroup/cpu.stat'
```

You should see `nr_throttled` and `throttled_usec` increasing steadily. A
throttled-ratio near 1.0 (`nr_throttled / nr_periods`) means the container is
being frozen in **almost every single window** — that is exactly what turns into
P99 latency spikes on a request-serving app.

### (Optional) via cAdvisor / Prometheus

If you scrape the kubelet's cAdvisor endpoint, the same signal is exposed as:

```
container_cpu_cfs_throttled_periods_total
container_cpu_cfs_periods_total
container_cpu_cfs_throttled_seconds_total
```

A useful PromQL alerting expression (throttled fraction over 5 min):

```promql
rate(container_cpu_cfs_throttled_periods_total{pod="cfs-throttle"}[5m])
/
rate(container_cpu_cfs_periods_total{pod="cfs-throttle"}[5m])
```

The `cpu.stat` read above is the guaranteed-to-work version; the metric is the
production-monitoring version.

---

## 4. Fix it: raise or remove the limit

The most common fix for a latency-sensitive service is to **remove the CPU limit
entirely while keeping the request** (keeps scheduling/QoS behaviour, removes the
quota that causes throttling). We do that here.

`fixed-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cfs-fixed
  labels:
    app: cfs-fixed
spec:
  containers:
    - name: burner
      image: polinux/stress:latest
      command: ["stress"]
      args: ["--cpu", "4", "--timeout", "600s"]
      resources:
        requests:
          cpu: "250m"       # request kept: scheduler still reserves a share
          memory: "64Mi"
        limits:
          memory: "128Mi"   # memory limit kept; NO cpu limit == no CFS quota
```

```bash
kubectl apply -f fixed-pod.yaml
kubectl wait --for=condition=Ready pod/cfs-fixed --timeout=60s
```

Now read the same counters — on a node with spare cores, `nr_throttled` should
stay flat (throttling stopped because there is no quota to enforce):

```bash
# cgroup v2
kubectl exec cfs-fixed -- cat /sys/fs/cgroup/cpu.max
# max 100000   <-- "max" means UNLIMITED quota == no throttling

kubectl exec cfs-fixed -- sh -c 'grep -E "nr_periods|nr_throttled" /sys/fs/cgroup/cpu.stat'
sleep 10
kubectl exec cfs-fixed -- sh -c 'grep -E "nr_periods|nr_throttled" /sys/fs/cgroup/cpu.stat'
# nr_throttled should NOT be climbing
```

`kubectl top pod cfs-fixed` will now show the app using the cores it actually
wants (up to ~4 cores if the node has them), because nothing is capping it.

### Alternative fix: just raise the limit

If policy requires a limit, raising it so the quota comfortably exceeds the
real parallel demand also stops throttling. For 4 busy workers, a limit of `"4"`
(quota = 400 ms per 100 ms period, spread across 4 cores) removes the stalls.
Set `limits.cpu: "4"` and repeat the `cpu.stat` check.

---

## 5. Cleanup

```bash
kubectl delete pod cfs-throttle cfs-fixed --ignore-not-found

# and tear down the cluster
kind delete cluster --name cfs-lab
# or
minikube delete
```

---

## What you just demonstrated

1. A **CPU limit** became a CFS **quota per 100 ms period**.
2. A **multi-threaded** workload drained that quota early and got **frozen** for
   the rest of each window — sub-100 ms stalls invisible to averaged CPU charts.
3. The **truth was in `cpu.stat`** (`nr_throttled` / `throttled_usec`), not in
   `kubectl top`.
4. **Removing the limit** (keeping the request) eliminated the quota and the
   throttling.

# Q216 — Hands-on lab: measure real latencies on your machine

The table in `README.md` is worth internalizing, but numbers you *measure
yourself* stick far better. This lab benchmarks the whole hierarchy on your own
hardware — RAM, SSD, localhost network, Redis (in Docker), and a simulated
cross-region hop — and lines them up against the classic figures.

---

## 0. Prerequisites

- Docker (for Redis).
- Python 3.9+ (`pip install redis==5.0.8`).

---

## 1. RAM vs SSD vs syscall overhead (pure Python)

Create `bench_local.py`:

```python
# bench_local.py
import os
import time
import tempfile

def bench(label, fn, iters):
    # warm up
    for _ in range(min(iters, 1000)):
        fn()
    t = time.perf_counter_ns()
    for _ in range(iters):
        fn()
    total_ns = time.perf_counter_ns() - t
    per = total_ns / iters
    unit = "ns"
    val = per
    if per >= 1_000_000:
        val, unit = per / 1_000_000, "ms"
    elif per >= 1000:
        val, unit = per / 1000, "µs"
    print(f"{label:<42} {val:8.2f} {unit}/op")

# --- RAM: read an element from an in-memory list ---
data = list(range(10_000_000))
i = 0
def ram_read():
    global i
    i = (i + 4096) % len(data)
    return data[i]
bench("RAM: random-ish list read", ram_read, 2_000_000)

# --- SSD: random 4 KB read from a file (bypass Python buffering) ---
tmp = tempfile.NamedTemporaryFile(delete=False)
tmp.write(os.urandom(64 * 1024 * 1024))   # 64 MB file
tmp.flush()
tmp.close()
fd = os.open(tmp.name, os.O_RDONLY)
size = 64 * 1024 * 1024
off = 0
def ssd_read():
    global off
    off = (off + 987_654) % (size - 4096)
    os.pread(fd, 4096, off)               # positional read, real syscall
bench("SSD: random 4KB pread (may be cached)", ssd_read, 200_000)

# Drop caches note: on Linux the OS page cache will serve repeat reads from RAM.
# To measure true SSD latency you'd need to drop caches (needs root):
#   sync; echo 3 > /proc/sys/vm/drop_caches
# The cached number still usefully shows syscall + page-cache cost.

os.close(fd)
os.unlink(tmp.name)

# --- syscall baseline: getpid() is a cheap kernel round trip ---
bench("syscall: os.getpid()", os.getpid, 1_000_000)
```

```bash
python bench_local.py
```

**What you should see (order of magnitude):** RAM read in the tens of
nanoseconds, `getpid` syscall in the hundreds of ns to low µs, and the "SSD"
read likely in the low µs *because the OS page cache serves it from RAM* — a
great teaching moment about why caching layers matter. Uncached SSD would be
~10–100 µs.

---

## 2. Localhost network round trip

Even a loopback TCP round trip costs far more than RAM. Create `bench_loopback.py`:

```python
# bench_loopback.py
import socket
import threading
import time

HOST, PORT = "127.0.0.1", 9099

def server():
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind((HOST, PORT))
    s.listen(1)
    conn, _ = s.accept()
    conn.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)  # no Nagle delay
    while True:
        data = conn.recv(16)
        if not data:
            break
        conn.sendall(data)   # echo

threading.Thread(target=server, daemon=True).start()
time.sleep(0.3)

c = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
c.connect((HOST, PORT))
c.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)

N = 100_000
t = time.perf_counter_ns()
for _ in range(N):
    c.sendall(b"ping")
    c.recv(16)
per_us = (time.perf_counter_ns() - t) / N / 1000
print(f"loopback TCP round trip: {per_us:.2f} µs/op")
c.close()
```

```bash
python bench_loopback.py
```

**What you should see:** single-digit to tens of µs per round trip — orders of
magnitude slower than a RAM read, and this is *loopback* (no real wire). It
frames why a same-datacenter round trip (~0.5 ms) is another ~50–100x on top.

---

## 3. Redis round trip (in-memory DB over the network)

```bash
docker run -d --name lat-redis -p 6379:6379 redis:7-alpine
```

Create `bench_redis.py`:

```python
# bench_redis.py
import time
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)
r.set("k", "v")

# Single round trips (each GET is one network round trip)
N = 20_000
t = time.perf_counter_ns()
for _ in range(N):
    r.get("k")
per_us = (time.perf_counter_ns() - t) / N / 1000
print(f"Redis GET (1 round trip):      {per_us:8.2f} µs/op")

# Pipelined: amortize the round trip across many commands
BATCH = 1000
t = time.perf_counter_ns()
for _ in range(N // BATCH):
    pipe = r.pipeline(transaction=False)
    for _ in range(BATCH):
        pipe.get("k")
    pipe.execute()
per_us_pipe = (time.perf_counter_ns() - t) / N / 1000
print(f"Redis GET (pipelined x{BATCH}):    {per_us_pipe:8.2f} µs/op")
print(f"-> pipelining is ~{per_us/per_us_pipe:.0f}x faster per op "
      f"(round trip amortized)")
```

```bash
python bench_redis.py
```

**What you should see:** a single `GET` in the tens-to-hundreds of µs (dominated
by the loopback round trip, not Redis's work), and the pipelined version 10–100x
faster *per op* because it amortizes the round trip. This is the central lesson:
**the round trip dominates, so batching/removing round trips is the big win.**

---

## 4. Simulated cross-region latency

Real cross-region round trips are ~50–150 ms (bounded ultimately by the speed of
light: ~5 µs/km one-way in fiber, so e.g. Amsterdam↔Virginia ~6,000 km ≈ 30 ms
one-way *minimum*, before routing/processing). Simulate it with a network delay
so you can feel it. On Linux:

```bash
# add 60 ms of latency to loopback (needs root; uses tc netem)
sudo tc qdisc add dev lo root netem delay 60ms
python bench_loopback.py     # now each "round trip" ~120 ms
sudo tc qdisc del dev lo root netem     # remove it when done
```

**What you should see:** the loopback round trip jumps to ~120 ms (2×60 ms).
Now imagine your request makes three serial cross-region calls — that's ~360 ms
gone before any real work. This is why you *parallelize* independent calls and
*colocate* chatty services.

---

## 5. Line them up

Collect your measured numbers and compare to the classic table:

| Layer                        | Classic figure  | Your measurement |
|------------------------------|-----------------|------------------|
| RAM read                     | ~100 ns         | ______           |
| Syscall (getpid)             | ~0.1–1 µs       | ______           |
| SSD random 4 KB (uncached)   | ~10–100 µs      | ______ (cached)  |
| Loopback TCP round trip      | ~5–30 µs        | ______           |
| Redis GET (1 round trip)     | ~0.05–0.5 ms    | ______           |
| Redis GET pipelined          | ~1–10 µs/op     | ______           |
| Cross-region round trip      | ~50–150 ms      | ______ (netem)   |

---

## 6. Cleanup

```bash
docker rm -f lat-redis
sudo tc qdisc del dev lo root netem 2>/dev/null || true
```

---

## What you proved

- The hierarchy spans ~7 orders of magnitude, from ns (RAM) to hundreds of ms
  (cross-region / cold start).
- Crossing a boundary (syscall → loopback → real network → cross-region) costs
  roughly 10–100x each step.
- The **round trip**, not the work, usually dominates — which is why
  pipelining, batching, caching, colocation, and parallelizing independent
  calls are the levers that actually move latency.

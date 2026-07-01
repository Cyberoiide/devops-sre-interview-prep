# Q208 — Hands-on lab: watch the complexity curves diverge

Big-O is abstract until you *time* it. This lab runs three algorithms — an O(n)
linear scan, an O(n log n) sort, and an O(n²) nested loop — across growing input
sizes on your own machine and prints the growth so you can see O(n²) fall off a
cliff while the others stay flat.

---

## 0. Prerequisites

- Python 3.9+ (standard library only — no pip installs).

---

## 1. The benchmark script

Create `bench_bigo.py`:

```python
# bench_bigo.py
import random
import time

def timed(fn, data):
    """Return milliseconds to run fn(data)."""
    t = time.perf_counter_ns()
    fn(data)
    return (time.perf_counter_ns() - t) / 1_000_000

# --- O(n): linear scan, find the max ---
def linear_scan(data):
    m = data[0]
    for x in data:            # one pass over n elements
        if x > m:
            m = x
    return m

# --- O(n log n): sort (Python's Timsort) ---
def sort_it(data):
    return sorted(data)       # O(n log n) comparison sort

# --- O(n^2): naive "is there a duplicate?" via nested loop ---
def has_dup_quadratic(data):
    n = len(data)
    for i in range(n):        # for each element...
        for j in range(i + 1, n):   # ...compare to every later element
            if data[i] == data[j]:
                return True
    return False

SIZES = [1_000, 2_000, 4_000, 8_000, 16_000, 32_000]
# Note: the O(n^2) loop is a pure-Python nested loop, so it dominates the
# runtime — n=32000 already takes ~10s. Add 64_000 if you want to feel the
# cliff (~40s), but you don't need it to see the ~4x-per-doubling pattern.

print(f"{'n':>8} | {'O(n) scan':>12} | {'O(n log n) sort':>16} | {'O(n^2) loop':>14}")
print("-" * 60)

prev = {}
for n in SIZES:
    # distinct values so the quadratic check does its full worst-case work
    data = random.sample(range(n * 10), n)

    t_lin  = timed(linear_scan, data)
    t_sort = timed(sort_it, data)
    t_quad = timed(has_dup_quadratic, data)

    print(f"{n:>8} | {t_lin:>10.3f}ms | {t_sort:>14.3f}ms | {t_quad:>12.3f}ms")

    # show the growth *ratio* vs the previous size (n doubled each step)
    if prev:
        print(f"{'  x factor':>8} | {t_lin/prev['lin']:>11.1f}x | "
              f"{t_sort/prev['sort']:>15.1f}x | {t_quad/prev['quad']:>13.1f}x")
    prev = {"lin": t_lin, "sort": t_sort, "quad": t_quad}
```

```bash
python bench_bigo.py
```

---

## 2. Read the growth *ratios* — this is the whole point

Each row **doubles n**. Watch how the time-per-step multiplies:

- **O(n) scan** — doubling n should roughly **double** the time (~2x per step).
- **O(n log n) sort** — a bit more than double (~2.1–2.3x): the `log n` factor
  grows slightly as n grows.
- **O(n²) loop** — doubling n **quadruples** the time (~4x per step). This is the
  tell: if you double the input and the work goes up ~4x, you have a quadratic.

The O(n²) column is already the slowest even at n=1,000 (pure-Python nested
loops are expensive per op), and by n=32,000 it's tens of *thousands* of times
slower than the scan. But the constant factor isn't the lesson — the **ratio**
is: watch the O(n²) `x factor` sit near 4x every time n doubles, while scan and
sort stay near 2x. That divergence *is* the difference between Big-O classes.

**Sample shape (your numbers will differ, the ratios won't):**

```
       n |    O(n) scan | O(n log n) sort |    O(n^2) loop
------------------------------------------------------------
    1000 |      0.018ms |         0.115ms |       11.181ms
    2000 |      0.015ms |         0.137ms |       38.693ms
  x factor |       0.8x |            1.2x |           3.5x
    4000 |      0.032ms |         0.287ms |      154.499ms
  x factor |       2.1x |            2.1x |           4.0x   <- quadratic quadruples
    ...
   32000 |      0.311ms |         3.068ms |    11329.750ms
```

---

## 3. Prove the O(n²) "cliff" a different way — worst case matters

The duplicate check above returns early if it finds a dup. Feed it distinct data
(as the script does) and it runs the *full* n² — the worst case. Now try the
opposite: put a duplicate at the very front and watch it return instantly.

Add this to the bottom of `bench_bigo.py` and re-run:

```python
print("\n--- worst case vs best case for the O(n^2) check (n=32000) ---")
worst = random.sample(range(320_000), 32_000)          # all distinct -> full n^2
best  = [7, 7] + random.sample(range(320_000), 31_998)  # dup up front -> O(1)
print(f"worst (all distinct): {timed(has_dup_quadratic, worst):.3f}ms")
print(f"best  (early dup):    {timed(has_dup_quadratic, best):.6f}ms")
```

The best case finishes in microseconds; the worst case takes the full quadratic
hit. **Big-O usually means worst case** — you design for the input that makes it
slow, because that's the input production will eventually send you.

---

## 4. The O(n log n) fix for the O(n²) problem

The naive duplicate check is O(n²). Sorting first (O(n log n)) then scanning
adjacent pairs (O(n)) is O(n log n) overall — and a hash set is O(n). Compare:

```python
def has_dup_sorted(data):        # O(n log n): sort, then one linear pass
    s = sorted(data)
    return any(s[i] == s[i+1] for i in range(len(s) - 1))

def has_dup_set(data):           # O(n): hash set membership
    seen = set()
    for x in data:
        if x in seen:
            return True
        seen.add(x)
    return False

big = random.sample(range(6_400_000), 640_000)   # 20x bigger than before
print("\n--- same job, three complexity classes (n=640000, all distinct) ---")
print(f"O(n log n) sort+scan: {timed(has_dup_sorted, big):.2f}ms")
print(f"O(n) hash set:        {timed(has_dup_set, big):.2f}ms")
# don't even try has_dup_quadratic at this size — it's ~400x the n=32000 run = minutes
```

```bash
python bench_bigo.py
```

**What you should see:** both the O(n log n) and O(n) versions finish in a couple
hundred milliseconds on 640,000 items, while the O(n²) version at the same size
would take *tens of minutes* (n² is ~400x the work of the n=32,000 run). Same
problem, and the right complexity class is the entire difference between
shippable and not.

---

## What you proved

- Timing across doubling inputs reveals the class directly: O(n) doubles, O(n²)
  quadruples, O(n log n) sits just above linear.
- Constant factors are invisible on the curve — the O(n²) loop starts *tied*
  with the others and still loses badly at scale.
- The practical lesson: when a loop scans a collection *inside* another loop over
  the same collection, you've written O(n²). Replacing it with a sort
  (O(n log n)) or a hash set (O(n)) is often a one-line change that turns a
  future outage into a non-event.

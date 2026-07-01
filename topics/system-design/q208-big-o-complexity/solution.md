# Q208 — Solution and walkthrough

## The one-paragraph answer

Big-O describes how an algorithm's cost **grows with input size `n`**, ignoring
constant factors and lower-order terms — it's about the shape of the curve, not
wall-clock time. O(n log n) is the class that sits between linear O(n) and
quadratic O(n²): the work grows a little faster than linearly because of a
`log n` factor. The canonical examples are the efficient comparison sorts —
**merge sort** and **heapsort** (O(n log n) worst case) and **quicksort**
(O(n log n) average) — and it's no accident they all land there: n log n is the
**proven lower bound for any comparison-based sort**, which is why every
standard-library sort (Go's `sort`, Python's Timsort, Java's) delivers it. As an
operator you care because the gap between O(n log n) and O(n²) is invisible on
test data and catastrophic at scale — at a million items it's a ~50,000x
difference in work, which is exactly the kind of thing that passes review and
then takes the service down.

---

## What Big-O actually measures

Big-O is an **asymptotic** bound: it tells you how cost behaves *as `n` gets
large*. Three rules follow:

1. **Drop constants.** `2n`, `500n`, and `0.5n` are all O(n). A constant factor
   shifts the line up or down but doesn't change its shape, and shape is what
   determines who wins at scale.
2. **Keep only the dominant term.** `n² + 1000n + 10⁶` is O(n²) — for large `n`
   the `n²` swamps everything else.
3. **Usually worst case.** Unless stated otherwise, O(f(n)) means the worst-case
   input. You design for the input that makes it slow, because production will
   eventually send it.

It's a *model*, not a stopwatch. An O(n) algorithm with a huge constant can be
slower than an O(n²) one for small `n` — but there's always a crossover point
past which the better class wins, and at production scale you're always past it.

---

## The classes, ordered, with real examples

Reason in these tiers (numbers assume n = 1,000,000):

- **O(1) — constant.** Cost independent of `n`. Hash-map lookup, array index by
  offset, Redis `GET`, pushing to a stack. ~1 operation.
- **O(log n) — logarithmic.** Cost grows by a constant each time `n` *doubles*.
  Binary search, a B-tree / balanced-BST index lookup. ~20 operations at 1M —
  this is why database indexes are magic: they turn a table scan into ~20 steps.
- **O(n) — linear.** Cost proportional to `n`. One loop over a list, `grep` over
  a file, summing an array. 1,000,000 operations.
- **O(n log n) — linearithmic.** Linear work repeated across `log n` levels. The
  efficient sorts; also many divide-and-conquer algorithms. ~20,000,000 ops.
- **O(n²) — quadratic.** Nested loop over the same data. Bubble/insertion sort,
  naive "compare every pair", naive dedup. 1,000,000,000,000 ops — a *million
  times* more than the linear version.
- **O(2ⁿ) — exponential.** Cost doubles with each added element. Brute-force
  subset enumeration, naive recursive Fibonacci. Intractable past small `n` —
  n=100 is more operations than there are atoms in the observable universe.

The two transitions worth burning in: **O(log n) vs O(n)** (the value of an
index) and **O(n log n) vs O(n²)** (the value of a decent algorithm).

---

## Why O(n log n) is the sorting floor

Merge sort makes the structure obvious: repeatedly split the array in half until
you have single elements — that's `log₂ n` levels of splitting — then merge back
up, and each level touches all `n` elements once. `log n` levels × `n` work per
level = **O(n log n)**, guaranteed worst case. Heapsort gets there differently:
`n` extract-max operations, each O(log n) to re-heapify.

And you *can't* do better with comparisons. There are `n!` possible orderings of
`n` items. Each comparison yields one bit of information (this < that, or not),
so to distinguish among `n!` outcomes you need at least `log₂(n!)` comparisons,
and by Stirling's approximation `log₂(n!) ≈ n log₂ n`. That's a hard lower bound:
**no comparison sort beats O(n log n)**.

You only go faster by *not comparing*. Counting sort and radix sort run in O(n)
(or O(n·k) for k-digit keys) because they bucket by value instead of comparing —
but they only work on bounded integer-like keys, not arbitrary comparables.
That's the trade: give up generality, buy linear time.

Quicksort is the practical favorite despite an O(n²) worst case (all elements to
one side of every pivot). Real implementations dodge it with randomized or
median-of-three pivots, making the pathological case astronomically unlikely.
Python's Timsort and Java's dual-pivot quicksort / Timsort are hybrids tuned for
real-world, partially-sorted data — all still O(n log n).

---

## Why an operator should care

You rarely write a sort. You constantly write — or inherit — code whose
complexity decides whether it scales:

- **The accidental O(n²).** Someone loops over a list and, inside, does a
  `list.contains()` or a linear search over the same list. Fine for the 50 test
  items; a nested million-element scan in production. The fix is usually a
  one-liner: a `set` (O(1) lookup) or a sort (O(n log n)) turns it into O(n) or
  O(n log n). The lab shows exactly this.
- **The missing index.** An unindexed `WHERE` or join is a table scan — O(n) per
  lookup, O(n²) if it's inside a loop of lookups. An index makes it O(log n).
  This is the single most common "why is the database on fire" cause.
- **Regex backtracking.** A poorly written regex can be O(2ⁿ) on adversarial
  input — the classic ReDoS outage.
- **Capacity planning.** Knowing the class tells you what happens when traffic
  10x's. Linear? 10x the cost. Quadratic? 100x. That's the difference between
  "add a node" and "the system is down."

The interview answer that lands: *"O(n log n) is the sort floor and what every
stdlib sort gives you; the thing I actually watch for in systems is code that's
secretly O(n²) — a scan inside a loop, or a query without an index — because it
looks fine in review and only shows up under load."*

---

## What NOT to do

- Don't confuse Big-O with speed. It predicts *scaling*, not the constant. Profile
  for the constant; use Big-O to reason about growth.
- Don't optimize complexity you don't need. For small, bounded `n`, an O(n²)
  loop can be simpler and perfectly fine — say so, and note the ceiling.
- Don't forget space complexity. Merge sort is O(n) *extra memory*; an algorithm
  can be time-optimal and still OOM. Time and space are separate budgets.
- Don't quote a best case as if it's the bound. "Quicksort is O(n log n)" is the
  average; its worst case is O(n²), which is why pivots get randomized.

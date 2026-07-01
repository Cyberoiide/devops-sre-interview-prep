# Q208 — Interviewer follow-ups

**Q: You keep saying "drop the constants." Doesn't the constant ever matter?**
It matters enormously in practice — it's just not what Big-O measures. Big-O
tells you *which algorithm wins as n grows*; the constant tells you *by how much*
and *where the crossover is*. For small, bounded n an O(n²) algorithm with a tiny
constant can beat an O(n log n) one with a big constant — which is exactly why
real sort implementations switch to insertion sort (O(n²) but tiny constant) for
small subarrays. So: Big-O to pick the class, profiling to tune the constant, and
know your n.

**Q: Prove to me that comparison sorting can't beat O(n log n).**
There are `n!` possible orderings of n distinct items. A comparison sort
distinguishes them using yes/no comparisons, and any sequence of comparisons is a
binary decision tree whose leaves are the possible orderings. A binary tree with
`n!` leaves has height at least `log₂(n!)`, and by Stirling `log₂(n!) ≈ n log₂ n`.
The height is the worst-case number of comparisons, so any comparison sort needs
at least ~n log n comparisons. You escape only by not comparing — counting/radix
sort bucket by value in O(n), at the cost of only working on bounded keys.

**Q: If counting/radix sort is O(n), why isn't it the default everywhere?**
Because it only works when keys are integers (or map cleanly to bounded integer
ranges) and the range k isn't enormous — radix is O(n·d) for d digits, and its
memory and constant factors are real. General-purpose sorts must handle arbitrary
comparable types (strings, tuples, custom comparators) where "bucket by value"
has no meaning. So stdlibs ship a comparison sort (O(n log n), fully general) and
leave radix/counting as a specialist tool you reach for on known integer data.

**Q: Quicksort is O(n²) worst case. Why is it still the go-to?**
Its *average* is O(n log n) with excellent constants and cache behavior — it sorts
in place with sequential memory access, which beats merge sort's extra-array
overhead in practice. The O(n²) worst case (a pivot that always splits 1 vs n-1)
is avoided by randomizing the pivot or using median-of-three, making the bad case
astronomically unlikely on real input. Where worst-case guarantees are mandatory
(e.g. adversarial input), you use heapsort or an introsort hybrid that detects bad
recursion depth and switches to heapsort.

**Q: How does this connect to a database index?**
A full table scan is O(n) per lookup; do that inside a loop of m lookups and
you're at O(n·m), effectively quadratic. A B-tree index turns each lookup into
O(log n) — ~20 hops at a million rows instead of a million. That's the same
O(log n) vs O(n) gap as binary search vs linear search, which is why "add an
index" is the most common performance fix in the world. The catch: indexes cost
O(log n) to maintain on every write and consume space, so you index read-hot
columns, not everything.

**Q: What's amortized complexity — is that cheating?**
No, it's averaging cost over a sequence of operations. A dynamic array's `append`
is O(1) *amortized*: most appends are O(1), but occasionally the array is full and
it reallocates and copies all n elements (O(n)). Because doublings get rarer as
the array grows, the *total* cost of n appends is O(n), so each is O(1) on
average. It's honest as long as you care about total throughput over many ops
rather than the worst-case latency of a single op — if you have a hard per-op
latency SLO, that occasional O(n) resize spike still matters.

**Q: Where does space complexity bite you even when time is fine?**
Merge sort is O(n log n) time but needs O(n) auxiliary memory for the merge — on
a memory-constrained box that's an OOM, not a slowdown. Loading a huge result set
into a list to dedup is O(n) space; on a big enough n you exhaust RAM before you
exhaust time. Recursion adds O(depth) stack — a naive recursion over a million
items can stack-overflow. Time and space are independent budgets; an algorithm
optimal in one can blow the other, so you state both.

**Q: You call something "O(2ⁿ)" — give me a real one and how you'd escape it.**
Naive recursive Fibonacci recomputes the same subproblems exponentially —
`fib(n)` calls `fib(n-1)` and `fib(n-2)`, branching to ~2ⁿ calls. Brute-force
"try every subset" (knapsack, set cover) is the same shape. You escape with
**memoization / dynamic programming** — cache subproblem results so each is
computed once, collapsing Fibonacci to O(n). When no such structure exists (truly
combinatorial problems), you fall back to approximation algorithms, heuristics, or
accept exponential only for tiny n.

**Q: Someone hands you code and asks "is this fast enough?" How do you answer
with Big-O?**
First find the dominant operation and how it scales with the real input size —
look for loops inside loops (O(n²)), scans that could be lookups (O(n) → O(1)),
and unindexed queries. Then plug in the *actual* n now and n after 10x growth: if
it's O(n log n) at a million and traffic 10x's, cost ~11x's — fine. If it's O(n²),
cost 100x's — plan the rewrite now. Big-O turns "feels slow" into "here's the
class, here's what happens at 10x, here's the one hot spot to fix."

# Q208 — Big-O complexity: explain O(n log n)

## The scenario (as an interviewer would pose it)

> "Walk me through Big-O notation. Explain what O(n log n) means and give me an
> algorithm that runs in it. Then tell me why I, as someone who runs services
> rather than writes sorting algorithms, should care.
>
> I'm not looking for a textbook recital. I want to see that you can place the
> common complexity classes in order, reason about what happens as the input
> grows from ten items to ten million, and connect that to something real —
> a query, a log-processing job, a config parser that someone dropped an O(n²)
> loop into and now falls over at scale."

## What the interviewer is really probing

1. Do you understand that **Big-O describes growth, not wall-clock time** — how
   the cost scales with input size `n`, ignoring constants and lower-order terms?
2. Can you **order the common classes** — O(1) < O(log n) < O(n) < O(n log n) <
   O(n²) < O(2ⁿ) — and give a real example of each?
3. Do you know *why* O(n log n) is special: it's the proven **lower bound for
   comparison-based sorting**, and it's what every standard-library sort
   (Go's `sort`, Python's Timsort, Java's) actually delivers?
4. Can you connect it to operations you run — an unindexed join, a nested loop
   over a growing list, a regex that backtracks — where the difference between
   O(n log n) and O(n²) is the difference between "fine" and "3am page"?

## The mental model that matters more than the definition

Big-O answers one question: **as `n` grows, how does the work grow?**

- Constants don't matter. `2n` and `500n` are both O(n) — because for large
  enough `n`, an O(n log n) algorithm beats an O(n²) one *regardless* of the
  constant factors. The curve wins, not the coefficient.
- You keep only the dominant term. `n² + 3n + 100` is O(n²); the `3n` and `100`
  vanish as `n` grows.
- The point is **scale**. At n=10 the difference is invisible. At n=1,000,000 an
  O(n²) algorithm does a *trillion* operations while O(n log n) does ~20 million
  — a ~50,000x gap. That gap is what takes a service down.

## The common classes, ordered

Reason in these tiers. Numbers assume n = 1,000,000.

| Class        | Name         | ~Ops at n=1M   | Real example                                        |
|--------------|--------------|----------------|-----------------------------------------------------|
| O(1)         | Constant     | 1              | Hash-map lookup, array index, Redis `GET`           |
| O(log n)     | Logarithmic  | ~20            | Binary search, balanced-tree / B-tree index lookup  |
| O(n)         | Linear       | 1,000,000      | Scan a list, single loop, `grep` over a file        |
| O(n log n)   | Linearithmic | ~20,000,000    | Merge sort, heapsort, quicksort (avg), any good sort |
| O(n²)        | Quadratic    | 1,000,000,000,000 | Nested loop, bubble sort, naive dedup / pair check |
| O(2ⁿ)        | Exponential  | absurd (>10³⁰⁰⁰⁰⁰) | Brute-force subsets, naive recursive Fibonacci  |

The jump from O(n log n) to O(n²) is the one that bites in practice: both look
fine on a laptop with test data, and only one survives production volume.

## Why O(n log n) specifically

It sits between linear and quadratic, and it's the home of the efficient
comparison sorts:

- **Merge sort** — split in half (log n levels of splitting), merge each level in
  O(n) work → O(n log n), guaranteed worst case.
- **Heapsort** — build a heap, extract-max n times, each extraction O(log n) →
  O(n log n) worst case.
- **Quicksort** — O(n log n) *average*, but O(n²) worst case on bad pivots (why
  real implementations randomize or median-of-three the pivot).

It's provably optimal: any sort that works by **comparing elements** must do at
least ~n log n comparisons (there are n! possible orderings, and each comparison
gives one bit, so you need log₂(n!) ≈ n log n comparisons). You only beat it by
*not comparing* — counting/radix sort on bounded integers, which is a different
game. That's why every general-purpose standard-library sort is O(n log n).

## Deliverables

- `exercise.md` — a runnable Python script that **times** a linear scan, an
  n log n sort, and an n² nested loop across growing input sizes, and prints the
  growth curves so you *see* the divergence.
- `solution.md` — the full written answer, the classes, and how to reason about
  them as an operator.
- `deeper.md` — the follow-up questions an interviewer asks next.

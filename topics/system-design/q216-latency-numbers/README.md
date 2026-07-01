# Q216 — Latency numbers every engineer should know

## The scenario (as an interviewer would pose it)

> "Give me a feel for the relative speed of the things a system touches. How
> long does it take to read from RAM versus a local SSD versus another machine
> in the same availability zone versus another region? Where does an in-memory
> database query land? What about a serverless cold start?
>
> I don't want you to recite numbers to the nanosecond — I want to see that you
> can *reason* with them. If a user expects a page in under a second, and your
> request fans out to a cache, a database, and a downstream service, can the
> budget close? Where's the bottleneck, and what would you cut first?"

## What the interviewer is really probing

1. Do you have the **orders of magnitude** internalized — nanoseconds (RAM),
   microseconds (SSD, intra-datacenter network), milliseconds (cross-region,
   disk seek, in-memory DB round trip), hundreds of milliseconds (cold starts,
   cross-continent round trips)?
2. Can you reason about a **latency budget**: user expectation minus fixed costs
   = what you have left, and does the design fit?
3. Do you know the *shape* of the classic "Latency Numbers Every Programmer
   Should Know" table (Jeff Dean / Peter Norvig lineage) — not to memorize, but
   to reason with?
4. Do you understand what dominates: **network round trips and serialization
   across boundaries**, not CPU, in most distributed systems?

## The mental model that matters more than the numbers

Memorizing the table is a party trick; the useful skill is **budget reasoning**:

- Users perceive < 100 ms as instant, tolerate up to ~1 s before losing focus,
  and start abandoning past a few seconds. SaaS targets a 0.5–1 s response.
- Your response time is the **sum of the critical path**: every serial hop
  (cache lookup, DB query, downstream call, serialization, network) adds up.
- The wins come from **removing round trips** (cache, batch, colocate,
  parallelize independent calls) far more than from shaving CPU. A single
  cross-region round trip (~tens of ms) can cost more than thousands of RAM
  accesses.

## The classic table (approximate, order-of-magnitude)

These are the "numbers every engineer should know", rounded to teach scale.
Exact values vary by hardware and year — the **ratios** are the point.

| Operation                                   | Latency        | In "human" terms (×1e9) |
|---------------------------------------------|----------------|-------------------------|
| L1 cache reference                          | ~1 ns          | 1 second                |
| Branch mispredict                           | ~3 ns          | 3 seconds               |
| L2 cache reference                          | ~4 ns          | 4 seconds               |
| Mutex lock/unlock                           | ~17 ns         | 17 seconds              |
| Main memory (RAM) reference                 | ~100 ns        | ~1.5 minutes            |
| Compress 1 KB with a fast compressor        | ~2 µs          | ~30 minutes             |
| Read 1 MB sequentially from RAM             | ~3 µs          | ~50 minutes             |
| Read 4 KB randomly from NVMe SSD            | ~10–20 µs      | ~4 hours                |
| Read 1 MB sequentially from NVMe SSD        | ~50 µs         | ~14 hours               |
| Round trip within same datacenter / AZ      | ~0.5 ms        | ~6 days                 |
| Read 1 MB sequentially from SSD (older SATA)| ~1 ms          | ~11.5 days              |
| Disk seek (spinning HDD)                    | ~10 ms         | ~4 months               |
| Round trip cross-AZ, same region            | ~1–2 ms        | ~2–3 weeks              |
| In-memory DB query round trip (e.g. Redis)  | ~1–5 ms        | up to ~2 months         |
| Round trip cross-region (e.g. NL↔US)        | ~50–150 ms     | ~3–14 years             |
| Lambda / serverless cold start              | ~200–500 ms    | ~decades                |

The right-hand column scales every latency by 1e9 (a nanosecond becomes a
second) so the ratios become human-graspable: RAM is "a minute", a
same-DC round trip is "a week", a cross-region hop is "years".

## Deliverables

- `exercise.md` — measure these on *your own machine*: RAM vs. disk vs.
  localhost network vs. Redis (in Docker) vs. a simulated cross-region hop, and
  compare against the table.
- `solution.md` — the numbers, the reasoning framework, and worked budget
  examples.
- `deeper.md` — interviewer follow-ups with concise answers.

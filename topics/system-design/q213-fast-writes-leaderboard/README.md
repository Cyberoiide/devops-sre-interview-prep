# Q213 — Fast writes: a real-time game leaderboard

## The scenario (as an interviewer would pose it)

> "You're building the backend for a popular online game. There are millions of
> players. Scores change constantly — a busy match generates thousands of score
> updates per second. Players expect to open the leaderboard and *instantly*
> see the global top 100, and also see *their own rank* ('you are #4,215,003
> of 8,000,000') with no perceptible delay.
>
> Design the data store and access patterns for this leaderboard. Assume the
> write rate is very high and rank queries must be effectively instant. How do
> you store scores, update them, and answer 'what is player X's rank?' and
> 'give me the top N'? Then tell me how this scales and what breaks."

## What the interviewer is really probing

1. Do you recognize that **the hard part is rank, not storage**? Storing scores
   is trivial; computing rank on demand over millions of rows is what kills a
   naive design.
2. Do you know that a relational `SELECT COUNT(*) WHERE score > X` or `ORDER BY
   score LIMIT N` is `O(N)` per query and won't survive this load?
3. Do you know the idiomatic answer — **Redis Sorted Sets (ZSET)** — and *why*
   it fits: it keeps members ordered by score at all times, so rank is
   maintained incrementally at write time and read in `O(log N)`?
4. Can you go beyond the happy path: ties, persistence, sharding, memory,
   time-windowed leaderboards, and hot-key concerns?

## Why the naive answer fails

A SQL table `players(id, score)`:

- **Top N**: `ORDER BY score DESC LIMIT 100` needs an index scan; feasible with
  an index but the index must be maintained under a very high write rate, and
  every update reshuffles ordering.
- **A player's rank**: `SELECT COUNT(*) FROM players WHERE score > :x` — this is
  `O(N)`, a full scan of everyone above them, run for *every* rank lookup. With
  millions of players and many lookups per second this is a non-starter.

The insight: you want a structure that **maintains sorted order incrementally as
writes happen**, so both "top N" and "rank of X" are cheap reads. That is
exactly a sorted set.

## Deliverables

- `exercise.md` — build a working leaderboard on Redis in Docker, load ~1M
  entries, and query ranks and top-N with real commands and timings.
- `solution.md` — the full written design, complexity analysis, and scaling
  discussion.
- `deeper.md` — interviewer follow-ups (ties, persistence, sharding,
  time-windows, hot keys) with concise answers.

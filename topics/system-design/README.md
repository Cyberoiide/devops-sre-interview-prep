# System Design — Interview Prep

Hands-on study material for DevOps / SRE and backend system-design interviews.
Each question has its own folder with four files:

- **README.md** — the scenario, stated the way an interviewer poses it.
- **exercise.md** — a runnable lab (Docker + Redis + small scripts) you can
  actually execute to *see* the concept.
- **solution.md** — the full written answer and walkthrough.
- **deeper.md** — the follow-up questions an interviewer asks next, with concise
  answers.

## Questions

- [q212 — The thundering herd](./q212-thundering-herd/) — a recovering backend
  gets slammed by every queued/retrying request at once; tame it with backoff +
  jitter and request collapsing / singleflight.
- [q213 — Fast writes: game leaderboard](./q213-fast-writes-leaderboard/) —
  millions of real-time score updates with instant rank retrieval, built on
  Redis Sorted Sets (`O(log N)` writes, ranks precomputed at write time).
- [q214 — Matchmaking race condition](./q214-matchmaking-race-condition/) — the
  same player gets matched into two games at once; fix the check-then-act race
  with atomic `SETNX` / Lua claims and prove it with metrics.
- [q216 — Latency numbers every engineer should know](./q216-latency-numbers/) —
  the ns → µs → ms → hundreds-of-ms hierarchy, and how to reason about latency
  budgets instead of memorizing figures.
- [q220 — Redis vs Memcached](./q220-redis-vs-memcached/) — the differences that
  matter (durability, data model, threading) and the specific workload where
  Memcached is actually the better pick.

## Running the labs

All exercises are self-contained and use Docker. General prerequisites:

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install redis==5.0.8 pymemcache==4.0.0   # pymemcache only needed for q220
```

Each `exercise.md` starts the containers it needs and ends with a cleanup step
(`docker rm -f ...`). Nothing requires a cloud account.

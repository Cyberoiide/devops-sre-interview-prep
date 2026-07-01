# Q214 — The matchmaking race condition

## The scenario (as an interviewer would pose it)

> "You run the matchmaking service for an online game. Players enter a lobby;
> the matchmaker pairs waiting players by their parameters (skill rating, region,
> game mode) and forms matches. Under normal load it's fine.
>
> During a traffic spike you start getting bug reports: the *same player* is
> being placed into **two different matches at the same time**. They load into
> one game, then get yanked into another, or two games both think they own the
> player. Sometimes a match forms with a player who's already busy.
>
> This is a classic **race condition**. Two matchmaker workers read the lobby at
> nearly the same instant, both see player P as available, and both grab P.
> Walk me through why this happens and how you'd fix it — and I want to hear
> more than 'add a lock'. Show me the progression from a naive fix to something
> that actually holds up in a distributed lobby, and tell me how you'd *prove*
> in production that the race is gone."

## What the interviewer is really probing

1. Do you understand the race precisely — a **check-then-act** (TOCTOU:
   time-of-check to time-of-use) gap where two workers both pass the "is P
   available?" check before either acts?
2. Can you walk the ladder of solutions and say when each is appropriate:
   - **App-level lock** — a mutex inside one process. Fine for a single
     matchmaker; useless the moment you run more than one.
   - **Distributed lock (Redlock)** — coordinate mutual exclusion across many
     matchmaker instances.
   - **Atomic claim with `SETNX` / atomic Lua** — often the *better* answer:
     make "add P to a match" an atomic conditional operation so P is added only
     if not already claimed. No separate lock to manage.
3. Do you know the correctness details: **atomicity**, lock **TTL**, **fencing
   tokens**, and why a naive `GET`-then-`SET` is broken?
4. **Observability**: how do you *know* it's fixed? Emit a metric on every
   claim-denied / lock-contention event; graph it in Grafana. A near-zero rate
   of "double-claim prevented" spikes during load means the guard is doing its
   job (and a *rising* count of prevented races tells you contention is real and
   being handled, not silently corrupting state).

## The bug, precisely

```
Worker A                         Worker B
--------                         --------
GET player:P:status  -> "waiting"
                                 GET player:P:status  -> "waiting"   <-- both see "waiting"
(decides: claim P)               (decides: claim P)
SET player:P:status "matched"    SET player:P:status "matched"
add P to match #1                add P to match #2                   <-- P is in TWO matches
```

The window between the read ("check") and the write ("act") is the race. Any
fix must make "if available, then claim" a **single atomic step** that only one
worker can win.

## Deliverables

- `exercise.md` — reproduce the double-match race with concurrent clients in
  Docker + Redis, then fix it with `SETNX` and with an atomic Lua script, and
  add a metric that counts prevented races.
- `solution.md` — the full written answer, the ladder of fixes, and the
  observability story.
- `deeper.md` — interviewer follow-ups (atomicity, TTL, fencing, Redlock
  critique) with concise answers.

# Databases

Interview-prep study material for DevOps/SRE roles, focused on the database
knowledge you're expected to have as the person who reviews the migrations and
gets paged when a query melts the database — **breadth, not DBA depth.** Every
exercise runs locally: **Postgres in Docker**, or plain **SQLite** if you have
no Docker, so you can practice on a laptop with no cloud account.

> Where an exercise shows Postgres syntax, the SQLite differences are called out
> inline. The concepts are identical; only a few DDL keywords change.

## Questions

| ID | Topic | One-line summary |
|----|-------|------------------|
| [q204](./q204-schema-design/) | Schema design | The top considerations when designing a schema — normalization vs JOIN cost, data types, foreign keys, constraints, indexing, and when to denormalize. |

## Folder layout

Each question folder contains exactly four files:

- **README.md** — the scenario, phrased the way an interviewer would pose it.
- **exercise.md** — a hands-on lab with concrete, runnable SQL.
- **solution.md** — the detailed written answer and walkthrough.
- **deeper.md** — likely interviewer follow-up questions with concise answers.

## How to use this repo

1. Read the question's `README.md` and try to answer it out loud (whiteboard style).
2. Do the lab in `exercise.md` — actually run the SQL, don't just read it. Feeling
   the database reject an orphan insert sticks; reading about it doesn't.
3. Compare your reasoning against `solution.md`.
4. Drill the follow-ups in `deeper.md` until the answers are reflexive.

## Prerequisites

- **Docker** (for the Postgres path) — `postgres:16-alpine` is pulled on first run.
- **or** `sqlite3` >= 3.x (for the no-Docker path) — ships on most systems.
- A shell (`bash`/`zsh`). No cloud account, no credentials.

Each exercise lists exactly what it needs and ends with a cleanup step.

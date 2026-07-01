# Q204 — What are the top considerations when you design a database schema?

## The scenario (as an interviewer would pose it)

> "You're not the DBA on this team, but you're the DevOps/SRE — you review the
> migrations, you get paged when a query melts the database at 2am, and you're
> the one who has to explain to the incident channel why a `total` column
> overflowed on Black Friday. So tell me: when someone hands you a new schema,
> or asks you to sanity-check one, what are the things you actually look at?
> Walk me through the top considerations for schema design, and be honest about
> where a DBA goes deeper than you need to."

## What the interviewer is really probing

This question is a *breadth* check, not a DBA-depth check. They want to know:

1. Do you understand **normalization** well enough to spot both redundancy
   *and* the over-normalized JOIN-monster that tanks read latency — and can you
   articulate the trade-off rather than reciting "3NF good"?
2. Do you know that **data types are a durability and correctness decision**,
   not a formatting detail — `INT` vs `BIGINT`, `VARCHAR` vs `TEXT`, `NUMERIC`
   vs `float` — and that the wrong choice is a future outage?
3. Do you understand **referential integrity** — that foreign keys are what
   stop the database from slowly filling with orphaned, un-joinable garbage?
4. Can you round it out: **primary keys, indexing (and its write cost),
   constraints (`NOT NULL`, `UNIQUE`, `CHECK`), when denormalization is
   actually justified, and where NoSQL/schema-on-read changes the picture?**

The honest framing they're listening for: *you know enough to catch the
dangerous mistakes in review and reason about them in an incident, and you know
when to pull in a real DBA.* That's the DevOps/SRE job here, not designing a
1000-table OLTP system from scratch.

## The short version of the answer

- **Normalize to kill redundancy, then stop.** Redundant data drifts out of
  sync and corrupts silently; but each normal form you add is another JOIN at
  read time. Normalize the write path; **denormalize deliberately** for
  read-heavy paths where the JOIN cost hurts and you can keep copies consistent.
- **Pick data types like they're permanent, because they effectively are.**
  Right integer width (will this ID exceed 2.1 billion?), `VARCHAR(n)`/`TEXT`
  over `CHAR(n)` for variable data, `NUMERIC` for money (never `float`),
  timezone-aware timestamps. Wrong types either waste space at scale or fail to
  fit real values later — and changing a type on a huge live table is a painful
  migration.
- **Foreign keys enforce referential integrity.** Without them the app is the
  *only* thing stopping orphaned rows, and the app has bugs. FKs let the
  database reject bad data structurally.
- **Then the supporting cast:** a real primary key, indexes on what you
  actually query (and awareness that every index taxes every write), and
  `NOT NULL` / `UNIQUE` / `CHECK` constraints to encode invariants in the schema
  instead of hoping the app enforces them.
- **Know the escape hatch:** NoSQL / schema-on-read trades these guarantees for
  flexibility and horizontal scale — a valid choice for some workloads, but you
  give up exactly the integrity the relational model hands you for free.

## Deliverables

- `exercise.md` — a hands-on lab (Postgres in Docker, or SQLite if you have no
  Docker). You'll *feel* an over-normalized 5-way JOIN, watch the database
  reject an orphan insert once a foreign key exists (and silently accept
  corruption when it doesn't), overflow a too-small integer column, and fix all
  three.
- `solution.md` — the full written answer and the review checklist.
- `deeper.md` — the follow-up questions an interviewer asks next, with concise
  answers.

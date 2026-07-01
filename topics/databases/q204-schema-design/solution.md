# Q204 — Solution and walkthrough

## The one-paragraph answer

Good schema design is a small set of trade-offs held in tension. **Normalize**
to eliminate redundant data that would otherwise drift out of sync — but stop
before you've shredded every concept into its own table, because each normal
form is another JOIN on the read path; denormalize *deliberately* where reads
hurt and you can keep the copies consistent. **Choose data types as permanent
decisions**: correct integer width (`bigint` for anything that grows),
`numeric` for money, `text`/`varchar` over `CHAR(n)` for variable strings,
timezone-aware timestamps — wrong types either waste space at scale or fail to
fit real values later, and altering a column type on a huge live table is a
painful, locking migration. **Declare foreign keys** so the database itself
rejects orphaned rows instead of trusting bug-free application code. Round it
out with a real **primary key**, **indexes** on what you actually query (paid
for on every write), and **`NOT NULL`/`UNIQUE`/`CHECK` constraints** that encode
invariants in the schema. And know the contrast: **NoSQL / schema-on-read**
trades all of this for flexibility and horizontal scale — sometimes the right
call, but you're giving up exactly the guarantees the relational model hands
you for free.

---

## The three that matter most

### 1. Normalization — kill redundancy, but don't over-do it

**What it buys you.** Normalization means storing each fact once. If a
customer's country lives in one place, you change it in one place. Store it
redundantly across a million order rows and an update that misses some rows
leaves the data *inconsistent* — the classic update anomaly. Third normal form
(3NF) is the usual target for a transactional (OLTP) schema: every non-key
column depends on the key, the whole key, and nothing but the key.

**Where it goes wrong.** Each table you split off is a JOIN you pay on every
read. The lab's country → city → address → customer → order chain turns *"list
orders in France"* into a five-table JOIN. Country and city almost never change
independently for a given address, so splitting them bought a consistency
guarantee you'll essentially never exercise — and cost a JOIN on every single
read, plus a much larger search space for the query planner.

**The balance.** Normalize the **write path** (where anomalies actually bite),
and **denormalize the read path deliberately** when JOIN cost hurts and you can
keep the duplicated data consistent (via the app, triggers, or a materialized
view). Denormalization is a considered optimization with a maintenance cost,
not a default and not laziness. See the "when denormalization is justified"
section below.

### 2. Data types — pick them like they're permanent

They effectively are: changing a column type on a large, live table is a
rewrite that locks and needs a maintenance window. Get them right up front.

- **Integer width.** `int`/`int4` caps at ~2.1 billion. Any identifier or
  counter that could grow past that must be `bigint`/`int8` from day one. The
  lab overflows a `serial` (32-bit) sequence at 2,147,483,647 and the table can
  take no more rows — a real, repeated production outage. Default identifiers to
  `bigint`.
- **Exact vs approximate numbers.** Money and anything needing exact arithmetic
  is `numeric`/`decimal`. `float`/`double` is binary floating point:
  `0.1 + 0.2 = 0.30000000000000004`. Never store currency in a float.
- **Strings.** `text` or `varchar(n)` for variable-length data. `CHAR(n)` is
  fixed-length and **space-pads** to `n` (`'ab'::char(5)` is `'ab   '`), which
  surprises people on comparisons and concatenation; reserve it for genuinely
  fixed-width codes. In Postgres `text` and `varchar` are the same speed —
  `varchar(n)` just adds a length check; use `n` only when there's a real domain
  limit.
- **Time.** Use timezone-aware timestamps (`timestamptz` in Postgres). Naive
  local timestamps are a bug waiting for the next DST change or region.
- **Booleans, enums, UUIDs, JSON.** Use the real type (`boolean`, native `enum`
  or a lookup table, `uuid`, `jsonb`) instead of stuffing everything into
  strings — the database can then validate and index it properly.

The mental test: *"what's the largest / most precise value this could ever
hold, and does it need exact arithmetic?"* Answer for the lifetime of the
table, not today's data.

### 3. Foreign keys — referential integrity

A foreign key says "this column must point at a row that exists in that table."
It's what stops the slow accumulation of **orphaned rows** — an `order` whose
`customer_id` matches no customer, a `line_item` for a deleted order. Without
the FK, the *application* is the only guardian of integrity, and applications
have bugs, races, forgotten code paths, ad-hoc fix-up scripts, and that one
legacy service nobody owns. The lab shows a no-FK table happily accepting an
order for customer `999` that doesn't exist; add the FK and the same insert is
rejected at the database level.

FKs also define **delete behavior**: `ON DELETE RESTRICT` (default — block
deleting a customer who has orders), `ON DELETE CASCADE` (delete their orders
too), or `ON DELETE SET NULL`. Choosing this consciously prevents both
accidental orphaning and accidental mass deletion.

The one honest caveat: FKs add a small write-time check and can complicate bulk
loads and sharding (cross-shard FKs aren't enforceable). Those are reasons to
manage them carefully at extreme scale, not reasons to skip them on a normal
OLTP database.

---

## The supporting cast (mention these to show breadth)

### Primary keys

Every table needs one — a column (or set) that uniquely identifies a row and is
the target for foreign keys. Prefer a **surrogate key** (an auto-generated
`bigint` or `uuid`) over a **natural key** (email, SSN) because natural keys
change and leak business meaning into references. `uuid` is handy when IDs are
generated client-side or across shards (no central sequence), at the cost of
larger, less cache-friendly indexes than a sequential `bigint`.

### Indexing — and its cost on writes

Indexes turn a sequential scan (read every row) into an index scan (jump
straight to matching rows) — the lab shows the plan flip after
`CREATE INDEX`. Index the columns you actually **filter, join, or sort on**.
But every index is a copy of that data the engine must **update on every
`INSERT`/`UPDATE`/`DELETE`**, plus disk and memory. So:

- Don't index everything "just in case" — unused indexes are pure write tax.
- Composite index column order matters (leftmost-prefix rule).
- Foreign key columns are usually worth indexing (JOINs and cascade checks use
  them).
- Watch write-heavy tables: more indexes = slower writes and more
  write amplification.

### Constraints — `NOT NULL`, `UNIQUE`, `CHECK`

These encode invariants **in the schema** so they hold for every writer, not
just the app:

- `NOT NULL` — the column is required. Cheap, and prevents a whole class of
  "why is this null" bugs downstream.
- `UNIQUE` — no duplicates (one email per account). Backed by an index, so it
  also speeds lookups.
- `CHECK` — an arbitrary boolean rule (`balance >= 0`, `status IN (...)`,
  `end_date >= start_date`). Business rules that live *only* in application code
  are enforced only until the next code path forgets them.

### When denormalization is justified

Denormalize on purpose when:

- **Read-heavy / analytics / reporting** workloads where JOIN cost dominates and
  data changes rarely — star schemas in a warehouse are deliberately
  denormalized for exactly this.
- A **hot read path** needs to avoid a specific expensive JOIN (e.g. store a
  cached `order_count` on `customer` rather than counting orders every read).
- You can keep the duplicated data consistent — via application logic,
  triggers, or a **materialized view** (which is denormalization the database
  maintains for you).

The trade you're accepting: duplicated data can drift, and writes now touch more
places. That's fine when reads vastly outnumber writes and you've measured the
JOIN as the bottleneck. It's not fine as a reflexive default.

### The NoSQL / schema-on-read contrast

Relational databases are **schema-on-write**: you define the shape up front and
the database enforces types, constraints, and referential integrity on every
write. NoSQL stores (document DBs like MongoDB, wide-column like Cassandra,
key-value like DynamoDB) are typically **schema-on-read**: you store flexible
documents and the *application* interprets structure at read time. That buys
flexibility (evolve the shape without migrations) and, for some designs, easier
horizontal scaling — but you generally **give up** foreign keys, multi-table
JOINs, and often multi-row ACID transactions. You end up denormalizing by
necessity (embedding related data in one document) and owning integrity in
application code. It's a legitimate choice for the right workload — huge scale,
flexible/evolving schema, access patterns known in advance — but it's a
*trade*, not a free upgrade. The relational model hands you integrity for free;
NoSQL asks you to earn it back in code.

---

## The DevOps/SRE-vs-DBA honesty

Be upfront about the framing. As the DevOps/SRE you should be able to:

- Read a migration and catch the dangerous mistakes — a missing FK, a 32-bit ID
  on a table that'll grow, money in a float, a missing `NOT NULL`, an index
  added to a hot write path without thought.
- Reason about a slow query in an incident (`EXPLAIN`, is it scanning? is a
  JOIN exploding? is there an index?).
- Know when denormalization is a fix and when it's the cause.

Where the DBA goes deeper and you hand off: advanced index strategy (partial,
covering, GIN/GiST), partitioning schemes, query-planner tuning and statistics,
storage/vacuum/bloat internals, replication topology, and capacity planning.
Knowing *where that line is* is itself the senior answer.

---

## The 60-second review checklist

When a schema/migration lands in review, scan for:

1. **Primary key** on every table (surrogate `bigint`/`uuid`)?
2. **Foreign keys** on every relationship, with a deliberate `ON DELETE`?
3. **Integer widths** — `bigint` for anything that grows? Money in `numeric`,
   not `float`?
4. **`NOT NULL`** on everything that's logically required? `UNIQUE` on natural
   keys like email? `CHECK` for value ranges/enums?
5. **Indexes** on the columns actually queried/joined — and *not* a pile of
   speculative ones taxing writes?
6. **Normalization** sane — no obvious redundancy, but not shredded into a
   JOIN-monster either?
7. **Timestamps** timezone-aware?

If those seven are clean, the schema is in good shape for review; the rest is
DBA depth.

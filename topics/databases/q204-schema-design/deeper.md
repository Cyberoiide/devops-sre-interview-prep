# Q204 — Interviewer follow-ups

**Q: What actually is 1NF / 2NF / 3NF — can you define them without hand-waving?**
1NF: every column holds a single atomic value (no arrays/CSV stuffed in a
column, no repeating groups). 2NF: 1NF plus every non-key column depends on the
*whole* primary key (only matters for composite keys — kills partial
dependencies). 3NF: 2NF plus no non-key column depends on another non-key
column (no transitive dependencies — e.g. don't store `zip` *and* the `city`
that `zip` determines in the same table). The one-liner people quote: every
non-key attribute depends on "the key, the whole key, and nothing but the key."
3NF is the normal target for OLTP; BCNF is a slightly stricter variant that
rarely comes up in practice.

**Q: You keep saying "don't over-normalize" — give me the rule for when to
denormalize.**
Denormalize when reads dominate, you've *measured* a JOIN as the bottleneck, and
the duplicated data changes rarely enough that you can keep it consistent
(app logic, triggers, or a materialized view). Analytics/reporting (star
schemas) and hot read paths (a cached counter on a parent row) are the classic
cases. Don't denormalize reflexively — the cost is drift risk and writes that
now touch more places. Normalize the write path, denormalize the read path on
purpose.

**Q: `VARCHAR(255)` vs `TEXT` in Postgres — does it matter?**
Performance-wise, no — Postgres stores them identically and `text` has no length
limit; `varchar(n)` just adds a length check. Use `varchar(n)` only when `n` is
a *real* domain constraint (a country code is 2 chars). The `VARCHAR(255)`
default is a MySQL-era habit (255 was the largest length storable in one byte);
it's cargo-culted into Postgres where it means nothing. In MySQL the choice has
more nuance (row format, index prefix lengths), so know your engine.

**Q: `CHAR(n)` vs `VARCHAR(n)` — is there ever a reason to use `CHAR`?**
Rarely. `CHAR(n)` is fixed-length and space-pads shorter values to `n`, which
bites you on comparisons and concatenation. It's only worth it for values that
are genuinely always exactly `n` characters (a fixed 3-letter currency code,
say) where the padding never happens — and even then most people just use
`varchar`/`text`. Modern engines don't give `CHAR` a meaningful storage or speed
edge.

**Q: Foreign keys have a cost. When would you deliberately not use them?**
At extreme scale or in sharded/distributed setups where a referenced row lives
on another shard — the FK simply can't be enforced cross-shard, so integrity
moves to the application. Very high-throughput bulk-load pipelines sometimes
drop/defer FKs during load and re-validate after. And some NoSQL stores don't
offer them at all. Those are conscious trades at the edges; on a normal
single-node OLTP database, use FKs — the integrity is worth the tiny write-time
check.

**Q: `ON DELETE CASCADE` — love it or fear it?**
Respect it. It's correct for true ownership (delete an order → delete its line
items, which are meaningless alone). It's dangerous when the "child" has
independent value or when a delete could cascade far wider than the person
running it expects — one `DELETE` quietly wiping thousands of rows. Default to
`RESTRICT` (block the delete) and reach for `CASCADE` only where the child
genuinely cannot outlive the parent. `SET NULL` fits optional relationships.

**Q: Surrogate key (auto-int/UUID) or natural key (email, SSN)?**
Surrogate, almost always. Natural keys change (people change email, SSN formats
vary, "unique" business values turn out not to be) and every foreign key
referencing them then has to change too, and they leak PII/business meaning into
every reference. Use a surrogate `bigint`/`uuid` as the PK and put a `UNIQUE`
constraint on the natural key to still enforce its uniqueness. Best of both.

**Q: `bigint` vs `uuid` for primary keys?**
`bigint` (sequential): compact, cache-friendly, fast index inserts (appends to
the end), but requires a central sequence and leaks row counts/ordering. `uuid`:
generatable anywhere (client-side, across shards, no coordination) and
non-guessable, but larger (16 bytes) and random UUIDv4 scatters index inserts
causing page splits/fragmentation — `uuidv7` (time-ordered) fixes much of that.
Pick `bigint` for a single-node app, `uuid` (ideally v7) when you need
distributed/offline ID generation.

**Q: How do indexes actually make reads faster, and why not index everything?**
An index is usually a B-tree keyed on the indexed column(s), so the engine
navigates to matching rows in O(log n) instead of scanning all n rows. You don't
index everything because each index is a maintained copy that must be updated on
every write (write amplification), consumes disk and memory, and can even slow
the planner. Index what you filter/join/sort on; drop what nothing uses. It's a
read-speed-for-write-cost trade, made per column.

**Q: What's a composite index and why does column order matter?**
An index on multiple columns, e.g. `(tenant_id, created_at)`. Because it's
sorted by the leftmost column first, it serves queries that filter on a
*leftmost prefix* — `tenant_id`, or `tenant_id AND created_at` — but *not* a
query filtering on `created_at` alone. So order columns by how you query:
equality/most-selective/most-common filter first. Getting the order wrong means
the index sits unused.

**Q: When would you reach for NoSQL instead of a relational schema?**
When the access patterns are known and simple (key lookups), you need massive
horizontal scale or very high write throughput beyond one node, the data is
naturally document-shaped or the schema evolves constantly, and you can live
without cross-entity JOINs and (often) multi-row transactions. Document stores
(MongoDB) for flexible nested data, wide-column (Cassandra) for write-heavy
time-series at scale, key-value (DynamoDB/Redis) for simple high-QPS lookups.
The cost: you denormalize by necessity and own integrity in application code —
you're trading the relational guarantees for flexibility and scale, so only do
it when you actually need what it's selling.

**Q: A query is slow in production. Walk me through it as the SRE, not the DBA.**
`EXPLAIN ANALYZE` the query. Is it a **Seq Scan** on a big table where you
expected an index — missing/unused index, or a non-sargable predicate
(function on the column, leading-wildcard `LIKE`)? Is a **JOIN** producing far
more rows than expected — bad join order, missing FK index, or exploding
cardinality? Are the planner's **row estimates** wildly off — stale statistics,
run `ANALYZE`? Is it fast alone but slow under load — lock contention or
saturated I/O, not the plan? That triage (scan vs index, join blowup, stale
stats, contention) is the SRE's job; deep planner/statistics tuning is where you
loop in the DBA.

**Q: The schema's already live and wrong — how do you change a column type
safely on a huge table?**
Don't do a blocking `ALTER ... TYPE` that rewrites and locks the whole table.
The safe pattern: add a new column of the correct type, backfill it in batches
(so you don't hold one giant transaction/lock), have the app dual-write to both,
switch reads to the new column, then drop the old one. Tools like
`pg_online_schema_change`/`gh-ost` (MySQL) automate this. The whole ordeal is
why you pick the right type *before* the table is huge — the migration is the
expensive part, which is the entire point of getting types right up front.

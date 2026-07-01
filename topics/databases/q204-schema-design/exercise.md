# Q204 — Hands-on lab: feel good and bad schema design

You will build small schemas and *feel* the difference: an over-normalized
design that forces a 5-way JOIN, a missing foreign key that lets orphaned rows
rot your data, a too-small integer column that overflows, and the fixes for
each. Reading about normalization is forgettable; watching Postgres reject an
orphan insert (or silently accept one) is not.

---

## 0. Prerequisites

Pick **one** path:

**Path A — Postgres in Docker (recommended, matches production reality):**

```bash
docker run -d --name schema-lab -e POSTGRES_PASSWORD=pw -p 5432:5432 postgres:16-alpine
sleep 5
# open a psql shell inside the container:
docker exec -it schema-lab psql -U postgres
```

**Path B — SQLite (no Docker, everything is local):**

```bash
sqlite3 schema-lab.db
```
> SQLite note: foreign keys are **off by default**. Run `PRAGMA foreign_keys = ON;`
> at the start of every session or FK enforcement in section 2 won't fire.

Everything below is written for Postgres. Differences for SQLite are called out
inline. Run the DDL in your shell, then the queries.

---

## 1. Over-normalization: the 5-way JOIN you can feel

A textbook-"correct" normalization can split a single concept across so many
tables that every read becomes a JOIN marathon. Build the extreme version:

```sql
CREATE TABLE countries (id serial PRIMARY KEY, name text NOT NULL);
CREATE TABLE cities    (id serial PRIMARY KEY, name text NOT NULL,
                        country_id int REFERENCES countries(id));
CREATE TABLE addresses (id serial PRIMARY KEY, street text,
                        city_id int REFERENCES cities(id));
CREATE TABLE customers (id serial PRIMARY KEY, name text NOT NULL,
                        address_id int REFERENCES addresses(id));
CREATE TABLE orders    (id serial PRIMARY KEY, customer_id int REFERENCES customers(id),
                        total numeric(10,2));
```
> SQLite: replace `serial PRIMARY KEY` with `INTEGER PRIMARY KEY` and `text`/`int`
> work as-is.

Seed a little data:

```sql
INSERT INTO countries (name) VALUES ('France');
INSERT INTO cities (name, country_id) VALUES ('Paris', 1);
INSERT INTO addresses (street, city_id) VALUES ('1 rue de Rivoli', 1);
INSERT INTO customers (name, address_id) VALUES ('Alice', 1);
INSERT INTO orders (customer_id, total) VALUES (1, 42.00);
```

Now answer a dead-simple business question — *"list orders placed in France"* —
and count the JOINs it takes:

```sql
SELECT o.id, cu.name, ci.name AS city, co.name AS country, o.total
FROM orders o
JOIN customers cu ON cu.id = o.customer_id
JOIN addresses a  ON a.id  = cu.address_id
JOIN cities    ci ON ci.id = a.city_id
JOIN countries co ON co.id = ci.country_id
WHERE co.name = 'France';
```

Five tables to answer one trivial question. On a real dataset with millions of
orders, that's five index lookups (or worse, scans) per row, and the query
planner has 5! = 120 possible join orders to consider. **This is the cost of
normalizing past the point of usefulness.** `country` and `city` are slowly-
changing reference data that virtually never update independently for a given
address — splitting them into their own tables bought you almost no
consistency benefit and cost you JOINs on every single read.

**See what the planner does** (Postgres):

```sql
EXPLAIN ANALYZE
SELECT o.id FROM orders o
JOIN customers cu ON cu.id = o.customer_id
JOIN addresses a  ON a.id  = cu.address_id
JOIN cities    ci ON ci.id = a.city_id
JOIN countries co ON co.id = ci.country_id
WHERE co.name = 'France';
```
> SQLite: use `EXPLAIN QUERY PLAN` (drop `ANALYZE`).

### The fix: denormalize the read path

`country` and `city` are naturally attributes of an address. Fold them back in:

```sql
CREATE TABLE addresses_v2 (
  id serial PRIMARY KEY,
  street  text,
  city    text NOT NULL,
  country text NOT NULL
);
CREATE TABLE customers_v2 (
  id serial PRIMARY KEY,
  name text NOT NULL,
  address_id int REFERENCES addresses_v2(id)
);
CREATE TABLE orders_v2 (
  id serial PRIMARY KEY,
  customer_id int REFERENCES customers_v2(id),
  total numeric(10,2)
);

INSERT INTO addresses_v2 (street, city, country) VALUES ('1 rue de Rivoli','Paris','France');
INSERT INTO customers_v2 (name, address_id) VALUES ('Alice', 1);
INSERT INTO orders_v2 (customer_id, total) VALUES (1, 42.00);
```

The same question is now two JOINs, and `country` is an indexable column:

```sql
CREATE INDEX ON addresses_v2 (country);

SELECT o.id, c.name, a.city, a.country, o.total
FROM orders_v2 o
JOIN customers_v2 c ON c.id = o.customer_id
JOIN addresses_v2 a ON a.id = c.address_id
WHERE a.country = 'France';
```

**Takeaway:** normalization prevents update anomalies (change "France" in one
place, not a million rows). But data that never changes independently doesn't
need its own table — folding it back in trades a theoretical anomaly you'll
never hit for real read performance. Balance, don't cargo-cult 3NF.

---

## 2. Missing foreign key: silent data corruption

This is the scary one. Build two tables where `orders.customer_id` has **no**
foreign key:

```sql
CREATE TABLE customers_nofk (id serial PRIMARY KEY, name text NOT NULL);
CREATE TABLE orders_nofk    (id serial PRIMARY KEY, customer_id int, total numeric(10,2));

INSERT INTO customers_nofk (name) VALUES ('Alice');       -- id = 1
INSERT INTO orders_nofk (customer_id, total) VALUES (1, 10.00);   -- valid
INSERT INTO orders_nofk (customer_id, total) VALUES (999, 20.00); -- ORPHAN — accepted!
```

That second insert points at a customer that does not exist. The database
**accepted it without complaint.** You now have an order that can never be
attributed to a customer. Find the damage:

```sql
SELECT o.id, o.customer_id
FROM orders_nofk o
LEFT JOIN customers_nofk c ON c.id = o.customer_id
WHERE c.id IS NULL;      -- returns order 2 (customer_id 999) — corrupt
```

In production this happens from race conditions, buggy deploys, bad backfills,
or a `DELETE FROM customers` that forgot the orders. It corrupts *slowly and
silently* — reports go subtly wrong, JOINs drop rows, and nobody notices until
finance does.

### The fix: declare the foreign key

```sql
CREATE TABLE customers_fk (id serial PRIMARY KEY, name text NOT NULL);
CREATE TABLE orders_fk (
  id serial PRIMARY KEY,
  customer_id int NOT NULL REFERENCES customers_fk(id),   -- the FK
  total numeric(10,2)
);

INSERT INTO customers_fk (name) VALUES ('Alice');            -- id = 1
INSERT INTO orders_fk (customer_id, total) VALUES (1, 10.00);   -- ok
INSERT INTO orders_fk (customer_id, total) VALUES (999, 20.00); -- REJECTED
```

That last insert fails:

```
ERROR:  insert or update on table "orders_fk" violates foreign key constraint
DETAIL: Key (customer_id)=(999) is not present in table "customers_fk".
```
> SQLite (with `PRAGMA foreign_keys = ON`): `Error: FOREIGN KEY constraint failed`.

The database now **structurally refuses** orphaned rows. Deleting a referenced
customer is also blocked (or cascades, if you say `ON DELETE CASCADE`):

```sql
DELETE FROM customers_fk WHERE id = 1;
-- ERROR: update or delete on "customers_fk" violates foreign key constraint on "orders_fk"
```

**Takeaway:** without a foreign key, your application code is the *only* thing
preventing orphans — and application code has bugs, races, and forgotten edge
cases. The FK makes referential integrity a property of the schema, enforced on
every write path including the ones you forgot about (manual fixes, scripts,
that one legacy service).

---

## 3. Wrong data type: the integer that overflows

Ship an ID or a counter as a 32-bit `int` and you've planted a time bomb: it
overflows at 2,147,483,647. Watch it (Postgres):

```sql
CREATE TABLE events_small (id serial PRIMARY KEY, kind text);   -- serial = int4
-- pretend we've grown; jump the sequence near the 32-bit ceiling:
SELECT setval('events_small_id_seq', 2147483646);
INSERT INTO events_small (kind) VALUES ('ok');       -- id = 2147483647, the max
INSERT INTO events_small (kind) VALUES ('boom');     -- OVERFLOW
```

The second insert fails:

```
ERROR:  nextval: reached maximum value of sequence "events_small_id_seq" (2147483647)
```

Your table can take no more rows. This is a real, recurring production outage
(a famous one took out a large ticketing platform's ID column). The fix is to
have used a 64-bit type from the start:

```sql
CREATE TABLE events_big (id bigserial PRIMARY KEY, kind text);  -- bigserial = int8
```
> SQLite is dynamically typed — `INTEGER` is already 64-bit, so it won't
> reproduce this overflow. This is exactly why the type choice is invisible
> until you're on a real engine that enforces widths. Read the Postgres output.

While you're here, see two more type traps:

```sql
-- Money in floating point loses cents:
SELECT 0.1::float8 + 0.2::float8;        -- 0.30000000000000004  (wrong for money)
SELECT 0.1::numeric + 0.2::numeric;      -- 0.3                   (use numeric/decimal)

-- CHAR(n) pads with spaces; VARCHAR/text does not:
SELECT ('ab'::char(5) || '|') AS padded, ('ab'::varchar || '|') AS not_padded;
-- padded = 'ab   |'   not_padded = 'ab|'
```

**Takeaway:** data types are a *durability* decision. Ask "what's the largest
value this could ever hold?" and "does this need exact arithmetic?" up front —
because `ALTER TABLE ... ALTER COLUMN TYPE bigint` on a live billion-row table
means a full rewrite, locks, and a maintenance window. Default to `bigint` for
identifiers, `numeric` for money, `text`/`varchar` for strings, and
`timestamptz` for time.

---

## 4. Constraints: encode invariants in the schema

`NOT NULL`, `UNIQUE`, and `CHECK` push rules the app *hopes* to enforce down
into the database, where they hold no matter which client writes.

```sql
CREATE TABLE accounts (
  id      serial PRIMARY KEY,
  email   text    NOT NULL UNIQUE,                 -- required + no duplicates
  balance numeric NOT NULL DEFAULT 0 CHECK (balance >= 0)   -- can't go negative
);

INSERT INTO accounts (email, balance) VALUES ('a@x.com', 100);   -- ok
INSERT INTO accounts (email, balance) VALUES ('a@x.com', 50);    -- UNIQUE violation
INSERT INTO accounts (email, balance) VALUES ('b@x.com', -5);    -- CHECK violation
INSERT INTO accounts (email)          VALUES (NULL);             -- NOT NULL violation
```

Each of the last three is rejected by the database itself:

```
ERROR: duplicate key value violates unique constraint "accounts_email_key"
ERROR: new row violates check constraint "accounts_balance_check"
ERROR: null value in column "email" violates not-null constraint
```

**Takeaway:** a constraint is a guarantee that's true for *every* writer,
forever — including the migration script, the psql session, and the service
you haven't written yet. Business rules enforced only in app code are enforced
only until the next code path forgets them.

---

## 5. Indexing: the read/write trade-off

Indexes make reads fast and writes slower — every index must be updated on
every `INSERT`/`UPDATE`/`DELETE`. See both halves:

```sql
CREATE TABLE big (id bigserial PRIMARY KEY, email text, created_at timestamptz DEFAULT now());
INSERT INTO big (email)
SELECT 'user' || g || '@x.com' FROM generate_series(1, 200000) g;

-- unindexed lookup: sequential scan
EXPLAIN ANALYZE SELECT * FROM big WHERE email = 'user150000@x.com';

CREATE INDEX idx_big_email ON big (email);

-- indexed lookup: index scan, far cheaper
EXPLAIN ANALYZE SELECT * FROM big WHERE email = 'user150000@x.com';
```
> SQLite: use `EXPLAIN QUERY PLAN SELECT ...` before and after
> `CREATE INDEX idx_big_email ON big(email);` — it flips from "SCAN big" to
> "SEARCH big USING INDEX".

You'll see the plan change from a **Seq Scan** (reads every row) to an **Index
Scan**. But note the cost you *didn't* see: those 200k inserts each had to
maintain the primary-key index, and now maintain the email index too.

**Takeaway:** index the columns you actually filter/join/sort on — no more.
Every unused index is pure write overhead and wasted space. "Add an index" is
not free; it's a trade you make deliberately against your write path.

---

## 6. Cleanup

```bash
# Path A (Docker):
docker rm -f schema-lab

# Path B (SQLite):
rm schema-lab.db
```

---

## What you proved

- **Over-normalization** turned one trivial question into a 5-way JOIN;
  folding stable reference data back in cut it to two and made the filter
  indexable.
- **A missing foreign key** silently accepted an orphaned order; declaring the
  FK made the database reject it structurally.
- **A 32-bit integer** overflowed at ~2.1 billion and bricked the table; a
  64-bit type from the start avoids it — and `float` money loses cents where
  `numeric` doesn't.
- **Constraints** (`NOT NULL`/`UNIQUE`/`CHECK`) enforced invariants for every
  writer, not just the app.
- **An index** flipped a sequential scan into an index scan — at a write cost
  you pay on every mutation.

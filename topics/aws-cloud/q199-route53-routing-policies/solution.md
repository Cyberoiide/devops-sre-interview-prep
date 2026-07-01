# Solution — Route 53 routing policies

## The core insight

Route 53 is a DNS service, so a "routing policy" decides **which record
value(s) it returns for a query**. There are **seven** policies, each keyed on a
**different decision criterion**. The skill is matching the criterion to the
requirement — and not confusing the two look-alike pairs (Simple/Multivalue and
Geolocation/Geoproximity).

## The seven policies, precisely

### 1. Simple

A single record set. It can hold **one or multiple values**; if multiple, Route
53 returns **all of them in random order** and the client's resolver picks one.
**No health checks** — a dead endpoint stays in the answer. Use it for a plain
"this name → this value" mapping. It is *not* round robin with health awareness;
that's Multivalue.

### 2. Weighted

Multiple records with the **same name and type**, each assigned a **weight**;
Route 53 returns each in proportion to `weight / sum(weights)`. Set a weight to
`0` to drain a record. Use for **canary releases, A/B testing, blue-green, and
gradual traffic shifting** between environments. Each record needs a unique
`set_identifier`.

### 3. Latency-based

Records tagged with **AWS Regions**; Route 53 returns the one that gives the
querying user the **lowest network latency** to that Region. Key subtlety:
**lowest latency, not nearest geography** — AWS measures actual latency, so the
"fastest" Region for a user isn't always the geographically closest. Use it to
serve users from whichever Region responds quickest.

### 4. Failover

**Active/passive.** You designate a **PRIMARY** and a **SECONDARY** record; Route
53 serves the primary **while its health check passes**, and **flips to the
secondary when the primary's health check fails**. This is the policy for
"failover to a backup region/site." It is **entirely health-check driven** — no
health check, no failover.

### 5. Geolocation

Routes on the **location of the *user*** — by **continent, country, or (in the
US) state**. Use it for **content localization** ("EU users → EU site, JP users
→ Japanese site") and **compliance/geo-fencing** (serve or block by jurisdiction,
license restrictions). You should configure a **`default` (`*`) record** as a
catch-all for locations you didn't explicitly map, or those users get no answer.

### 6. Geoproximity

Routes on the **geographic distance between the user and the *resource***, and
uniquely lets you apply a **`bias`**: a positive bias **expands** a resource's
effective region so it draws users from farther away, a negative bias shrinks
it. Use it to shift traffic toward/away from a location by tuning the bias rather
than hard geographic borders. **Requires Route 53 Traffic Flow** (the visual
policy editor / traffic policies).

### 7. Multivalue answer

Returns **up to 8 healthy records** at random, each optionally associated with a
**health check** — only **healthy** values are returned. It's "Simple **with
health checks and multiple answers**," giving crude client-side load
distribution and improving availability. AWS is explicit that it is **not a
substitute for an Elastic Load Balancer** — there's no connection-level
balancing, session affinity, or sub-second failover; it's DNS-level and subject
to TTL/caching.

## The two accuracy traps

### Simple vs Multivalue ("is Simple round robin?")

No. Both can return multiple values in random order, but **Simple has no health
checking** — it will happily hand out a dead endpoint. **Multivalue answer adds
per-record health checks** and returns only healthy values (up to 8). So if you
want "return several endpoints and skip the dead ones," that's **Multivalue**,
not Simple. And neither is a real load balancer: DNS caching and TTLs mean
distribution is approximate and failover is slow relative to an LB.

### Geolocation vs Geoproximity

- **Geolocation** — decision input is **where the *user* is** (continent /
  country / US state). Discrete buckets, plus a default catch-all. "European
  users get the EU stack."
- **Geoproximity** — decision input is **distance between user and resource**,
  continuous, with a **`bias`** knob to grow or shrink a resource's pull.
  Requires **Traffic Flow**.

Mnemonic: Geo**location** = the user's *location*; Geo**proximity** = *proximity*
(distance) to a resource, tunable with bias.

## How health checks tie it together

**Failover** and **Multivalue** depend directly on **Route 53 health checks**,
and Weighted/Latency/Geolocation records can *also* be associated with a health
check so an unhealthy target is dropped from that policy's candidate set. A
health check monitors an endpoint (HTTP/HTTPS/TCP), a **CloudWatch alarm**, or
the status of **other health checks** (calculated). Two parameters set the
detection speed: `request_interval` (10s or 30s) and `failure_threshold`
(consecutive checks). Time to fail over ≈ `failure_threshold × request_interval`
**plus the record's TTL** (clients keep cached answers until TTL expires) — which
is why failover records use **low TTLs** (e.g. 60s).

## Answering the interviewer's scenario

> "Route 53 has seven policies, each keyed on a different criterion. For 'failover
> to a backup region' I'd use the **Failover** policy — a PRIMARY and SECONDARY
> record, with a **health check** on the primary; Route 53 flips to the secondary
> when the primary goes unhealthy, and I'd keep the TTL low so clients follow the
> flip quickly. For 'European users to the EU stack' I'd use **Geolocation**,
> which routes on the *user's* location, with a **default** record as a catch-all.
> If I needed both behaviours combined I'd compose them with **Traffic Flow**.
> I'd be careful not to reach for **Geoproximity** here — that routes on
> *distance to the resource* with a bias knob, which is a different question — or
> to assume **Simple** does health-aware round robin; that's **Multivalue
> answer**, the only multi-record policy besides Failover that respects health
> checks."

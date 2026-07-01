# Deeper — follow-up interview questions

### 1. Is "Simple" routing just round robin?

No — and this is the trap. A Simple record can hold multiple values, which Route
53 returns in random order (the resolver then picks), so it *looks* round-robin,
but it has **no health checks**: a failed endpoint stays in the answer and
clients keep hitting it. The policy that adds per-record health checks and
returns only healthy values (up to 8) is **Multivalue answer**. If you want
health-aware distribution across several endpoints, use Multivalue, not Simple.

### 2. Multivalue answer vs an actual load balancer?

Multivalue is **DNS-level** and coarse: Route 53 hands back up to 8 *healthy*
records at random, and the client picks. There's **no connection-level
balancing, no session stickiness, no sub-second failover**, and it's subject to
**DNS caching / TTLs** — resolvers and clients cache answers, so a dead endpoint
can be used until the TTL expires. An **ELB** balances actual connections in real
time, does health checks continuously, and reacts far faster. AWS explicitly says
Multivalue is **not a replacement for an ELB**; it's a lightweight availability
boost, not a balancer.

### 3. Geolocation vs Geoproximity — the crisp distinction?

**Geolocation** routes on **where the *user* is** — continent, country, or US
state — in discrete buckets, with a `*` default catch-all. **Geoproximity**
routes on the **geographic distance between the user and the *resource***, and
adds a **`bias`**: increase it to expand a resource's region (pull users from
farther away), decrease it to shrink it. Geoproximity also **requires Traffic
Flow**. Rule of thumb: user-location rules → Geolocation; "shift traffic between
sites by distance/bias" → Geoproximity.

### 4. Latency-based routing sends users to the "fastest" region — is that the nearest one?

Not necessarily. Latency-based routing uses **AWS's measured network latency**
between the user's location and each tagged Region, and returns the
lowest-latency Region. Because of peering, backbone topology, and transit paths,
the lowest-latency Region can differ from the geographically nearest one. If you
want strictly geographic/administrative control, use Geolocation or Geoproximity
instead — latency routing optimizes for *speed*, not *place*.

### 5. How does failover actually detect that the primary is down, and how fast is it?

Via a **Route 53 health check** attached to the PRIMARY record. The check probes
the endpoint (HTTP/HTTPS/TCP), or watches a **CloudWatch alarm**, or aggregates
**other health checks** (calculated health check). Detection time ≈
`failure_threshold × request_interval` (e.g. 3 × 30s = ~90s), and then clients
must wait for the **record's TTL** to expire before they re-query and get the
secondary. So total failover time ≈ detection + TTL — which is why failover
records use **low TTLs** (30–60s). DNS failover is inherently slower than an
LB's; set expectations accordingly.

### 6. Why do multi-record policies need a `SetIdentifier`?

Because Weighted, Latency, Failover, Geolocation, Geoproximity, and Multivalue
all create **multiple record sets with the same name and type**. Route 53 needs a
way to tell them apart within that group — the **`SetIdentifier`** is that unique
label per record. Without it you can't have two `api.example.com A` records
coexisting under a routing policy. (Simple routing, being a single record set,
doesn't use one.)

### 7. What can a Route 53 health check actually monitor?

Three kinds: **endpoint checks** (HTTP, HTTPS, or TCP to an IP/domain, optionally
matching a string in the response body), **CloudWatch alarm checks** (healthy
follows the alarm state — useful for private resources Route 53 can't reach
directly, or metric-based conditions), and **calculated checks** (combine the
results of up to 256 child health checks with AND/OR/NOT logic). You tune
`request_interval` (10s or 30s) and `failure_threshold`. Health checkers query
from **multiple AWS locations**, so a target is only "unhealthy" when enough
locations agree — reducing false positives from a single bad path.

### 8. Can you combine policies (e.g. latency AND weighted)?

Yes, by **nesting** with **Route 53 Traffic Flow** / traffic policies. Traffic
Flow lets you build a tree: e.g. a latency rule at the top that, within the
chosen Region, points at a **weighted** group for a canary, each of whose targets
has a **health check**. This is how real designs express "route to the fastest
Region, and within it do a 90/10 canary, and fail out any unhealthy target."
Alias records also matter here — an **alias** points a Route 53 record at an AWS
resource (ELB, CloudFront, S3 website) and is free/health-aware, unlike a raw
`CNAME`.

### 9. Which policies respect health checks?

**Failover** and **Multivalue answer** are the two built around health checks.
But Weighted, Latency-based, Geolocation, and Geoproximity records can **each be
associated with a health check** too, so an unhealthy record is dropped from that
policy's candidate set (e.g. a weighted record with a failing check stops
receiving its share). **Simple** routing is the odd one out — it **cannot** be
associated with a health check, which is exactly why it can return dead
endpoints.

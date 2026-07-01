# q199 — Route 53 routing policies

## The interview question

> "A product team says they want 'DNS-based failover to a backup region, and
> also route European users to our EU stack.' You're the one who has to
> implement it in Route 53. Which routing policies does Route 53 give you, what
> does each actually do, and which ones solve *this* request? Be precise —
> people mix these up."

Route 53 offers **seven** routing policies and interviewers love this question
because the names are deceptively similar (Geolocation vs Geoproximity;
Simple vs Multivalue). A strong answer enumerates all seven, states the
*decision criterion* each uses, and calls out the pairs people conflate.

## The seven policies

| Policy | Routes based on | One-liner |
|---|---|---|
| **Simple** | nothing (single record) | One record; if it has multiple values, all are returned in random order and the *resolver* picks. **No health checks.** |
| **Weighted** | assigned weights | Split traffic by ratio across records (A=70, B=30). Canaries, A/B, gradual shifts. |
| **Latency-based** | lowest network latency to an AWS Region | Send the user to the Region that's *fastest for them* (not necessarily nearest). |
| **Failover** | health check | Active/passive: serve primary while healthy, flip to secondary when the primary's health check fails. |
| **Geolocation** | the **user's location** (continent/country/state) | Route by *where the user is* — content localization, compliance. Has a "default" catch-all. |
| **Geoproximity** | **geographic distance** user↔resource, with a **bias** knob | Route by distance, and you can expand/shrink a resource's region with `bias`. (Requires Traffic Flow.) |
| **Multivalue answer** | health checks, returns up to 8 healthy records | Like Simple-with-health-checks: returns multiple *healthy* records for client-side balancing. Not a substitute for an LB. |

## Two accuracy traps to nail

- **"Simple = round robin" is wrong.** A Simple record can hold multiple values
  and Route 53 returns them in random order, but there are **no health checks**
  — a dead endpoint stays in the answer. **Multivalue answer** is the one that
  adds per-record health checks and only returns *healthy* values. Neither is a
  real load balancer; they're client-side/DNS-level distribution.
- **Geolocation ≠ Geoproximity.** **Geolocation** routes on **where the *user*
  is** (their continent/country/state) — think "EU users get the EU site."
  **Geoproximity** routes on **geographic distance between the user and the
  *resource*** and lets you shift traffic with a **bias** value (make one
  region "bigger" so it draws users from farther away). Geoproximity requires
  **Route 53 Traffic Flow**.

## Answering *this* request

- "Failover to a backup region" → **Failover** policy (primary/secondary +
  health checks).
- "European users to the EU stack" → **Geolocation** (by user location).
- You'd layer these with **Traffic Flow** if you need both in one policy tree,
  and every policy that flips traffic depends on **Route 53 health checks**.

## Files in this topic

- `exercise.md` — Terraform defining **weighted** and **failover** record sets
  (plus health checks), validated with `terraform plan`; notes on what needs a
  real hosted zone and the (limited) LocalStack Route 53 support.
- `solution.md` — all seven policies with when-to-use, the two accuracy traps in
  depth, and how health checks drive failover.
- `deeper.md` — follow-up interview questions with concise answers.

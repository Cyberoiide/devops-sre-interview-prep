# Solution — Designing a secure VPC

## The core insight

Security in a VPC is **defence in depth**: independent layers so that any single
mistake — an over-broad SG rule, a mislabelled subnet — doesn't expose the whole
estate. The five load-bearing decisions are **segmentation, the two firewall
layers, controlled egress, private service access, and observability.** Below,
each with the *why* an interviewer will push for.

## 1. Subnet segmentation (public vs private, multi-AZ)

Split the VPC into **public** and **private** subnets, replicated across **at
least two Availability Zones**.

- **Public subnet** = has a route `0.0.0.0/0 → internet gateway`. Only put
  things here that genuinely must be internet-facing: the **load balancer**, a
  **bastion/SSM endpoint**, and the **NAT gateway** itself.
- **Private subnet** = **no route to an internet gateway**. Application servers
  and databases live here. They're unreachable from the internet by *routing*,
  before any firewall even evaluates a packet — that's the strongest guarantee.
- **Multi-AZ** because a subnet is AZ-scoped; spreading tiers across ≥2 AZs is
  what makes the design survive an AZ failure. (Security and availability
  overlap here.)

The tell of a good answer: "the database is in a private subnet with no IGW
route, so even a wide-open security group can't make it internet-reachable."

## 2. The two firewall layers: Security Groups and NACLs

This is the question inside the question. Both filter traffic, but they're
different tools:

| | **Security Group** | **Network ACL** |
|---|---|---|
| Attaches to | ENI / instance | Subnet |
| State | **Stateful** (return traffic auto-allowed) | **Stateless** (must allow both directions) |
| Rules | **Allow only** (implicit deny-all) | **Allow *and* Deny** |
| Evaluation | All rules together (most-permissive union) | **In rule-number order**, first match wins |
| Typical use | Primary, per-workload control | Coarse subnet guardrail / explicit blocks |

**Why "stateful vs stateless" matters in practice:** with a Security Group, if
you allow inbound `443`, the response is automatically allowed back out — you
don't write a return rule. With a **NACL you must allow the return traffic
explicitly**, and because responses use **ephemeral ports** (roughly
1024–65535), a NACL that allows inbound 443 but forgets outbound ephemeral will
silently break the connection. That gotcha is a favourite interview trap.

**How to use them together:** Security Groups do the real work (tiered,
least-privilege, referencing each other — the ALB SG, the app SG that only
trusts the ALB SG, the DB SG that only trusts the app SG). NACLs are a **coarse
subnet-level backstop** — e.g. an explicit `Deny` of a known-bad CIDR, or a
blanket rule that the DB subnet never talks to `0.0.0.0/0`. Because SGs can't
express *deny*, NACLs are the layer for explicit blocklists.

**Least privilege everywhere:** reference **SGs by ID, not CIDR ranges** ("allow
5432 from the app SG" not "from 10.0.0.0/16"), open the narrowest port range,
and give the DB SG **no internet egress** — a database has no reason to call out
to `0.0.0.0/0`, and removing that egress limits data exfiltration if it's
compromised.

## 3. Controlled egress: NAT gateways

Private-subnet resources still need **outbound** internet for OS updates,
pulling images, calling external APIs. A **NAT gateway** (sitting in a *public*
subnet) gives them exactly that: **outbound-initiated connections only, no
inbound**. The private route table sends `0.0.0.0/0 → NAT gateway`; the NAT
gateway forwards via the IGW using its own public IP. Nothing on the internet
can initiate a connection back in.

Notes an interviewer likes: NAT gateways are **AZ-scoped** (one per AZ for HA,
or you create a single-AZ dependency), they're **managed and scale
automatically** (vs a self-managed NAT *instance*), and they **cost per hour +
per GB** — which is a real reason to add VPC endpoints (next).

## 4. Private access to AWS services: VPC endpoints

By default, an instance in a private subnet reaching **S3 or DynamoDB** would go
out through the NAT gateway and across the public internet to the service's
public endpoint. **VPC endpoints** keep that traffic on the **AWS backbone**,
which is more secure (never traverses the internet), often faster, and cheaper
(no NAT data-processing charge). Two kinds:

- **Gateway endpoints** — only **S3 and DynamoDB**. Implemented as a **route
  table entry** (a prefix-list route to the service). **Free.** You can even run
  a private subnet with *no NAT at all* and still reach S3.
- **Interface endpoints (PrivateLink)** — for **most other services** (SSM, ECR,
  Secrets Manager, KMS, etc.). Implemented as an **ENI with a private IP** in
  your subnet; you hit the service via that ENI. **Billed hourly + per GB.**

Bonus security: endpoints support **endpoint policies** so you can restrict, at
the network layer, *which* S3 buckets or actions are reachable through them.

## 5. Observability: VPC Flow Logs (+ WAF at the edge)

- **VPC Flow Logs** record metadata for **accepted and rejected** IP flows
  (src/dst, ports, bytes, action) to CloudWatch Logs or S3. You want them on for
  **forensics** (what talked to what during an incident), **intrusion
  detection**, and the everyday "why can't A reach B" (a `REJECT` on the right
  port points straight at the SG/NACL at fault). They're metadata, not packet
  contents — cheap to run, invaluable after the fact.
- **WAF** belongs at the **edge** (on the ALB or CloudFront), not in the VPC
  routing layer — it's L7 (SQLi/XSS/bots), complementing the L3/L4 SG+NACL
  filtering. See q207.

## The interview summary

> "I'd segment the VPC into public and private subnets across at least two AZs —
> only the load balancer, bastion, and NAT gateways go public; app and DB sit in
> private subnets with **no internet-gateway route**, so they're unreachable by
> routing alone. I use **Security Groups** as the primary, stateful, per-tier
> control referencing each other by ID for least privilege, and **NACLs** as a
> stateless subnet-level backstop for explicit denies. Private subnets get
> outbound-only internet via a **NAT gateway**, and I add **VPC endpoints** —
> gateway for S3/DynamoDB, interface for the rest — so service traffic stays on
> the AWS backbone instead of the internet. I turn on **VPC Flow Logs** for
> forensics and put **WAF** at the edge for L7. The theme is defence in depth:
> routing, two firewall layers, controlled egress, private service paths, and
> logging."

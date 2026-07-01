# Deeper — follow-up interview questions

### 1. Security Group vs NACL — the real differences?

A **Security Group** is **stateful**, attaches to an **ENI/instance**, and is
**allow-only** (everything not allowed is implicitly denied); all its rules are
evaluated together as a permissive union. A **NACL** is **stateless**, attaches
to a **subnet**, supports **both allow and deny** rules, and is evaluated **in
rule-number order, first match wins**. Practically: SGs are your primary
per-workload control; NACLs are a coarse subnet guardrail and the *only* place
you can express an explicit **deny** (SGs can't deny). The stateful/stateless
distinction is the one that bites people (next question).

### 2. Why does "stateless" force you to write return-traffic rules on a NACL?

Because a NACL doesn't remember connections. With a **stateful** SG, allowing
inbound TCP 443 automatically permits the response back out — the SG tracks the
flow. A **NACL** evaluates each direction independently, so allowing inbound 443
does nothing for the reply; you must **also allow outbound on the ephemeral port
range** (~1024–65535) that the response uses, and often the matching inbound
ephemeral for outbound-initiated flows. Forgetting the ephemeral range is the
classic "connection hangs / times out but the SG looks fine" NACL bug.

### 3. Gateway endpoint vs interface endpoint?

**Gateway endpoints** serve **only S3 and DynamoDB**, are implemented as a
**route-table entry** (a managed prefix-list route to the service), and are
**free**. **Interface endpoints** use **AWS PrivateLink**: an **ENI with a
private IP** in your subnet that you reach the service through; they cover
**most other services** and are **billed per hour + per GB**. Both keep traffic
on the AWS backbone instead of the public internet. A nice consequence: with an
S3 gateway endpoint a private subnet can reach S3 with **no NAT gateway at all**.

### 4. NAT gateway vs internet gateway — and can a private subnet have both?

An **internet gateway (IGW)** enables **bidirectional** internet connectivity
(inbound and outbound) for resources with public IPs — it's what makes a subnet
"public." A **NAT gateway** enables **outbound-only** internet for resources with
private IPs — inbound-initiated connections are impossible. A subnet is "private"
precisely because its route table points `0.0.0.0/0` at a **NAT gateway (or
nothing)**, not an IGW. If you route a private subnet to the IGW it stops being
private. The NAT gateway itself lives in a *public* subnet so it can reach the
IGW.

### 5. What do VPC Flow Logs capture, and what can't they do?

They capture **flow metadata**: source/dest IP and port, protocol, packet/byte
counts, start/end time, and the **action (ACCEPT/REJECT)** — per network
interface, subnet, or whole VPC, delivered to CloudWatch Logs or S3. Great for
forensics, anomaly detection, and diagnosing connectivity ("a REJECT on 5432
means the SG/NACL blocked it"). What they **can't** do: they're **not packet
capture** — no payloads — and some traffic is excluded (e.g. to the Amazon DNS
server, DHCP, instance-metadata, license activation). For payload inspection
you'd use **VPC Traffic Mirroring** instead.

### 6. How do you give a private instance outbound internet without any NAT gateway?

Several options. For **AWS services**, use **VPC endpoints** (gateway for
S3/DynamoDB, interface for others) — no NAT needed at all, and that covers a lot
(ECR pulls, SSM, Secrets Manager, patching from S3-backed repos). For general
internet you could run a **NAT instance** (cheaper, self-managed, a SPOF unless
you build HA) instead of a managed NAT gateway. And for admin access you don't
need inbound SSH at all — **SSM Session Manager** over an interface endpoint
gives you a shell with no bastion and no open port 22.

### 7. Where does WAF fit, and why not "in the VPC"?

WAF is a **Layer-7** control that attaches to a **CloudFront distribution, an
Application Load Balancer, or API Gateway** — the *edge/entry* of your traffic —
not to VPC route tables. It inspects HTTP(S) for SQLi, XSS, bad bots, and does
rate limiting. SGs and NACLs operate at **L3/L4** (IPs and ports) and can't see
an SQL-injection string. So they're complementary: WAF filters *malicious
application requests* at the edge; SG/NACL filter *which hosts and ports can
talk* inside the network.

### 8. What's the difference between subnet CIDR sizing choices and security?

Mostly indirect, but interviewers probe it. Over-large subnets waste address
space and can encourage flat, permissive rules; too-small subnets run out of IPs
(remember AWS reserves **5 addresses per subnet**). The security-relevant point
is **segmentation by function** — separate subnets/route tables per tier so you
can attach different NACLs and route differently (public vs private is the
minimum; larger designs add dedicated data, management, and endpoint subnets).
Segmentation, not CIDR math, is what limits blast radius.

### 9. How do you keep this posture from drifting (public access sneaking back)?

Use **AWS Config rules** (or Security Hub) to continuously check for things like
security groups open to `0.0.0.0/0` on sensitive ports, subnets that gained an
IGW route, or unrestricted egress, and alert/auto-remediate. Enforce the
non-negotiables preventively with **SCPs** (e.g. deny creating an IGW route on
tagged private subnets) and codify the whole VPC in **Terraform** so changes go
through review rather than console clicks. Flow Logs + GuardDuty add runtime
detection on top.

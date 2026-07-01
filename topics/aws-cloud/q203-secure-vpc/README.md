# q203 — Designing a secure VPC

## The interview question

> "We're standing up a new workload in a fresh AWS account — a public-facing web
> tier and a private application + database tier. You own the network. Talk me
> through the VPC design and, specifically, the security best practices you'd
> insist on. Assume the interviewer will push on *why* for each one."

This is a design question disguised as a checklist. Anyone can say "use security
groups"; the signal is whether you can lay out a **defence-in-depth** network —
subnet tiering, the two firewall layers and how they differ, egress control,
private access to AWS services, and observability — and justify each choice.

## The mental model: layers, not a wall

A secure VPC isn't one control; it's several independent layers so that one
misconfiguration doesn't expose everything:

1. **Segmentation** — public vs private subnets across multiple AZs.
2. **Two firewall layers** — Security Groups (stateful, per-ENI) and NACLs
   (stateless, per-subnet).
3. **Controlled egress** — NAT gateways for private-subnet outbound; no direct
   inbound.
4. **Private paths to AWS services** — VPC endpoints so S3/DynamoDB/etc traffic
   never touches the public internet.
5. **Visibility** — VPC Flow Logs (and Layer-7 protection with WAF at the edge).

## What you should be able to answer

- **Public vs private subnets.** Only internet-*facing* resources (a load
  balancer, a bastion, NAT gateways) go in public subnets; app servers and
  databases go in private subnets with **no route to an internet gateway**.
  Spread across ≥2 AZs.
- **Security Groups vs NACLs** — the classic pairing. SGs are **stateful**,
  attached to ENIs/instances, **allow-only** (implicit deny), evaluated as a
  whole. NACLs are **stateless**, attached to subnets, support **explicit
  allow *and* deny**, evaluated in rule-number order. Know why "stateful vs
  stateless" changes how you write return-traffic rules.
- **VPC Flow Logs** — capture accepted/rejected traffic metadata to
  CloudWatch/S3 for forensics, intrusion detection, and debugging "why can't A
  reach B".
- **VPC endpoints** — **Gateway** endpoints (S3, DynamoDB; free; route-table
  entry) and **Interface** endpoints (most other services; ENI + PrivateLink;
  hourly cost) so private resources reach AWS APIs without a NAT/IGW hop.
- **NAT gateways** — let private subnets make **outbound** connections (updates,
  API calls) while remaining **unreachable from the internet**.
- **WAF at the edge** for L7, and the principle of **least privilege** applied
  to every SG rule.

## Files in this topic

- `exercise.md` — Terraform for a minimal secure VPC (public + private subnets
  over 2 AZs, IGW, NAT gateway, SGs, an S3 gateway endpoint, and Flow Logs),
  validated with `terraform plan`; plus a LocalStack path that builds the VPC
  primitives with the CLI.
- `solution.md` — the full design walkthrough with the *why* behind each layer,
  and a SG-vs-NACL comparison table.
- `deeper.md` — follow-up interview questions with concise answers.

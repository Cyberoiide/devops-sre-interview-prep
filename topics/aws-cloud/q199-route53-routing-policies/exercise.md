# Exercise — Weighted and failover record sets in Route 53

This lab defines **weighted** and **failover** routing in **Terraform** (with
the **health check** that drives failover), validated with `terraform plan`. A
second section exercises Route 53 record CRUD against **LocalStack** so you can
see record sets and set identifiers for real, for free.

## What's honest about this lab

- **The Terraform is real, applyable config** and `plan`s with no spend. Route 53
  is cheap but **not free**: a hosted zone is **$0.50/month** and health checks
  are **~$0.50/month** each — so a live `apply` costs a little; destroy it after.
- **Route 53 routing policies are DNS resolution behaviour** — you can only truly
  *observe* weighted split / failover flipping with a **real hosted zone and a
  real domain** (or by querying the zone's Route 53 name servers with `dig`).
- **LocalStack Community emulates the Route 53 API** (create zone, records, set
  identifiers) well enough to practice authoring and read records back, but it
  **does not run health checks or resolve with policy logic** — so "does it
  actually fail over" needs a real account.

## Prerequisites

- `terraform` >= 1.5 (or `tofu`).
- `docker` + [LocalStack](https://docs.localstack.dev/getting-started/) + the
  `awslocal` wrapper.
- AWS CLI v2, `dig` (from `dnsutils`/`bind-tools`) for the real-account check.

---

## Part A — Define weighted + failover routing in Terraform (`plan`-only)

`providers.tf`:

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}
provider "aws" {
  region = "us-east-1"
}
```

`main.tf`:

```hcl
# A private demo zone (example.internal) so nothing collides with real DNS.
resource "aws_route53_zone" "demo" {
  name = "demo.internal"
}

########################################
# WEIGHTED — 90/10 canary split on api.demo.internal
# Both records share the SAME name+type; weight sets the ratio.
########################################
resource "aws_route53_record" "api_stable" {
  zone_id        = aws_route53_zone.demo.zone_id
  name           = "api.demo.internal"
  type           = "A"
  ttl            = 60
  set_identifier = "stable"           # required to disambiguate same-name records
  weighting_policy_placeholder = true # (see note) -- real attr is weighted_routing_policy below

  weighted_routing_policy {
    weight = 90                        # 90% of answers -> this record
  }
  records = ["10.0.0.10"]
}

resource "aws_route53_record" "api_canary" {
  zone_id        = aws_route53_zone.demo.zone_id
  name           = "api.demo.internal"
  type           = "A"
  ttl            = 60
  set_identifier = "canary"
  weighted_routing_policy {
    weight = 10                        # 10% -> canary
  }
  records = ["10.0.0.20"]
}

########################################
# FAILOVER — active/passive on www.demo.internal, driven by a health check
########################################
# Health check on the PRIMARY endpoint. When it goes unhealthy,
# Route 53 stops answering with the primary and serves the secondary.
resource "aws_route53_health_check" "primary" {
  fqdn              = "primary.demo.internal"
  type              = "HTTPS"
  port              = 443
  resource_path     = "/healthz"
  failure_threshold = 3
  request_interval  = 30
}

resource "aws_route53_record" "www_primary" {
  zone_id         = aws_route53_zone.demo.zone_id
  name            = "www.demo.internal"
  type            = "A"
  ttl             = 60
  set_identifier  = "primary"
  health_check_id = aws_route53_health_check.primary.id
  failover_routing_policy {
    type = "PRIMARY"
  }
  records = ["10.0.1.10"]
}

resource "aws_route53_record" "www_secondary" {
  zone_id        = aws_route53_zone.demo.zone_id
  name           = "www.demo.internal"
  type           = "A"
  ttl            = 60
  set_identifier = "secondary"
  failover_routing_policy {
    type = "SECONDARY"               # served only when PRIMARY is unhealthy
  }
  records = ["10.0.2.10"]
}
```

> **Fix before `apply`:** the line `weighting_policy_placeholder = true` above is
> a deliberate marker — **delete it**. The real weighted config is the
> `weighted_routing_policy { weight = ... }` block plus the required
> `set_identifier`. It's here so you notice the two non-obvious requirements of
> *any* multi-record policy: **every record needs a unique `set_identifier`**,
> and records that share a name+type are distinguished only by that identifier +
> their policy block.

Validate (after deleting the placeholder line):

```bash
terraform init
terraform validate      # "Success!"
terraform plan
```

**Confirm the routing story in the plan:**

1. The two `api.demo.internal` A records share a name/type and differ only by
   `set_identifier` + `weight` (90 vs 10) — that's the weighted split.
2. The `www` records use `failover_routing_policy` with `PRIMARY`/`SECONDARY`,
   and the **PRIMARY carries a `health_check_id`** — failover is *driven by the
   health check*, not by time.
3. Low **TTL (60s)** so clients pick up a failover flip quickly (see deeper.md
   on why TTL matters for failover).

---

## Part B — Route 53 record CRUD on LocalStack (free)

```bash
localstack start -d

# Create a hosted zone.
ZONE=$(awslocal route53 create-hosted-zone \
  --name demo.internal --caller-reference $(date +%s) \
  --query 'HostedZone.Id' --output text)

# Add two weighted records (same name/type, different SetIdentifier + Weight).
awslocal route53 change-resource-record-sets --hosted-zone-id $ZONE --change-batch '{
  "Changes": [
    {"Action":"CREATE","ResourceRecordSet":{
      "Name":"api.demo.internal","Type":"A","TTL":60,
      "SetIdentifier":"stable","Weight":90,
      "ResourceRecords":[{"Value":"10.0.0.10"}]}},
    {"Action":"CREATE","ResourceRecordSet":{
      "Name":"api.demo.internal","Type":"A","TTL":60,
      "SetIdentifier":"canary","Weight":10,
      "ResourceRecords":[{"Value":"10.0.0.20"}]}}
  ]
}'

# Read them back — note both share Name/Type, split by SetIdentifier + Weight.
awslocal route53 list-resource-record-sets --hosted-zone-id $ZONE \
  --query "ResourceRecordSets[?Name=='api.demo.internal.']"
```

This proves the *shape* of a multi-value routing policy (the `SetIdentifier` +
policy fields). LocalStack won't actually weight-resolve or run health checks —
that's the real-account part.

Teardown:

```bash
localstack stop
```

---

## Part C — Observe real resolution (real hosted zone, optional)

To *see* the policy work you need a hosted zone whose name servers you can query
directly (no real domain purchase needed — you can query the zone's NS set):

- [ ] `terraform apply` the Part A config on a real account (a public hosted
      zone; remember the **$0.50/mo zone + $0.50/mo per health check**).
- [ ] Find the zone's name servers:
      `aws route53 get-hosted-zone --id <id> --query DelegationSet.NameServers`.
- [ ] **Weighted:** query repeatedly and watch the ~90/10 split:
      `for i in $(seq 20); do dig +short @<ns> api.demo.internal A; done | sort | uniq -c`
- [ ] **Failover:** point the health check at a real endpoint, take it down,
      wait ~`failure_threshold * request_interval`, then query and confirm the
      answer flipped from the primary IP to the secondary.
- [ ] `terraform destroy` — deletes the zone **and** the billed health check.

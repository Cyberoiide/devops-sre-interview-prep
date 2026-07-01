# Solution — Securing the origin behind CloudFront

## The core insight

CloudFront is a **security boundary**, but only if the origin refuses everyone
who didn't come through it. A CDN in front of a *public* bucket is a cache with
a wide-open back door: the `*.s3.amazonaws.com` URL still serves objects, and
every protection you attached to the edge (WAF, geo-restriction, signed URLs,
logging) is bypassed. So security here is **layered**, and the layers answer
different questions:

| Layer | Question it answers | Mechanism |
|---|---|---|
| Origin lock-down | "Can anyone reach the origin *directly*?" | **OAC** (S3) / secret header + WAF (custom origin) |
| Request filtering | "Is this request *malicious*?" | **AWS WAF** web ACL |
| Access control | "Is this viewer *allowed*?" | **Geo-restriction**, **signed URLs/cookies** |
| Transport | "Is the connection *encrypted*?" | **TLS** (viewer + origin) |

## Layer 1 — Lock the origin to CloudFront only

### S3 origins: Origin Access Control (OAC)

The modern, recommended mechanism. Two moves:

1. **Make the bucket fully private.** Turn on **S3 Block Public Access** (all
   four flags) and remove any public bucket policy / object ACLs. Now nothing
   is reachable without authentication.
2. **Grant read to exactly your distribution.** Attach an **OAC** to the
   distribution's origin (`signing_behavior = "always"`, `signing_protocol =
   "sigv4"`). CloudFront then **SigV4-signs** every origin fetch. The bucket
   policy allows `s3:GetObject` to the `cloudfront.amazonaws.com` service
   principal, gated by a condition:

   ```json
   "Condition": { "StringEquals": { "AWS:SourceArn": "<distribution ARN>" } }
   ```

   That `AWS:SourceArn` pin is the crux — it ties access to **one specific
   distribution**, so another account's CloudFront distribution (or a raw
   public request) doesn't match and is denied.

Result: `https://<dist>.cloudfront.net/x` → 200; `https://<bucket>.s3.../x` →
403.

### OAC vs OAI — say why OAC wins

**OAI (Origin Access Identity)** was the old way: a special CloudFront identity
you granted bucket read to. It's **legacy / effectively deprecated** and AWS
recommends OAC for all new distributions because OAI:

- **doesn't support SSE-KMS** encrypted objects (OAC signs with SigV4, so it can
  fetch KMS-encrypted objects — OAI can't);
- **doesn't work in all AWS Regions** (notably Regions launched after 2022 /
  ones that require SigV4-only);
- **only supports `GET`/`HEAD`** and static signing — OAC supports dynamic
  requests including `PUT`/`POST` and per-request SigV4 signing;
- is a static principal rather than a per-request signed identity.

If asked "OAI or OAC?" the answer is **OAC**, and OAI only shows up on old
distributions you're maintaining.

### Custom (non-S3) origins: secret header + WAF

You can't attach an S3 bucket policy to an ALB, EC2, or an external HTTP origin.
Lock it down instead with a **shared secret**:

1. Configure CloudFront to **inject a custom header** on origin requests, e.g.
   `X-Origin-Verify: <long-random-secret>` (store the secret in Secrets Manager
   and rotate it).
2. **Reject requests that lack it.** Either a **WAF web ACL on the ALB** (a rule
   that blocks unless the header matches), or an origin-side check.
3. Additionally, **restrict the origin's security group / firewall to
   CloudFront's published IP ranges** (`AWS_IP_ADDRESS_RANGES`, or the managed
   `com.amazonaws.global.cloudfront.origin-facing` prefix list on the SG). IP
   ranges alone are weak (any CloudFront customer shares them), which is why the
   secret header is the primary control and IP restriction is defence in depth.

## Layer 2 — AWS WAF (L7 request filtering)

Attach a **WAF web ACL** to the distribution (`web_acl_id`). WAF inspects L7 and
blocks/counts requests before they reach the origin. Use the **AWS Managed Rule
groups** as a baseline rather than hand-writing signatures:

- `AWSManagedRulesCommonRuleSet` — common exploit patterns (bad inputs, some
  XSS/injection coverage).
- `AWSManagedRulesSQLiRuleSet` — SQL injection.
- `AWSManagedRulesKnownBadInputsRuleSet`, bot control, IP reputation lists.
- **Rate-based rules** — throttle a source IP over N requests / 5 min (cheap
  DDoS / brute-force mitigation).

WAF for a CloudFront distribution is **scope `CLOUDFRONT` and must be created in
`us-east-1`** (it's a global-edge resource) — a classic gotcha.

## Layer 3 — Access control

- **Geo-restriction.** CloudFront can **allow-list or block-list by country**
  (`restrictions.geo_restriction`) at the edge, e.g. only serve to `US/CA/GB`,
  or block sanctioned countries. It's coarse (country granularity, GeoIP-based)
  but free and enforced before the origin.
- **Signed URLs / signed cookies.** For **private or paid content**, CloudFront
  serves objects only if the request carries a valid, **time-limited signature**
  created with your key pair. Use **signed URLs** for individual files (a
  download link that expires) and **signed cookies** when you need to grant
  access to *many* files (a whole video library) without rewriting every URL.
  This is the CDN-level equivalent of an S3 presigned URL.

## Layer 4 — TLS everywhere

- **Viewer → CloudFront:** `viewer_protocol_policy = redirect-to-https` so plain
  HTTP is upgraded, and set a modern security policy
  (`minimum_protocol_version = TLSv1.2_2021`). For a custom domain, attach an
  **ACM certificate in `us-east-1`** (CloudFront only reads certs from there).
- **CloudFront → origin:** for custom origins use `origin_protocol_policy =
  https-only` so the back-half is encrypted too. (S3 origins are reached over
  HTTPS automatically.)

## Putting it together (the interview summary)

> "First I make the bucket private — Block Public Access on — and attach an
> **Origin Access Control** to the distribution, then write a bucket policy that
> allows read only to `cloudfront.amazonaws.com` with an `AWS:SourceArn`
> condition pinned to that one distribution. That's OAC, the modern replacement
> for OAI — OAI can't do SSE-KMS or all Regions. Now the origin only answers my
> CDN. On top of that I attach a **WAF** web ACL with the AWS managed SQLi/common
> rule groups and a rate-based rule, turn on **geo-restriction**, use **signed
> URLs/cookies** for any private content, and enforce **TLS 1.2+** viewer-side
> with `redirect-to-https`. For a non-S3 origin I'd swap OAC for a **secret
> custom header CloudFront injects plus a WAF rule** that drops requests without
> it, and restrict the origin SG to CloudFront's IP ranges as defence in depth."

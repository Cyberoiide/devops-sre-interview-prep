# Deeper — follow-up interview questions

### 1. OAC vs OAI — what specifically can OAC do that OAI can't?

OAI (Origin Access Identity) is the legacy mechanism and AWS now recommends OAC
for everything new. OAC adds: **SSE-KMS support** (OAC SigV4-signs the origin
request, so it can fetch KMS-encrypted objects; OAI can't decrypt them),
**support for all AWS Regions** (including SigV4-only Regions launched after
2022, where OAI simply doesn't work), **dynamic requests** including `PUT`/`POST`
uploads (OAI is `GET`/`HEAD` only), and **per-request SigV4 signing** rather than
a static identity. There's no feature OAI has that OAC lacks — the only reason to
touch OAI today is maintaining an old distribution.

### 2. How does the bucket policy actually pin access to ONE distribution?

Through the **`AWS:SourceArn` condition key**. The policy allows `s3:GetObject`
to the `cloudfront.amazonaws.com` service principal *only when*
`AWS:SourceArn` equals your distribution's ARN. CloudFront sends that ARN on the
signed origin request. A request from any other distribution — even in the same
account, even another AWS customer's — carries a different `SourceArn` and fails
the condition, so it's denied. Without that condition you'd be trusting the
CloudFront *service* globally, which any CloudFront customer could exploit to
read your bucket (the "confused deputy" problem). `AWS:SourceAccount` is a
coarser alternative that pins to your account rather than one distribution.

### 3. WAF is attached but requests still hit the origin — why, and where must a CloudFront WAF live?

Two common causes. First, a CloudFront web ACL **must be created with scope
`CLOUDFRONT` in `us-east-1`** — a regional WAF (scope `REGIONAL`, any other
Region) can't attach to a distribution. Second, managed rule groups default to
their own actions but you may have set the group to **Count** (monitoring) rather
than **Block**, so it logs matches without stopping them. Check the web ACL's
rule actions and the `sampled_requests` in CloudWatch. Also remember WAF only
sees what reaches CloudFront — it can't filter a request that went **directly to
a still-public origin**, which is why origin lock-down (OAC) comes first.

### 4. Signed URLs vs signed cookies — when each?

Both prove a request is authorized via a time-limited signature made with your
CloudFront key. Use a **signed URL** when you're granting access to a **single
file** or need a self-contained, shareable link (a one-off download that
expires). Use **signed cookies** when a viewer needs access to **many files**
(streaming a whole video catalog, a members-only section) without re-signing
every URL — you set the cookie once and the browser sends it on every subsequent
object request. Signed cookies also let you keep clean, un-signed-looking URLs.

### 5. Geo-restriction — what are its limits?

CloudFront geo-restriction is **country-level**, GeoIP-based, and enforced at the
edge before the origin. Limits: it can't do sub-country granularity (no
state/city), GeoIP is imperfect and trivially bypassed with a VPN/proxy, and it's
an access control, **not** a compliance guarantee. For finer or auditable
geo-logic you'd use WAF's geo-match statement (which can combine country with
other conditions), or handle it in a CloudFront Function / Lambda@Edge. Treat it
as coarse routing/blocking, not a legal boundary.

### 6. You have a non-S3 origin (an ALB). How do you stop people hitting the ALB directly?

You can't use OAC (that's S3-specific), so use a **shared-secret custom header**:
configure CloudFront to inject `X-Origin-Verify: <secret>` on origin requests,
and put a **WAF rule on the ALB** that blocks any request whose header doesn't
match (store/rotate the secret in Secrets Manager). Layer on an **ALB security
group that only allows CloudFront's IP ranges** (the managed
`com.amazonaws.global.cloudfront.origin-facing` prefix list). The header is the
real control — IP ranges are shared across all CloudFront customers, so alone
they'd let another customer's distribution through; they're defence in depth.

### 7. Where does TLS terminate, and what about the CloudFront→origin hop?

TLS terminates **at the CloudFront edge** for the viewer connection (viewer →
edge), where you enforce `redirect-to-https` and TLS 1.2+; a custom domain needs
an **ACM cert in `us-east-1`** because that's the only Region CloudFront reads
certs from. The **edge → origin** hop is a *separate* connection: for a custom
origin set `origin_protocol_policy = https-only` (and validate the origin cert)
so it's encrypted end to end; for an S3 origin CloudFront uses HTTPS
automatically. A common mistake is securing only the viewer side and leaving the
origin fetch on plain HTTP.

### 8. What does OAC *not* protect against?

OAC only controls **who can read the origin**. It does nothing about **request
volume/cost from legitimate-looking traffic** (you still want WAF rate rules +
CloudFront caching), **application-layer bugs** on a dynamic origin,
**data-at-rest** (that's SSE-S3/SSE-KMS on the bucket), or someone with **direct
AWS IAM access** to the bucket (that's IAM, not OAC). It also doesn't stop a
misconfiguration that re-opens Block Public Access — so pair it with an SCP or
Config rule that keeps public access blocked, and monitor for drift.

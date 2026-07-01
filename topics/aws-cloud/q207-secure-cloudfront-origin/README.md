# q207 — Securing the origin behind CloudFront

## The interview question

> "You put CloudFront in front of an S3 bucket to serve a static site. A
> colleague points out that the bucket's `*.s3.amazonaws.com` URL still works
> directly from the public internet, so people can bypass the CDN entirely.
> Walk me through how you lock the origin down so it *only* answers requests
> that came through your distribution — and while you're at it, what else would
> you put at the edge to protect this thing?"

This is a layered-security question. The interviewer wants to see that you
understand CloudFront is not just a cache — it's a security boundary — and that
"the origin is still publicly reachable" is a real, common misconfiguration.
The strongest answers cover **origin lock-down**, **request filtering (WAF)**,
**access control (geo, signed URLs)**, and **transport security (TLS)**, and
know *which mechanism solves which problem*.

## Why "just use CloudFront" isn't enough

Putting a CDN in front of a bucket does not remove the bucket's own public
endpoint. If the bucket is public (or the object ACLs are), anyone who guesses
or scrapes the origin URL can:

- **skip your cache**, spiking origin cost and load;
- **skip everything you bolted onto the edge** — WAF rules, geo-restriction,
  signed-URL checks, logging — because none of that lives on the origin.

So the first job is to make the origin refuse anyone who isn't your
distribution. For S3 that's **Origin Access Control (OAC)**; for a custom
(non-S3) origin it's a **shared secret header + a WAF rule**.

## What you should be able to answer

- **Origin lock-down for S3: Origin Access Control (OAC)** — the modern
  mechanism. The bucket is kept fully private (Block Public Access ON), and its
  bucket policy grants read **only** to your specific CloudFront distribution
  (via the `AWS:SourceArn` condition). Requests from CloudFront are SigV4-signed
  so S3 can verify them.
- **Why OAC over the legacy OAI (Origin Access Identity).** OAI is deprecated:
  it doesn't support SSE-KMS encrypted objects, all AWS Regions, or dynamic
  (SigV4) request signing. AWS explicitly recommends OAC for new work.
- **AWS WAF (a web ACL)** attached to the distribution — L7 filtering for SQLi,
  XSS, bad bots, rate-based rules, and the AWS Managed Rule groups.
- **Geo-restriction** — allow-list or block-list traffic by country at the edge.
- **Signed URLs / signed cookies** — for private/paid content, so only holders
  of a time-limited, signed token can fetch objects.
- **Custom (non-S3) origins:** you can't use an S3 bucket policy, so lock the
  origin down with a **secret custom header** injected by CloudFront plus a WAF
  rule (or origin-side check) that rejects requests lacking it, and restrict the
  origin's security group / firewall to CloudFront's published IP ranges.
- **TLS everywhere** — Viewer Protocol Policy `redirect-to-https`, a modern
  security policy (TLS 1.2+), and HTTPS on the origin fetch too.

## Files in this topic

- `exercise.md` — Terraform that defines a CloudFront distribution + private S3
  origin + OAC bucket policy + WAF + geo-restriction, validated with
  `terraform plan`; plus a LocalStack path for the S3/OAC bucket-policy mechanics
  and notes on what needs a real account.
- `solution.md` — the full layered answer: OAC vs OAI, WAF, geo, signed URLs,
  custom-origin secret header, TLS, and how they fit together.
- `deeper.md` — follow-up interview questions with concise answers.

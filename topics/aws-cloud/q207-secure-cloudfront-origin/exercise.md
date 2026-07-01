# Exercise — Lock an S3 origin to CloudFront with OAC

This lab defines the whole secure stack in **Terraform** — private S3 bucket,
CloudFront distribution, **Origin Access Control (OAC)**, the bucket policy that
only trusts that one distribution, a **WAF** web ACL, and **geo-restriction** —
and validates it with `terraform plan`. You author real, correct resources
without spending a cent. A second section drives the *bucket-policy mechanics*
against **LocalStack** so you can see the private-vs-public behaviour for real.

## What's honest about this lab

- **CloudFront itself is a global, live-account service.** You can `plan` the
  Terraform below with no credentials, and it's the exact config you'd `apply`,
  but a real `apply` needs an AWS account. CloudFront is **not** in LocalStack
  Community (it's a Pro feature), so we don't emulate the CDN edge.
- **The S3 side — Block Public Access, the OAC bucket policy, and proving a
  public request is denied — runs fully on LocalStack Community, for free.**
- If you want to see the *end-to-end* "direct bucket URL is now Forbidden"
  behaviour at the edge, you need a real account; the last section is a Free-Tier
  checklist for that.

## Prerequisites

- `terraform` >= 1.5 (or `tofu`) — for the `plan` part, no cloud needed.
- `docker` + [LocalStack](https://docs.localstack.dev/getting-started/) and the
  `awslocal` wrapper (`pip install awscli-local`) — for the S3 part.
- AWS CLI v2.

---

## Part A — Author the secure stack in Terraform (`plan`-only, no cloud)

Create a working dir and drop these files in.

`providers.tf`:

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

# CloudFront's WAF (scope=CLOUDFRONT) and ACM certs MUST live in us-east-1.
provider "aws" {
  region = "us-east-1"
}
```

`main.tf`:

```hcl
locals {
  bucket_name = "q207-secure-origin-demo"   # must be globally unique on real AWS
}

########################################
# 1. Private origin bucket
########################################
resource "aws_s3_bucket" "origin" {
  bucket = local.bucket_name
}

# Belt and braces: keep every public path shut.
resource "aws_s3_bucket_public_access_block" "origin" {
  bucket                  = aws_s3_bucket.origin.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

########################################
# 2. Origin Access Control (the modern OAI replacement)
########################################
resource "aws_cloudfront_origin_access_control" "oac" {
  name                              = "q207-oac"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"   # always SigV4-sign origin requests
  signing_protocol                  = "sigv4"
}

########################################
# 3. WAF web ACL (L7 filtering) — scope CLOUDFRONT, region us-east-1
########################################
resource "aws_wafv2_web_acl" "edge" {
  name  = "q207-edge-acl"
  scope = "CLOUDFRONT"

  default_action {
    allow {}
  }

  # AWS-managed baseline: common exploits (bad inputs, some SQLi/XSS coverage).
  rule {
    name     = "aws-common"
    priority = 1
    override_action {
      none {}
    }
    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "aws-common"
      sampled_requests_enabled   = true
    }
  }

  # Dedicated SQL-injection managed group.
  rule {
    name     = "aws-sqli"
    priority = 2
    override_action {
      none {}
    }
    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesSQLiRuleSet"
        vendor_name = "AWS"
      }
    }
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "aws-sqli"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "q207-edge-acl"
    sampled_requests_enabled   = true
  }
}

########################################
# 4. CloudFront distribution
########################################
resource "aws_cloudfront_distribution" "cdn" {
  enabled             = true
  default_root_object = "index.html"
  web_acl_id          = aws_wafv2_web_acl.edge.arn

  origin {
    domain_name              = aws_s3_bucket.origin.bucket_regional_domain_name
    origin_id                = "s3-origin"
    origin_access_control_id = aws_cloudfront_origin_access_control.oac.id
    # No custom_origin_config / no public bucket: OAC signs the request.
  }

  default_cache_behavior {
    target_origin_id       = "s3-origin"
    viewer_protocol_policy = "redirect-to-https"   # TLS enforced at the edge
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]

    # Managed "CachingOptimized" policy id (AWS-provided, stable well-known id).
    cache_policy_id = "658327ea-f89d-4fab-a63d-7e88639e58f6"
  }

  # Geo-restriction: allow-list a couple of countries as an example.
  restrictions {
    geo_restriction {
      restriction_type = "whitelist"
      locations        = ["US", "CA", "GB"]
    }
  }

  # Default CloudFront cert => TLS 1.2+ via the *.cloudfront.net domain.
  # For a custom domain you'd attach an ACM cert (us-east-1) and set
  # minimum_protocol_version = "TLSv1.2_2021".
  viewer_certificate {
    cloudfront_default_certificate = true
  }

  price_class = "PriceClass_100"
}

########################################
# 5. Bucket policy: trust ONLY this distribution (the whole point)
########################################
data "aws_iam_policy_document" "origin" {
  statement {
    sid     = "AllowCloudFrontOACRead"
    effect  = "Allow"
    actions = ["s3:GetObject"]
    resources = ["${aws_s3_bucket.origin.arn}/*"]

    principals {
      type        = "Service"
      identifiers = ["cloudfront.amazonaws.com"]
    }

    # This is what pins access to ONE distribution. A request from any other
    # CloudFront distribution (or the public internet) does not match => denied.
    condition {
      test     = "StringEquals"
      variable = "AWS:SourceArn"
      values   = [aws_cloudfront_distribution.cdn.arn]
    }
  }
}

resource "aws_s3_bucket_policy" "origin" {
  bucket = aws_s3_bucket.origin.id
  policy = data.aws_iam_policy_document.origin.json
}
```

Validate the definitions without touching a cloud:

```bash
terraform init          # downloads the AWS provider only
terraform validate      # schema/type check — should say "Success!"
terraform plan          # renders the full plan; needs no real creds for review
```

> `terraform plan` may try to reach AWS for the managed cache-policy data or to
> refresh state. If you have no creds, `terraform validate` alone still proves
> the config is well-formed. To force an offline render you can point at dummy
> creds (`AWS_ACCESS_KEY_ID=test AWS_SECRET_ACCESS_KEY=test`) and add
> `skip_credentials_validation = true` / `skip_requesting_account_id = true` to
> the provider — plan will still surface any resource-graph errors.

**Read the plan and confirm the security story:**

1. `aws_s3_bucket_public_access_block` sets all four flags `true` — the origin
   has no public door.
2. The bucket policy's only `Allow` is to `cloudfront.amazonaws.com` **and**
   gated by `AWS:SourceArn == <this distribution>`.
3. The distribution references the OAC (`origin_access_control_id`), not a
   `custom_origin_config` and not a legacy OAI.
4. `web_acl_id` wires the WAF in; `geo_restriction` is present;
   `viewer_protocol_policy = redirect-to-https`.

---

## Part B — See the private-origin mechanics on LocalStack (free)

CloudFront isn't in LocalStack Community, but the *origin* half is. This proves
the behaviour that matters: a public request to a locked bucket is denied, and
only a policy-matching principal gets in.

```bash
# Start LocalStack (S3 is Community).
localstack start -d
awslocal s3api create-bucket --bucket q207-origin

# Upload an object.
echo '<h1>secret origin content</h1>' > index.html
awslocal s3 cp index.html s3://q207-origin/index.html

# Turn on Block Public Access — the same posture Terraform sets.
awslocal s3api put-public-access-block \
  --bucket q207-origin \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# A public (unsigned) GET of the object URL should FAIL — the bucket is private.
curl -s -o /dev/null -w '%{http_code}\n' \
  http://localhost:4566/q207-origin/index.html      # expect 403 (AccessDenied)

# A signed CLI request (authenticated principal) still works.
awslocal s3 cp s3://q207-origin/index.html -        # prints the content
```

The public `curl` is 403 while the authenticated CLI read succeeds — that's the
exact split OAC relies on: strip the public path, then grant read to exactly one
signed principal (your distribution). Apply a bucket policy with an
`AWS:SourceArn` condition and you can watch a non-matching principal get denied
too:

```bash
awslocal s3api put-bucket-policy --bucket q207-origin --policy '{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "OnlyOneDistro",
    "Effect": "Allow",
    "Principal": {"Service": "cloudfront.amazonaws.com"},
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::q207-origin/*",
    "Condition": {"StringEquals": {"AWS:SourceArn": "arn:aws:cloudfront::000000000000:distribution/EXAMPLE"}}
  }]
}'
awslocal s3api get-bucket-policy --bucket q207-origin --query Policy --output text
```

Teardown:

```bash
awslocal s3 rb s3://q207-origin --force
localstack stop
```

---

## Part C — End-to-end at the edge (real Free-Tier account, optional)

To actually watch the direct bucket URL return **403** while the CloudFront URL
serves the file, you need a live account. Checklist:

- [ ] Set a **billing alarm** ($1) before you start.
- [ ] `terraform apply` the Part A config (CloudFront + S3 + OAC + WAF).
      CloudFront and S3 GETs are within the **Free Tier** at these volumes; a
      WAF web ACL has a small monthly + per-rule charge, so delete it promptly.
- [ ] `aws s3 cp index.html s3://<bucket>/index.html` to seed content.
- [ ] Wait for the distribution to finish deploying
      (`aws cloudfront get-distribution --id <id> --query Distribution.Status`).
- [ ] `curl https://<dist>.cloudfront.net/index.html` → **200**, serves the file.
- [ ] `curl https://<bucket>.s3.amazonaws.com/index.html` → **403 AccessDenied**
      (the origin now refuses anyone but the distribution). This is the payoff.
- [ ] Try a request with a SQLi-looking query string and confirm WAF blocks /
      counts it (CloudWatch `sampled_requests`).
- [ ] `terraform destroy` — **especially delete the WAF web ACL and the
      distribution** so you stop accruing charges.

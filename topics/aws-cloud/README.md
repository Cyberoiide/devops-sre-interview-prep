# AWS / Cloud

Interview-prep study material for DevOps/SRE roles, focused on core AWS
services you get grilled on in system-design and troubleshooting rounds:
CloudFront/edge security, VPC network design, Route 53 DNS routing, and Lambda
performance. Each question is a scenario an interviewer would actually pose,
with a hands-on lab attached.

> **A note on labs.** AWS is a paid, credentialed platform, so unlike the
> Terraform topic these labs can't all run "for free on a laptop." Each
> `exercise.md` is explicit about what it needs and gives a no-cost path where
> one exists: **[LocalStack](https://www.localstack.dev/)** (a local AWS API
> emulator) for the CLI-driven parts, **Terraform in `plan`-only / validate
> mode** so you can author real resource definitions without applying them, or
> a **guided console walkthrough + checklist** where emulation isn't faithful.
> Anything that genuinely needs a live account is flagged, and stays inside the
> **Free Tier** where possible.

## Questions

| ID | Topic | One-line summary |
|----|-------|------------------|
| [q207](./q207-secure-cloudfront-origin/) | Secure CloudFront origin | The world can reach your S3 bucket directly, bypassing the CDN — how do you lock the origin to CloudFront only, and what else guards the edge? |
| [q203](./q203-secure-vpc/) | Secure VPC design | You're handed a blank account and told "build us a secure VPC" — what are the non-negotiable design decisions? |
| [q199](./q199-route53-routing-policies/) | Route 53 routing policies | A user asks for "DNS-based failover and geo-routing" — which of Route 53's seven routing policies do you reach for, and how do they actually differ? |
| [q195](./q195-lambda-cold-starts/) | Lambda cold starts | P99 latency spikes to seconds on a Lambda that's usually fast — what's a cold start, and how do you engineer them away? |

## Folder layout

Each question folder contains exactly four files:

- **README.md** — the scenario, phrased the way an interviewer would pose it.
- **exercise.md** — a hands-on lab with concrete, runnable commands (LocalStack,
  Terraform, CLI) or a guided walkthrough + checklist where a live account is
  needed.
- **solution.md** — the detailed written answer and walkthrough.
- **deeper.md** — likely interviewer follow-up questions with concise answers.

## How to use this repo

1. Read the question's `README.md` and try to answer it out loud (whiteboard style).
2. Do the lab in `exercise.md` — actually run the commands, don't just read them.
3. Compare your reasoning against `solution.md`.
4. Drill the follow-ups in `deeper.md` until the answers are reflexive.

## Prerequisites

- The **AWS CLI v2** installed (`aws --version`) — used against LocalStack or a
  real account.
- **[LocalStack](https://docs.localstack.dev/getting-started/)** (Docker-based)
  for the emulated labs. The free Community edition covers most of what these
  exercises need; a couple of features (CloudFront, Lambda Provisioned
  Concurrency) are Pro-only or not emulated and are called out in place.
- **`terraform` >= 1.5** (or `tofu` >= 1.6) for the resource-definition labs.
  You can `init`/`validate`/`plan` many of these with no cloud at all.
- Optional: a personal AWS account on the **Free Tier** for the parts that need
  real edge/DNS behaviour. Set a billing alarm first, and tear everything down
  when you're done — the labs tell you how.

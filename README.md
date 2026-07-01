# DevOps / SRE / Platform Engineering — Interview Prep Memo

A structured, hands-on study repo built from real interview questions. Each
question gets a scenario, a hands-on exercise you can actually run, a detailed
solution, and "go deeper" notes for the follow-up questions an interviewer
will throw at you.

> Source material: DevOps/SRE practice-interview YouTube playlists (questions
> #194–#226). Questions have been expanded into full labs — the videos are
> short, this repo is not.

## How to use this

1. Read the question in each folder's `README.md`. **Try to answer it out loud first.**
2. Do the `exercise.md` — hands-on beats reading every time.
3. Check yourself against `solution.md`.
4. Skim `deeper.md` for the follow-ups.

## Topics

| Topic | What's inside |
|-------|---------------|
| [Kubernetes](topics/kubernetes/) | Failed rollouts, CFS throttling, pending pods, ingress errors, ingress vs LB, node-pool drains, DaemonSets, custom schedulers, api-resources |
| [System Design](topics/system-design/) | Thundering herd, leaderboards, race conditions, latency numbers, Redis vs Memcached, Big-O, SSL/TLS termination |
| [AWS / Cloud](topics/aws-cloud/) | Securing CloudFront origins, VPC hardening, Route 53 routing policies, Lambda cold starts |
| [Databases](topics/databases/) | Schema design — normalization vs JOIN cost, data types, foreign keys, constraints, indexing |
| [Terraform / IaC](topics/terraform-iac/) | State recovery/list/move/remove, taint vs -replace, outputs, upgrades, VM→K8s DB migration, phantom rebuilds |
| [CI/CD & Linux](topics/cicd-linux/) | Pipelines failing on large PRs, CLOSE_WAIT connections, copy-on-write |
| [Career](topics/career/) | Top 5 topics to master (from real job-description analysis) |

## The 5 must-master topics (2026)

Derived from 50+ LinkedIn DevOps/SRE job descriptions by word frequency:

1. **Cloud** — AWS / Azure / GCP (be proficient in at least one)
2. **CI/CD pipelines** — GitHub Actions, Jenkins, or equivalent
3. **Monitoring & observability** — Prometheus, Grafana
4. **Infrastructure as Code** — Terraform / OpenTofu
5. **Kubernetes** — the de-facto infra layer

## Prerequisites for the labs

Most exercises assume some of:
- `docker` + `kind` or `minikube` (local Kubernetes)
- `kubectl`
- `terraform` or `tofu`
- `redis` (via Docker)
- A Linux shell

Each exercise lists exactly what it needs.

## License

MIT — study, fork, share.

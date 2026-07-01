# Exercise: Self-Assessment + Study-Plan Builder

The goal of this exercise is to turn the top-5 list into a personal, prioritized
study plan. You will (1) rate yourself honestly on each topic, (2) find your
weakest topics, and (3) commit to one hands-on project per gap. Hands-on beats
reading every time — an interviewer can tell within two questions whether you
have actually built something or only watched a course.

Optionally, before you start, reproduce the ranking for *your* target market so
you are studying the right things.

---

## Part 0 (optional): Reproduce the ranking for your market

Collect 20–30+ job descriptions for the exact role, seniority, and region you are
targeting. Paste the plain text of each into a single file, one after another,
called `postings.txt`. Then run a quick frequency pass:

```bash
# ponytail: crude but honest — lowercases, splits on non-letters,
# keeps 3+ letter words so short but important terms (aws, gcp, ci)
# survive, drops common words, shows the top 40 terms.
tr '[:upper:]' '[:lower:]' < postings.txt \
  | tr -c 'a-z' '\n' \
  | grep -E '.{3,}' \
  | grep -vwE 'the|and|for|with|will|team|work|role|have|this|that|your|from|were|been|help|able|such|into|more|also|must|are|our|you' \
  | sort | uniq -c | sort -rn \
  | head -40
```

This won't group synonyms (you'll see `kubernetes` and `k8s` separately, and
multi-word tools like "github actions" get split, so `github` and `actions` are
counted apart) — so read the raw output with judgment and tally canonical tools
by hand. It's enough to reveal whether your
market skews AWS vs Azure, Terraform vs CloudFormation, etc. Adjust your priority
order accordingly before continuing.

---

## Part 1: Rate yourself (1–5) on each topic

For each topic, circle the number that honestly matches you. Be strict —
"I watched a tutorial" is a 2, not a 4. The scale:

- **1 — None.** Never touched it. Can't define the core terms.
- **2 — Aware.** Read/watched about it. Could pass a definition quiz, but have not
  built anything.
- **3 — Tinkered.** Followed a tutorial or did a toy project. Would struggle to
  build from scratch without hand-holding.
- **4 — Proficient.** Built something real and non-trivial on my own. Can explain
  my choices and debug common problems.
- **5 — Owned in production (or equivalent).** Ran it where failure had real
  consequences. Can teach it and reason about trade-offs and failure modes.

> **The honesty test for a 4+:** For any topic you rate 4 or 5, you must be able
> to finish this sentence out loud with a specific, true story: *"One time I
> ______, and when ______ went wrong, I ______."* If you can't, drop your rating
> by one. That story is exactly what the interviewer is fishing for.

### Scorecard

| # | Topic | 1 | 2 | 3 | 4 | 5 |
|---|-------|---|---|---|---|---|
| 1 | Cloud (AWS / Azure / GCP) | ☐ | ☐ | ☐ | ☐ | ☐ |
| 2 | CI/CD pipelines | ☐ | ☐ | ☐ | ☐ | ☐ |
| 3 | Monitoring & observability | ☐ | ☐ | ☐ | ☐ | ☐ |
| 4 | Infrastructure as Code | ☐ | ☐ | ☐ | ☐ | ☐ |
| 5 | Kubernetes | ☐ | ☐ | ☐ | ☐ | ☐ |

---

## Part 2: Per-topic checklists

Tick each item you can do *unassisted, right now*. Your real level for a topic is
roughly the highest tier where you can tick every box. If you rated yourself 4 in
Part 1 but can't tick the "Proficient" boxes below, trust the checklist.

### Topic 1 — Cloud

**Aware (2)**
- [ ] I can name the core service categories (compute, storage, networking, IAM).
- [ ] I know the difference between a region and an availability zone.

**Tinkered (3)**
- [ ] I have launched a VM (EC2 / Azure VM / GCE) and SSH'd into it.
- [ ] I have created a storage bucket and set its access permissions.
- [ ] I have created an IAM user/role and attached a policy.

**Proficient (4)**
- [ ] I can set up a VPC/VNet with public and private subnets and explain the routing.
- [ ] I can explain least-privilege IAM and write a scoped policy.
- [ ] I have deployed an app reachable from the internet through a load balancer.
- [ ] I can read a cloud bill and name the two or three biggest cost drivers.

**Owned (5)**
- [ ] I have designed for high availability across multiple AZs.
- [ ] I have debugged a production networking/permissions issue under pressure.

### Topic 2 — CI/CD pipelines

**Aware (2)**
- [ ] I can explain what CI and CD each mean and how they differ.
- [ ] I can read a pipeline YAML file and describe what it does.

**Tinkered (3)**
- [ ] I have written a pipeline that builds and tests a project on every push.
- [ ] I have made a pipeline fail on purpose and read the logs to fix it.

**Proficient (4)**
- [ ] I have designed a multi-stage pipeline (build → test → deploy).
- [ ] I handle secrets correctly (no plaintext credentials in the repo).
- [ ] I have built and pushed a container image from a pipeline.
- [ ] I can explain how I would roll back a bad deploy.

**Owned (5)**
- [ ] I have set up a deployment gate/approval or an automated rollback.
- [ ] I have optimized a slow pipeline (caching, parallelism) and measured the win.

### Topic 3 — Monitoring & observability

**Aware (2)**
- [ ] I can explain metrics vs logs vs traces.
- [ ] I know what an alert and an SLO/SLI are.

**Tinkered (3)**
- [ ] I have run Prometheus and scraped metrics from a target.
- [ ] I have opened Grafana and viewed a metric on a graph.

**Proficient (4)**
- [ ] **I have built** a Grafana dashboard for a service/infra I ran myself.
- [ ] I have written a Prometheus alerting rule that fires on a real condition.
- [ ] I can name the four golden signals (or USE/RED) and what each tells me.
- [ ] I have instrumented an app to expose a custom metric.

**Owned (5)**
- [ ] I have defined SLOs and an error budget for a service.
- [ ] An alert I wrote caught a real incident, and I can tell that story.

### Topic 4 — Infrastructure as Code

**Aware (2)**
- [ ] I can explain what IaC is and why it beats clicking in a console.
- [ ] I know what `plan` and `apply` do.

**Tinkered (3)**
- [ ] I have written Terraform that provisions at least one real resource.
- [ ] I understand what the state file is and why it must not be lost.

**Proficient (4)**
- [ ] I have written a reusable module with variables and outputs.
- [ ] I use remote state with locking (e.g. S3 + DynamoDB, or a backend).
- [ ] I can explain the Terraform/BSL licensing change and what OpenTofu is.
- [ ] I have reviewed a `terraform plan` diff and caught a destructive change.

**Owned (5)**
- [ ] I manage multiple environments (dev/staging/prod) from the same code.
- [ ] I have run IaC in CI with policy checks and imported existing infra into state.

### Topic 5 — Kubernetes

**Aware (2)**
- [ ] I can explain Pod, Deployment, Service, and Namespace.
- [ ] I know the difference between a container and a pod.

**Tinkered (3)**
- [ ] I have deployed an app to a cluster (kind/minikube/managed) with `kubectl apply`.
- [ ] I have used `kubectl` to inspect logs and describe a resource.

**Proficient (4)**
- [ ] I have debugged a `CrashLoopBackOff` / `ImagePullBackOff` from first principles.
- [ ] I can write a Deployment + Service + Ingress from scratch.
- [ ] I understand requests/limits and what happens when a pod exceeds them.
- [ ] I have used ConfigMaps/Secrets and liveness/readiness probes.

**Owned (5)**
- [ ] I have packaged an app with Helm or deployed via GitOps (Argo CD / Flux).
- [ ] I have diagnosed a node/scheduling/networking problem in a real cluster.

---

## Part 3: Build your study plan

1. **Rank your topics by gap.** Copy your Part 1 scores into the table below. The
   gap column is `5 − your score`. Sort by biggest gap first — but if two topics
   tie, break the tie using the priority order (Cloud > CI/CD > Observability >
   IaC > Kubernetes), since earlier topics are more often screening filters.

   | Topic | Your score | Gap (5 − score) | Priority rank |
   |-------|-----------|-----------------|---------------|
   | Cloud | | | |
   | CI/CD | | | |
   | Observability | | | |
   | IaC | | | |
   | Kubernetes | | | |

2. **Pick your top 2 gaps.** Don't try to fix all five at once — you'll do a shallow
   job of everything. Fix the two biggest gaps first, then re-assess.

3. **Commit to the project for each gap** (below). Give yourself a deadline and a
   definition of done. The definition of done is: *you can demo it and tell the
   story from the honesty test in Part 1.*

---

## Suggested hands-on project per topic

Each project is scoped to be doable on a personal budget (free tiers / local
tooling) and produces something you can put in a portfolio and talk about in an
interview. They deliberately build on each other — do them in order and you end
up with one connected system (see the diagram in `README.md`).

### Project 1 — Cloud: "Public app on one provider"
Pick **one** provider (AWS unless your market says otherwise). Manually — via the
console first, to learn the primitives — deploy a small web app so that:
a VPC/VNet with a public and private subnet exists; the app runs on compute in the
private subnet; a load balancer in the public subnet fronts it; and access is
governed by a least-privilege IAM role, not root credentials.
**Done when:** you can hit the app from your browser and draw the network diagram
from memory, naming every hop.

### Project 2 — CI/CD: "Push to deploy"
Take the app from Project 1 and put a pipeline in front of it (GitHub Actions is
the easiest starting point). On every push: run tests, build a container image,
push it to a registry, and deploy the new version. Add a manual approval gate
before the deploy step.
**Done when:** a `git push` results in a running new version with zero manual
console steps, and you can explain how you'd roll back.

### Project 3 — Observability: "Dashboards for infra I own"
Stand up Prometheus and Grafana against the app from Projects 1–2. Expose at least
one custom application metric, build a dashboard covering the four golden signals,
and write one alerting rule (e.g. error rate > threshold for 5 minutes). Then
*break the app on purpose* and watch the alert fire.
**Done when:** you have a dashboard screenshot and a true story: "I built this,
and when I injected errors, my alert fired." Say **built**, not "used".

### Project 4 — IaC: "Terraform the whole thing"
Go back and rewrite Projects 1–3's infrastructure as Terraform (or OpenTofu). Use
remote state with locking, split reusable pieces into a module, and parameterize
so you can spin up a `dev` and a `staging` environment from the same code.
**Done when:** you can `terraform destroy` and `terraform apply` to recreate the
entire stack from nothing, and you can explain the state-locking setup.

### Project 5 — Kubernetes: "Run it on a cluster"
Deploy the app to Kubernetes — a local `kind`/`minikube` cluster to learn, then a
managed cluster (EKS/AKS/GKE) if budget allows. Write Deployment + Service +
Ingress, add liveness/readiness probes and resource requests/limits, and store
config in ConfigMaps/Secrets. Bonus: package it with Helm or deploy via Argo CD.
**Done when:** you can deliberately cause a `CrashLoopBackOff`, then diagnose and
fix it while narrating what you're checking and why.

> **The capstone.** If you finish all five, you have a single end-to-end project:
> Terraform provisions a Kubernetes cluster on a cloud provider, a CI/CD pipeline
> deploys your app to it, and Grafana dashboards watch it run. That one repo lets
> you answer nearly any DevOps/SRE screening question with "let me show you what I
> built." Pin it on your GitHub.

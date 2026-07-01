# Solution: Detailed Study Roadmap for the Top 5 Topics

This is the "answer key" for the self-assessment — a study roadmap for each of
the five topics. For every topic you'll find:

- **Key subtopics** — what to actually learn, roughly in order.
- **Canonical tools** — what the industry standardizes on (learn these first).
- **What "proficient" looks like** — the bar an interviewer is checking for.
- **Resources & project ideas** — where to learn it and what to build.

Work depth-first on your weakest topics (from the exercise), not breadth-first
across all five. Depth in two topics beats a shallow tour of five.

---

## Topic 1 — Cloud (AWS / Azure / GCP)

**Pick one provider and go deep.** Breadth across three clouds is worth less than
real depth in one — the concepts transfer, and interviewers respect depth. Choose
**AWS** unless your target market clearly skews Azure or GCP (check with the
exercise's frequency script). See `deeper.md` for the full "which cloud" argument.

### Key subtopics
1. **The mental model:** regions vs availability zones; the shared responsibility
   model; the global-vs-regional-vs-zonal nature of services.
2. **Compute:** VMs (EC2 / Azure VM / GCE), containers (ECS/Fargate, or the
   managed Kubernetes services), and serverless (Lambda / Functions).
3. **Networking:** VPC/VNet, subnets (public vs private), route tables, internet
   and NAT gateways, security groups vs NACLs, load balancers, DNS.
4. **Identity & access (IAM):** users, roles, policies, and — critically —
   least privilege. This is the #1 source of both interview questions and real
   security incidents.
5. **Storage:** object storage (S3 / Blob / GCS), block storage (EBS), and when
   to use which; storage classes and lifecycle.
6. **Managed data:** at least conceptually — managed databases (RDS), caching,
   and queues.
7. **Cost & billing:** how you get charged, how to read a bill, and the obvious
   cost traps (idle resources, egress, over-provisioning).

### Canonical tools
AWS (EC2, S3, VPC, IAM, RDS, CloudWatch, EKS) — the most in-demand. Azure and GCP
are the strong alternatives. The provider CLI (`aws` / `az` / `gcloud`) and its
web console.

### What "proficient" looks like
You can design and diagram a basic highly-available architecture: an app running
across two AZs, in private subnets, behind a load balancer, with a scoped IAM
role instead of long-lived root keys, and you can explain every arrow in the
diagram. You can debug "why can't this service reach that one" by reasoning
through security groups, routing, and IAM — the three usual culprits.

### Resources & project ideas
- **Certification as a syllabus (not a goal):** the AWS Solutions Architect
  Associate (or Azure AZ-104 / Google Associate Cloud Engineer) blueprint is an
  excellent, well-ordered checklist of what to learn. The cert itself is a nice
  resume signal, but the *knowledge* is the point.
- **Free learning:** the provider's own free tier + official tutorials; freeCodeCamp
  and similar full-length cloud courses on YouTube.
- **Project:** Exercise Project 1 — a public web app with a proper VPC, private
  compute, a load balancer, and least-privilege IAM. Then tear it down so you
  stop paying for it.

---

## Topic 2 — CI/CD pipelines

CI/CD is the closest thing to the *daily work* of the role, so interviewers probe
it hard. The tool matters less than the concepts: if you understand one pipeline
system deeply, you can pick up any other in a day.

### Key subtopics
1. **The distinction:** Continuous Integration (merge + build + test often) vs
   Continuous Delivery vs Continuous Deployment (auto-release to prod).
2. **Pipeline anatomy:** triggers (push, PR, tag, schedule, manual), stages,
   jobs, steps, and artifacts passing between them.
3. **Testing in the pipeline:** unit → integration → end-to-end; failing fast;
   quality gates.
4. **Building & shipping artifacts:** building container images, tagging them
   sensibly, pushing to a registry.
5. **Secrets management:** never in the repo — use the platform's secret store /
   OIDC federation to the cloud. This is a very common interview question.
6. **Deployment strategies:** rolling, blue-green, canary; approval gates; and —
   the question everyone asks — **how you roll back**.
7. **Pipeline hygiene:** caching and parallelism for speed; keeping pipelines
   as code in the repo.

### Canonical tools
**GitHub Actions** (easiest entry, ubiquitous), **GitLab CI**, and **Jenkins**
(still everywhere in enterprises — worth being conversant even if you learn
Actions first). Argo CD / Flux for the GitOps flavor of CD (overlaps with
Kubernetes).

### What "proficient" looks like
You can design a multi-stage pipeline from a blank file: it triggers on push,
runs tests, builds and pushes an image, and deploys — with secrets handled safely
and a clear rollback story. You can look at a slow pipeline and speed it up with
caching/parallelism, and explain the trade-off of an approval gate (safety vs
speed).

### Resources & project ideas
- **Docs first:** the GitHub Actions docs and starter workflows are genuinely
  good; GitLab CI docs likewise.
- **Project:** Exercise Project 2 — a "push to deploy" pipeline in front of your
  cloud app, with a manual approval gate. Deliberately break a test and watch the
  pipeline stop the deploy.
- **Talking point to prepare:** be ready to whiteboard a pipeline and answer
  "a bad version just went live — walk me through what happens next."

---

## Topic 3 — Monitoring & observability

This is the topic that most cleanly separates "I ran some servers" from "I owned
reliability." The single highest-leverage thing you can do here: **be able to say
you BUILT dashboards to monitor infrastructure you owned** — and back it with a
real story. Interviewers weight lived experience here heavily.

### Key subtopics
1. **The three pillars:** metrics, logs, and traces — what each is for and when
   you reach for which.
2. **Metrics deep-dive:** the Prometheus model (pull-based scraping, time series,
   labels), PromQL basics, exporters, and instrumenting your own app.
3. **Dashboards:** building Grafana dashboards that answer real questions, not
   vanity graphs. Learn the **four golden signals** (latency, traffic, errors,
   saturation) and/or **RED** (Rate, Errors, Duration) / **USE** (Utilization,
   Saturation, Errors) as your dashboard skeleton.
4. **Alerting:** writing alert rules that fire on symptoms users feel, not noise;
   avoiding alert fatigue; routing (Alertmanager → PagerDuty/Opsgenie/Slack).
5. **SLIs, SLOs, and error budgets:** the SRE vocabulary. Even a conceptual grasp
   sets you apart, especially for SRE-titled roles.
6. **Logs & traces:** centralized logging (Loki / ELK / cloud-native) and
   distributed tracing (OpenTelemetry, Jaeger/Tempo) — at least conceptually.

### Canonical tools
**Prometheus** + **Grafana** are the lingua franca — learn these first.
Alertmanager for routing. Then, as breadth: Loki/ELK (logs), OpenTelemetry +
Jaeger/Tempo (traces), and the cloud-native options (CloudWatch, Azure Monitor,
Google Cloud Operations). Datadog/New Relic as commercial all-in-ones.

### What "proficient" looks like
You have built a Grafana dashboard for something you ran, structured around the
golden signals, and written at least one alert that fired on a real condition.
You can explain the difference between a good alert (a human needs to act now) and
a bad one (noise), and you can sketch what you'd monitor for a given service and
why.

### Resources & project ideas
- **Docs & books:** the Prometheus and Grafana docs; the Google SRE book (free
  online) for the SLO/error-budget vocabulary.
- **Project:** Exercise Project 3 — Prometheus + Grafana against your app, a
  golden-signals dashboard, one alert rule, then break the app and watch it fire.
- **The story to rehearse:** "I built the dashboards and alerts for the service my
  team owned. One time, an alert I'd written for elevated error rate fired before
  any customer complained — turned out to be X — and because we caught it early we
  Y." A concrete, true version of this is worth more than any cert.

---

## Topic 4 — Infrastructure as Code

IaC is *how modern infrastructure is actually built and changed* — declaratively,
in version control, reviewed like application code. **Terraform is now the #1
tool** in the space and the one to learn.

> **The licensing shift (know this — it's a current-events interview question).**
> In August 2023, HashiCorp changed Terraform's license from the open-source
> MPL 2.0 to the **Business Source License (BSL 1.1)**, which restricts using
> Terraform to compete commercially with HashiCorp. In response, the community
> forked the last MPL-licensed version and created **OpenTofu** (now under the
> Linux Foundation) — a drop-in, open-source alternative. HashiCorp was
> subsequently acquired by IBM (2024). Practically: the two are still largely
> compatible, most jobs still say "Terraform," and knowing the language and the
> workflow transfers directly to OpenTofu. Being able to explain this story
> signals you follow the ecosystem.

### Key subtopics
1. **Declarative model:** you describe the desired end state; the tool figures out
   the changes. Contrast with imperative scripting.
2. **The core workflow:** `init` → `plan` → `apply` (→ `destroy`), and *reading a
   plan diff carefully* — especially spotting a destroy/replace before you run it.
3. **State:** what the state file is, why it's the source of truth, why losing or
   corrupting it is a disaster, and **remote state with locking** (S3 + DynamoDB,
   Terraform Cloud, GitLab, etc.) so a team doesn't clobber each other.
4. **Language building blocks:** resources, variables, outputs, data sources,
   providers, `for_each`/`count`, and expressions.
5. **Modules:** factoring reusable infrastructure into modules with clean inputs
   and outputs — the key to not repeating yourself.
6. **Environments:** managing dev/staging/prod from one codebase (workspaces or,
   more commonly, directory-per-env layouts).
7. **IaC in CI + policy as code:** running `plan` on PRs, `apply` on merge, and
   guardrails (OPA/Sentinel, `tflint`, `checkov`).
8. **Adjacent tools (conceptual breadth):** configuration management (Ansible) vs
   provisioning (Terraform); cloud-native IaC (CloudFormation, Pulumi, CDK).

### Canonical tools
**Terraform** (and its twin **OpenTofu**) first. Then, for breadth: Ansible
(config management), Pulumi/CDK (IaC in general-purpose languages),
CloudFormation (AWS-native). `tflint`/`checkov`/OPA for policy.

### What "proficient" looks like
You can write a reusable module, wire up remote state with locking, and stand up
multiple environments from one codebase. You read a `plan` and can point at the
line that would destroy production. You can explain the BSL/OpenTofu situation in
two sentences. You treat infra changes like code changes — reviewed, versioned,
and never applied blindly.

### Resources & project ideas
- **Docs & learning:** the official Terraform/OpenTofu tutorials (HashiCorp Learn
  is excellent and hands-on); the Terraform Associate cert as a syllabus.
- **Project:** Exercise Project 4 — rewrite your cloud stack as Terraform with
  remote state, a module, and dev + staging environments. Prove it by running
  `destroy` then `apply` to recreate everything from nothing.

---

## Topic 5 — Kubernetes

Kubernetes is still hot and has become the **de-facto base infrastructure layer**
for DevOps. It's increasingly *assumed* at mid-level and above, and even junior
roles now expect awareness. It's the deepest topic here — don't try to boil the
ocean; get solid on the core objects and on *debugging*, which is what the job
actually involves.

### Key subtopics
1. **Why it exists:** the problem of running many containers across many machines
   — scheduling, self-healing, scaling, service discovery.
2. **Core objects:** Pod, ReplicaSet, **Deployment**, **Service** (ClusterIP /
   NodePort / LoadBalancer), **Ingress**, Namespace. Know how they relate.
3. **Configuration:** ConfigMaps and Secrets; environment injection.
4. **Health & resources:** liveness/readiness/startup probes; resource
   **requests and limits** and what OOMKilled / throttling mean.
5. **Storage & stateful workloads:** PersistentVolumes/Claims, StatefulSets (at
   least conceptually).
6. **Debugging (the money skill):** `kubectl get/describe/logs/exec`, and reading
   the common failure states from first principles — `CrashLoopBackOff`,
   `ImagePullBackOff`, `Pending` (unschedulable), `OOMKilled`. Interviewers love
   "a pod is in CrashLoopBackOff — what do you check?"
7. **Scheduling basics:** how the scheduler places pods; requests, node capacity,
   taints/tolerations, affinity (conceptually).
8. **Packaging & delivery:** **Helm** (templated, versioned releases) and
   **GitOps** with Argo CD / Flux (the cluster syncs to a git repo). This ties
   back to CI/CD.
9. **Managed vs self-hosted:** EKS / AKS / GKE do the hard control-plane work —
   know that you'll almost always use a managed cluster in practice.

### Canonical tools
**kubectl** and a local cluster (**kind** or **minikube**) to learn on; a
managed cluster (**EKS/AKS/GKE**) for the real thing. **Helm** for packaging.
**Argo CD / Flux** for GitOps. `k9s`/Lens as quality-of-life TUIs.

### What "proficient" looks like
You can write a Deployment + Service + Ingress from scratch, set probes and
resource limits, and — most importantly — **debug a broken pod methodically**,
narrating what you check and why (describe the pod, read events, check logs, check
image/config). You understand you'll use a managed cluster and can explain what
the control plane does for you. Bonus points for Helm and a GitOps flow.

### Resources & project ideas
- **Learning:** the official Kubernetes docs and interactive tutorials;
  "Kubernetes the Hard Way" (build a cluster by hand — deep but illuminating);
  the CKA/CKAD certs as a rigorous, *hands-on* syllabus (these exams are
  practical, not multiple-choice — good signal).
- **Project:** Exercise Project 5 — deploy your app to `kind` locally, then a
  managed cluster; add probes, limits, ConfigMaps/Secrets; package with Helm or
  deploy via Argo CD. Deliberately cause a `CrashLoopBackOff` and fix it out loud.

---

## Putting it together

Do the five exercise projects in order and they compose into one end-to-end
system — the capstone described in `README.md` and `exercise.md`. That single
repository is the most efficient interview asset you can build: it lets you answer
almost any screening question with a concrete "here's what I built and why."
Pin it on GitHub, write a short README that tells the story, and rehearse walking
through it in five minutes.

For the broader career questions — which cloud to commit to, how to present all
this in a portfolio and resume, and what interview formats to expect — see
[`deeper.md`](./deeper.md).

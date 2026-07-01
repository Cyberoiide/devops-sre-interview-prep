# Top 5 Topics to Master for DevOps/SRE Interviews (2026)

## The premise

Job descriptions are noisy. Every DevOps/SRE posting lists a wall of tools, but
if you read enough of them, the same handful of skills keep surfacing as
must-haves rather than nice-to-haves. To cut through the noise, the approach
here is simple and repeatable: collect a large sample of real postings, strip
them down to word/phrase frequency, and let the data tell you where to spend
your limited study time.

For this module, the sample was 50+ DevOps and SRE job descriptions pulled from
LinkedIn (a mix of junior, mid, and senior roles, across startups and larger
companies). Each description was normalized (lowercased, tools grouped under a
canonical name — e.g. "GH Actions", "GitHub Actions", and "Actions" all count as
GitHub Actions) and counted by how many *distinct postings* mentioned each
concept. Counting distinct postings rather than raw word count matters: it stops
one verbose posting from skewing the ranking, and it answers the real question —
"what fraction of jobs expect this?"

> **Reproduce it yourself.** This is not a magic list. Scrape or hand-collect 30+
> postings for the exact role/region/seniority you are targeting, paste them into
> a single file, and run a word-frequency pass (even a quick `tr`/`sort`/`uniq`
> pipeline or a spreadsheet works). Your local market may weight things
> differently — Azure dominates some regions, GCP others. See
> [`exercise.md`](./exercise.md) for a starter script. Treat the ranking below as
> a strong default, not gospel.

## The result: the top 5 topics, in priority order

The five topics below appeared in the overwhelming majority of postings and are
ordered by how frequently they were required *and* how often they were treated
as a screening filter (i.e. "you will not get past the phone screen without
this").

| # | Topic | Appears in ~ | Why it ranks here |
|---|-------|-------------|-------------------|
| 1 | **Cloud** (AWS / Azure / GCP) | Nearly every posting | The platform everything else runs on. Proficiency in *at least one* is table stakes. AWS is the single most in-demand. |
| 2 | **CI/CD pipelines** | Nearly every posting | The core daily-work skill of the role — automating build, test, and deploy. GitHub Actions, Jenkins, GitLab CI. |
| 3 | **Monitoring & observability** | Most postings | The difference between "I ran servers" and "I owned reliability." Prometheus + Grafana are the common language. |
| 4 | **Infrastructure as Code** | Most postings | How modern infra is actually built and reviewed. Terraform is now the #1 tool (note the licensing shift and OpenTofu). |
| 5 | **Kubernetes** | Most mid/senior postings | Still hot, and the de-facto base infrastructure layer for DevOps. Increasingly assumed at mid-level and above. |

## How the five fit together

These are not five isolated silos — they are one pipeline. A realistic mental
model of a modern deployment ties all five together:

```
   You write Terraform (4)  ──▶  provisions a Kubernetes cluster (5)
          on a cloud provider (1)
                    │
   A CI/CD pipeline (2) builds your app, runs tests,
   and deploys it onto that cluster
                    │
   Prometheus + Grafana (3) watch the running system
   and page you when it breaks
```

If you can tell one coherent story that walks through this whole loop — "I wrote
the Terraform that stood up our EKS cluster, wired a GitHub Actions pipeline to
deploy to it, and built the Grafana dashboards we used on call" — you have
demonstrated all five topics in a single sentence, and you sound like someone who
has actually done the job. That story is the goal of this module.

## What each topic really means in an interview

- **Cloud** is not "I have an account." It is "I know the core primitives
  (compute, networking, IAM, storage) of one provider well enough to design and
  debug on it."
- **CI/CD** is not "I clicked run." It is "I designed a pipeline: what triggers
  it, what the stages are, how secrets are handled, how a bad deploy is rolled
  back."
- **Observability** is not "we had Grafana." It is "*I built* dashboards and
  alerts for infrastructure *I owned*, and here is a time an alert I wrote caught
  a real problem." (Say **built**, not "used".)
- **IaC** is not "I edited a `.tf` file." It is "I understand state, modules, plan
  vs apply, and why we review infrastructure changes like code."
- **Kubernetes** is not "I know it exists." It is "I can explain
  Pod/Deployment/Service, debug a `CrashLoopBackOff`, and reason about why the
  scheduler put a pod where it did."

## Next steps

1. [`exercise.md`](./exercise.md) — rate yourself on each topic and build a
   personal study plan with a hands-on project per gap.
2. [`solution.md`](./solution.md) — a detailed roadmap for each of the five
   topics: subtopics, canonical tools, what "proficient" looks like, and concrete
   resources/projects.
3. [`deeper.md`](./deeper.md) — broader career questions: which cloud to pick,
   how to show observability experience, portfolio tips, and common interview
   formats.

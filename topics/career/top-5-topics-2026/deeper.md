# Deeper: Broader Career Follow-Ups

The five topics tell you *what* to learn. This file covers the questions that
come up once you start actually applying: which cloud to commit to, how to prove
observability experience when you've never had a production on-call rotation, how
to build a portfolio that does the talking, and what the interviews themselves
actually look like.

---

## 1. Which cloud should I pick?

**Short answer: pick one and go deep. Default to AWS.**

- **AWS** has the largest market share and, consistently, the most job postings —
  so it maximizes the number of doors your effort opens. Unless you have a
  specific reason not to, this is the safe default.
- **Azure** is the strong choice if you're targeting enterprise, government, or
  Microsoft-heavy shops (lots of ".NET + Azure" ecosystems). In some regions and
  industries it out-postings AWS — check your local market with the frequency
  script in `exercise.md`.
- **GCP** is smaller in raw job count but strong in data/ML-heavy companies and
  startups, and Kubernetes-native shops (Google originated Kubernetes).

**Why depth beats breadth:** the core concepts — compute, networking, IAM,
storage, load balancing — are the same everywhere; only the names and consoles
differ. An interviewer would much rather hear "I know AWS networking cold" than "I
dabbled in all three." Once you're genuinely proficient in one, learning a second
is fast, and you can honestly say "I'm deep on AWS and the concepts port directly
to Azure/GCP." Don't spread yourself thin chasing three logos.

**Don't ignore the bill.** Practicing on real cloud costs money. Use free tiers,
set a billing alarm on day one, and `destroy` everything the moment you're done
with an exercise. Being cost-conscious is itself a DevOps skill worth mentioning
in interviews.

---

## 2. How do I show observability experience (without a prod on-call job)?

Observability is the topic candidates most often *have read about* but can't
*demonstrate* — so demonstrating it is a huge differentiator. The key phrasing,
repeated from the roadmap: say you **BUILT** dashboards to monitor infrastructure
you **OWNED**. "Used Grafana" is passive and forgettable; "built the dashboards
and alerts for the service my team ran" is active and memorable.

If you don't have production experience yet, manufacture legitimate experience:

1. **Own something, even if it's yours.** Your exercise capstone app *is*
   infrastructure you own. Instrument it, build its dashboards, write its alerts.
   That's real, and you can say so honestly.
2. **Break it on purpose and catch it.** Inject errors or load, watch your alert
   fire, and screenshot the dashboard mid-incident. Now you have a story with a
   beginning, middle, and end — which is exactly what behavioral interviewers want.
3. **Speak the vocabulary.** Golden signals / RED / USE, SLI/SLO/error budget,
   alert fatigue, symptom-based alerting. Using this language correctly signals
   maturity even at junior level.
4. **Have one concrete story ready.** Rehearse: *"I built X dashboard around the
   golden signals for the app I ran. When I load-tested it, my p99-latency alert
   fired at Y, which pointed at Z, and I fixed it by W."* Concrete beats generic
   every time.

---

## 3. Portfolio tips

Your goal is to let your work do the talking so you're not relying on the
interviewer to take your word for it.

- **One strong capstone > ten toy repos.** The connected end-to-end project
  (Terraform → cloud → Kubernetes → CI/CD → observability) is worth more than a
  pile of unrelated tutorials. Pin it.
- **The README is the portfolio.** Most people never run your code — they read
  the README. Include an architecture diagram, a short "why I built it this way,"
  a screenshot of the Grafana dashboard, and clear run instructions. A good
  diagram in the README does more than 500 lines of YAML nobody reads.
- **Show your reasoning, not just your YAML.** A short "decisions and trade-offs"
  section ("I chose a private subnet because…", "I set these resource limits
  because…") demonstrates the judgment that separates senior from junior.
- **Write it up.** A short blog post or even a detailed README walking through
  "how I broke my app and my alert caught it" is a portfolio piece *and* an
  interview story generator.
- **Certifications: signal, not substitute.** AWS SAA, Terraform Associate, CKA/
  CKAD are useful resume filters and excellent syllabi (see `solution.md`), but
  they never replace a project you can demo. Use them to structure learning and to
  get past HR keyword filters — then let the project prove you can actually do it.
- **Keep your GitHub tidy.** Green squares and a couple of well-documented infra
  repos beat an abandoned graveyard of forks.

---

## 4. Common interview formats

DevOps/SRE interviews vary by company, but you'll see some combination of these.
Prepare for each.

- **Recruiter / phone screen.** Keyword and basics check. This is where the top-5
  topics act as *filters* — you need to at least sound fluent in cloud, CI/CD, and
  (increasingly) Kubernetes to pass. Have your one-sentence story ready:
  "I built an end-to-end pipeline that…"
- **Technical / conceptual interview.** "Explain the difference between a
  Deployment and a StatefulSet." "How does Terraform state work?" "What are the
  four golden signals?" Depth in your chosen topics pays off here. It's fine to
  say "I haven't used X, but here's how I'd approach it" — reasoning beats bluffing.
- **Troubleshooting / scenario ("what would you do").** The most common and most
  revealing format. "A pod is in `CrashLoopBackOff` — walk me through your
  debugging." "A deploy went out and error rates spiked — what now?" They're
  testing your *method*, not memorized answers. Narrate a systematic process:
  observe → form hypothesis → check → narrow down. This is where your hands-on
  projects shine, because you've actually done it.
- **System design.** "Design a CI/CD pipeline for a team of 20." "Design a
  monitoring stack for a web service." Whiteboard-friendly. Structure your answer
  around the topics: how does code get built and deployed, how is infra
  provisioned, how do you know it's healthy, how do you roll back. The `README.md`
  loop diagram is a good scaffold.
- **Hands-on / take-home.** Some companies give a practical task: write a
  pipeline, fix a broken Terraform config, debug a misbehaving cluster. Your
  exercise projects are direct preparation.
- **Behavioral.** "Tell me about a time something broke and how you handled it."
  "Describe a system you're proud of." Use STAR (Situation, Task, Action, Result)
  and lean on your project stories — especially the "I broke it and my alert
  caught it" observability story.

**Cross-cutting advice:** interviewers hire people who can *tell a coherent story
about a system they built and ran*. Every topic in this module feeds that story.
If you can walk someone through your capstone — how it's provisioned, deployed,
and monitored, and how you'd debug it at 3am — you will out-interview candidates
who merely collected certificates.

---

## 5. Beyond the top 5 (what to add next)

Once the five are solid, these round you out and show up in more senior postings:

- **Linux & networking fundamentals** — assumed everywhere; brush up on
  processes, systemd, permissions, DNS, TCP/HTTP. Weakness here shows fast.
- **Scripting** — Bash for glue, Python for anything bigger. You'll be asked to
  automate something.
- **Security / DevSecOps** — least privilege, secrets management, image scanning,
  supply-chain basics. Increasingly non-optional.
- **GitOps** — Argo CD / Flux; the natural evolution of CI/CD + Kubernetes.
- **Service mesh & advanced networking** (Istio/Linkerd) — senior/platform roles.
- **FinOps / cost optimization** — "cut our cloud bill by X%" is a great story.
- **Soft skills** — incident communication, writing clear postmortems, and
  collaborating with developers. DevOps is a culture, not just a toolset, and
  interviewers probe for it.

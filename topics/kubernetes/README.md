# Kubernetes — Interview Prep

Scenario-driven Kubernetes questions for DevOps/SRE interviews. Each folder holds four files:

- `README.md` — the scenario/question, posed the way an interviewer would.
- `exercise.md` — a hands-on lab you can actually run (kind/minikube + kubectl + docker): make it break, then fix it.
- `solution.md` — the detailed written answer and walkthrough.
- `deeper.md` — the follow-up questions an interviewer would drill into, with concise answers.

## Questions

| # | Question | Hook |
|---|----------|------|
| [q194](./q194-api-resources-short-names/) | List resource types + short names | `kubectl api-resources` asks the API server itself — so it prints every kind, its short name, and namespaced flag, CRDs included. |
| [q198](./q198-deploy-custom-scheduler/) | Deploy a custom scheduler | A scheduler is just a watch/bind controller — deploy it as a normal Deployment + RBAC in kube-system, and pods opt in via `schedulerName`. |
| [q202](./q202-daemonset-vs-deployment/) | DaemonSet vs Deployment | One pod per node automatically (agents, log collectors, CNI) vs N scaled replicas the scheduler scatters wherever they fit. |
| [q206](./q206-move-workloads-new-node-pool/) | Move workloads to a new node pool | `cordon` to stop new pods, then `drain` to evict them onto the new pool — respecting PodDisruptionBudgets via the Eviction API. |
| [q210](./q210-cicd-rollout-db-failure/) | CI/CD rollout, DB failure | Probes report healthy, but every pod is dead in the water — the readiness probe never checked the database it depends on. |
| [q215](./q215-game-server-pod-crash/) | Game server pod crash | A crashing game server shouldn't cost players their match — externalize state, redirect sessions, and replay buffered actions. |
| [q217](./q217-cfs-throttling-p99/) | CFS throttling, bad P99 | P99 latency is awful but the CPU charts look calm — the CFS quota is throttling you inside each 100 ms window. |
| [q221](./q221-pod-stuck-pending/) | Pod stuck in Pending | Three families of scheduling failure — no matching node, not enough resources, or an unbound volume — and how `describe` tells you which. |
| [q224](./q224-ingress-error-codes/) | Ingress 500/502/503/504 | Four 5xx codes, four different stories: the app erred, there's no backend, the backend timed out, or the backend answered with garbage. |
| [q225](./q225-ingress-vs-load-balancer/) | Ingress vs Load Balancer | L7 host/path routing that fans out to hundreds of services vs L4 forwarding by IP:port — and why the ingress sits behind an LB, not instead of one. |
| [q226](./q226-ingress-502-causes/) | Ingress 502 causes | Why 502s spike during every deploy — the endpoint-removal race, keepalive mismatches, and a starved controller — and how to kill them. |

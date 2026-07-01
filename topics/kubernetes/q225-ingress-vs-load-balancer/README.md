# q225 — Ingress (controller) vs Load Balancer

## The question, as an interviewer would ask it

> "In Kubernetes, what are the top differences between an **Ingress** (and its
> controller) and a **Load Balancer**? When would you reach for one versus the
> other, and how do they relate to each other?"

This is a classic systems-design/fundamentals screening question. The
interviewer is checking whether you understand the OSI layers, how traffic
actually enters a Kubernetes cluster, and whether you can reason about cost and
architecture — not whether you can recite a definition.

A strong answer hits three axes and then corrects the common misconception that
Ingress *replaces* a load balancer.

## What you should be able to answer

- **Layer:** Which OSI layer does each operate at? (L7 HTTP/HTTPS for an Ingress
  controller vs L4 TCP/UDP for a classic/cloud network load balancer — and why
  a cloud *ALB* muddies the "LB = L4" shorthand.)
- **Routing intelligence:** What can an L7 Ingress route on that an L4 LB
  cannot? (Host header, URL path, headers, cookies; plus TLS termination, path
  rewrites, sticky sessions, header manipulation.)
- **Fan-out / cost:** Why does exposing 100 HTTP services via
  `Service type=LoadBalancer` cost roughly 100 cloud load balancers, while one
  Ingress can front all 100 behind a single external IP?
- **The relationship:** Ingress does **not** replace the load balancer. A single
  cloud L4 LB usually sits *in front of* the ingress controller, which then does
  the L7 routing. Be able to draw: `Cloud LB (L4) → Ingress controller (L7) →
  Services → Pods`.
- **Service types:** ClusterIP vs NodePort vs LoadBalancer, and where each fits
  relative to Ingress.
- **When to use which:** Raw TCP/UDP (Postgres, non-HTTP gRPC streams) → L4 LB.
  HTTP microservices needing host/path routing → Ingress.
- **The successor:** What the **Gateway API** is and why it is replacing Ingress.

## Files in this topic

- `exercise.md` — a runnable `kind` + `kubectl` lab that demonstrates the
  difference concretely (NodePort L4 forwarding vs one Ingress host/path-routing
  to two apps behind a single entry point).
- `solution.md` — the full walkthrough, ASCII architecture diagram, Service-type
  comparison, and decision guidance.
- `deeper.md` — follow-up interview questions with concise correct answers.

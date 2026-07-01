# Deeper — follow-up interview questions

Concise, correct answers to the follow-ups an interviewer typically asks after
the main comparison.

### 1. Does an Ingress replace the cloud load balancer?

No. The ingress controller is just Pods, and those Pods have to be reachable
from outside the cluster. That is normally done by exposing the controller with
a single `Service type=LoadBalancer` (or NodePort). So a cloud **L4 LB sits in
front of** the ingress controller, which then does L7 routing to Services. The
path is: `Cloud LB (L4) → Ingress controller (L7) → Services → Pods`. Ingress
consolidates *many* services behind that *one* LB — it doesn't eliminate it.

### 2. What is the difference between an "ingress controller" and the "Ingress resource"?

The **Ingress resource** is a declarative API object (`kind: Ingress`) that
describes desired HTTP routing rules — hosts, paths, backend Services, TLS. It
is inert data. The **ingress controller** is the running software (nginx,
Traefik, HAProxy, Envoy, etc., deployed as Pods) that *watches* Ingress objects
and actually implements the routing. Without a controller installed, creating an
Ingress object does nothing — no traffic is routed.

### 3. How would you expose a raw TCP/UDP service like Postgres?

With an **L4 load balancer**, i.e. `Service type=LoadBalancer` (cloud NLB) — not
Ingress. Ingress is HTTP/HTTPS-only, so it cannot carry the Postgres wire
protocol. Options: `type=LoadBalancer` for a dedicated external L4 endpoint;
`ingress-nginx`'s special TCP/UDP ConfigMap feature (a bolt-on, not standard
Ingress); or, on newer clusters, the Gateway API's `TCPRoute`. The clean,
protocol-correct answer for a database is an L4 LB.

### 4. Compare `type=LoadBalancer`, `NodePort`, and `ClusterIP`.

- **ClusterIP** (default): stable virtual IP reachable **only inside** the
  cluster; used for pod-to-pod / service-to-service traffic.
- **NodePort**: opens a static high port (default 30000–32767) on **every
  node**; `nodeIP:nodePort` forwards into the Service. Externally reachable but
  crude — high ports, one per service, you manage node IPs yourself.
- **LoadBalancer**: asks the cloud to provision an **external L4 load balancer**
  with its own external IP for this one Service. It builds on NodePort under the
  hood. One LB per Service.

They stack: LoadBalancer → NodePort → ClusterIP → Endpoints (Pods).

### 5. What are the cost implications of many `type=LoadBalancer` services?

Each `type=LoadBalancer` Service provisions its **own** cloud load balancer with
its own hourly charge, its own data-processing charge, and its own external IP.
Expose 100 HTTP services this way and you pay for ~100 load balancers and manage
100 IPs and 100 DNS/TLS setups. Consolidating them behind **one** ingress
controller (one LB, one IP) cuts the LB bill dramatically and centralises
certificate and DNS management. This cost/consolidation argument is often the
main practical reason to choose Ingress for HTTP fleets.

### 6. Why choose the Gateway API over Ingress?

Ingress standardised only basic host/path routing; everything else (rewrites,
timeouts, canary weights, auth) lived in **vendor-specific annotations**, so
manifests weren't portable and advanced features weren't part of the spec. The
**Gateway API** fixes this: it is **role-oriented** (GatewayClass / Gateway /
HTTPRoute split across infra, platform, and app teams), more **expressive**
(traffic splitting, header matching and mutation, weighted backends as
first-class fields), and covers **both L4 and L7** (`TCPRoute`, `UDPRoute`,
`TLSRoute`, `GRPCRoute`). Ingress is now effectively **feature-frozen** in its
favour, so Gateway API is the recommended direction for new work.

### 7. Where does TLS terminate in each design?

- **L7 Ingress:** typically **terminates TLS at the ingress controller**. The
  controller holds the cert (often in a Secret managed by cert-manager), does
  SNI-based cert selection, decrypts, and routes on the plaintext HTTP. Traffic
  to backends is then plain HTTP inside the cluster (or re-encrypted if you want
  end-to-end TLS).
- **L4 LB:** can **pass TLS through** to the backend (the app/pod terminates it)
  or terminate at the LB in TCP mode. Because it's L4 it can route on **SNI**
  (visible before decryption) but cannot route on decrypted path/headers.

Choosing where TLS terminates is a design decision: at the edge (Ingress) for
centralised cert management, or end-to-end/passthrough (L4) for zero-trust or
when the backend must see the client cert (mTLS).

### 8. Can an L4 load balancer do path-based routing?

No. Path is part of the HTTP request, which lives at L7. An L4 LB only sees the
TCP/UDP 4-tuple (IPs and ports) and forwards connections; it never parses HTTP,
so `/a` vs `/b` are indistinguishable to it. Path- and host-based routing
require an L7 device — an ingress controller, a cloud ALB, or a Gateway API
HTTPRoute.

### 9. If the ingress controller is behind a single LB, isn't it a single point of failure / bottleneck?

The single **logical** entry point does not mean a single instance. You run the
controller with **multiple replicas** (a Deployment/DaemonSet) behind the LB,
spread across nodes/zones, with a PodDisruptionBudget and an HPA so it scales
with traffic. The cloud LB itself is a managed, redundant service. So it's a
single *address*, not a single *machine* — capacity and availability come from
replication behind that address.

# Solution — Ingress (controller) vs Load Balancer

The clean answer is built on three differences, then one crucial clarification
that Ingress does not *replace* a load balancer but layers on top of one.

## Difference 1 — OSI layer: L7 routing vs L4 forwarding

- **Ingress controller = Layer 7 (application layer, HTTP/HTTPS).** It parses
  the actual HTTP request. It sees the request line, the `Host` header, the URL
  path, other headers, cookies, and the method. It can terminate TLS (including
  SNI-based selection of certificates), rewrite paths, and manipulate headers.
- **Classic / cloud network load balancer = Layer 4 (transport layer,
  TCP/UDP).** It forwards *connections* by `IP:port`. It does not parse HTTP; at
  most it distributes new connections across a backend pool (round-robin, or a
  hash of the 4-tuple for connection affinity). It is protocol-agnostic — it
  will happily carry HTTP, Postgres wire protocol, raw TCP, or UDP, because it
  never looks inside.

**Be precise about the shorthand.** "Load balancer = L4" is only mostly true.
Cloud providers offer *two* families:

- **Network Load Balancer (NLB / L4):** TCP/UDP, IP:port. This is what a
  Kubernetes `Service type=LoadBalancer` typically provisions.
- **Application Load Balancer (ALB / L7):** HTTP-aware, does path/host routing
  itself. So an ALB overlaps functionally with an Ingress controller.

The real contrast the interviewer wants is **L7 content-based routing vs L4
connection forwarding**, not the literal word "load balancer." An Ingress
controller is one way to get L7 routing that is portable across clouds and lives
inside Kubernetes; a cloud ALB is another.

## Difference 2 — Routing intelligence

Because it works at L4, a classic LB can only decide *"which backend gets this
connection"* using information available in the TCP/UDP headers: source/dest IP
and port, maybe a hash for stickiness. It cannot make different decisions for
`GET /images` versus `GET /api`, because it never sees those bytes as HTTP.

An L7 Ingress can route on request *content*:

| Route on           | L4 LB | L7 Ingress |
| ------------------ | :---: | :--------: |
| Destination IP:port|   ✅  |     ✅     |
| Host header (vhost)|   ❌  |     ✅     |
| URL path           |   ❌  |     ✅     |
| HTTP headers/cookies|  ❌  |     ✅     |
| TLS termination/SNI|  ⚠️*  |     ✅     |
| Path rewrite       |   ❌  |     ✅     |
| Header manipulation|   ❌  |     ✅     |

\* An L4 LB can *pass through* or terminate TLS at the TCP level, but it cannot
route based on the decrypted HTTP content the way an L7 proxy does (it can do
SNI-based routing since SNI is visible pre-decryption, but not path/header
routing).

Concrete example of what only L7 can do:

```
app.example.com            -> service A (frontend)
api.example.com/v2         -> service B (API v2)
example.com/images/*       -> service C (static assets)
```

All three can share one IP and one port. The Ingress controller reads the
request and dispatches. An L4 LB physically cannot do this.

## Difference 3 — Fan-out / consolidation (and cost)

- **`Service type=LoadBalancer`: one cloud LB per Service.** Each such Service
  asks the cloud to provision a dedicated L4 load balancer with its own external
  IP. Expose 100 HTTP microservices this way and you get ~100 load balancers,
  ~100 external IPs, and roughly 100× the LB bill.
- **Ingress: one entry point, many Services.** A single Ingress controller,
  behind a single external LB, fronts tens or hundreds of Services via
  name-based virtual hosting and path rules. Adding a service is editing one
  Ingress object — no new load balancer, no new IP, no extra base cost.

For HTTP workloads this consolidation is usually the deciding factor:
one L4 LB + one ingress controller + N Ingress rules ≪ N cloud load balancers.

## The crucial clarification: Ingress does NOT replace the LB

This is where weak answers fall down. An Ingress is not an alternative to a load
balancer — it sits **behind** one. The ingress controller is itself just Pods,
and Pods need to be reachable from outside. So the controller is normally
exposed via a `Service type=LoadBalancer` (or `NodePort`). The result is a
**layered** data path:

```
                    Internet
                        │
                        ▼
        ┌───────────────────────────────┐
        │   Cloud L4 Load Balancer       │   one LB, one external IP
        │   (Service type=LoadBalancer)  │   L4: forwards TCP :80/:443
        └───────────────┬───────────────┘
                        │
                        ▼
        ┌───────────────────────────────┐
        │   Ingress Controller Pods      │   L7: reads Host / path / headers,
        │   (nginx / traefik / envoy)    │   terminates TLS, routes
        └───┬───────────┬───────────┬────┘
            │           │           │
   Host/path rules from the Ingress resource(s)
            │           │           │
            ▼           ▼           ▼
        ┌───────┐   ┌───────┐   ┌───────┐
        │ Svc A │   │ Svc B │   │ Svc C │   ClusterIP Services
        └───┬───┘   └───┬───┘   └───┬───┘
            ▼           ▼           ▼
          Pods        Pods        Pods
```

So the single cloud L4 LB does the "get traffic into the cluster" job, and the
ingress controller does the "figure out which service this HTTP request is for"
job. They are complementary layers, not competitors.

## Service types vs Ingress — where each fits

- **ClusterIP (default):** a stable virtual IP reachable **only inside** the
  cluster. Used for service-to-service traffic. Not externally reachable. This
  is what Ingress rules point *at*.
- **NodePort:** opens a static port (default 30000–32767) on **every node**;
  external traffic to `nodeIP:nodePort` is forwarded to the Service. Crude,
  high-port, one port per service. Building block that `LoadBalancer` and some
  ingress setups sit on top of.
- **LoadBalancer:** provisions an **external cloud L4 load balancer** for this
  one Service and gives it an external IP. One LB per Service → the fan-out cost.
- **Ingress:** *not* a Service type. It is a **separate API object** describing
  L7 HTTP routing rules, and it does nothing without an **ingress controller**
  (the Pods that actually implement the routing). The controller is typically
  exposed with a single `type=LoadBalancer` Service.

Mental model: Service types answer *"how is this one workload exposed at L3/L4?"*
Ingress answers *"how do I HTTP-route many workloads behind one entry point?"*

## When to use L4 LB vs L7 Ingress

Use an **L4 load balancer** (`Service type=LoadBalancer` / cloud NLB) when:

- The protocol is **not HTTP**: raw TCP (Postgres, MySQL, Redis), UDP (DNS,
  game servers, QUIC where you want L4), MQTT, SMTP, etc.
- You want **TLS/mTLS to pass through untouched** to the backend.
- You need the **lowest-latency, highest-throughput** path and don't need
  content-based routing.
- You explicitly need **one dedicated LB/IP per service** (e.g. per-tenant IPs).

Use an **L7 Ingress** (controller behind one LB) when:

- Traffic is **HTTP/HTTPS** and you want **host/path routing**, TLS termination,
  redirects, rewrites, or header/cookie logic.
- You are exposing **many HTTP microservices** and want to consolidate them
  behind one IP to control cost and simplify DNS/TLS management.
- You want a **single place** to manage certificates (often with cert-manager).

Note: gRPC is HTTP/2, so it can go through an L7 Ingress *if* the controller
supports HTTP/2/gRPC; some teams still front raw high-throughput gRPC via L4 for
simplicity. Plain TCP databases should never go through Ingress — use L4.

## The successor: Gateway API

The **Gateway API** is the official evolution of Ingress, addressing Ingress's
biggest limitation: everything beyond basic host/path routing was crammed into
vendor-specific **annotations**, so manifests weren't portable and advanced
features were unstandardised.

Gateway API is **role-oriented** and splits concerns across CRDs:

- **GatewayClass** (cluster-scoped): which controller implementation to use
  (analogous to a StorageClass). Managed by the infrastructure provider.
- **Gateway** (namespaced): a concrete entry point — listeners, ports,
  protocols, TLS config. Managed by cluster/platform operators.
- **HTTPRoute** (and `TCPRoute`, `TLSRoute`, `GRPCRoute`, `UDPRoute`): the
  actual routing rules. Managed by application developers.

Advantages: it is more expressive (traffic splitting, header matching/mutation,
weighted backends as first-class fields rather than annotations), it cleanly
separates the platform-team vs app-team responsibilities, and it standardises
**both L4 and L7** routing in one API. Because of this, **Ingress is now
effectively feature-frozen** — new capabilities land in Gateway API, and it is
the recommended direction for new clusters (where a Gateway-API controller is
available).

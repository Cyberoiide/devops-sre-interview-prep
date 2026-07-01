# Q200 — Solution and walkthrough

## The one-paragraph answer

TLS termination is *where the encrypted connection gets decrypted*, and it's a
placement decision with many valid answers. The most common is the **load
balancer** — one cert, crypto offloaded from the backends, which then receive
plaintext (or a re-encrypted hop). You can also terminate at the **web/app server
itself** (simplest topology, but cert management on every box), a **reverse proxy**
like nginx (terminate + cache + route in one place), an **API gateway** (one cert
for many endpoints plus auth/rate-limiting), the **CDN edge** (decrypt near the
user so clients never reach origin), an **HSM** (private key never leaves
tamper-resistant hardware, for high-assurance handling), or via **mTLS /
client-side** where both peers present certs (zero-trust meshes). The three
handling modes are **termination** (decrypt here, plaintext behind),
**passthrough** (forward encrypted bytes untouched, backend terminates), and
**re-encryption / end-to-end** (decrypt, inspect, then open a fresh TLS
connection to the backend). The trade-off that governs the choice: once you
terminate, **the hop behind the terminator is plaintext unless you re-encrypt** —
tolerable inside a trusted network, unacceptable across untrusted links or under
compliance regimes that mandate encryption in transit end-to-end.

---

## The three handling modes (get these straight first)

Everything else is a variation on what a component does with the TLS connection:

1. **Termination (offload).** The component holds the cert + key, completes the
   handshake with the client, decrypts, and passes **plaintext** onward. Cheapest
   (one handshake, crypto done once, backends freed from TLS) and most common.
   Cost: the backend hop is unencrypted.
2. **Passthrough (TLS passthrough / L4).** The component does *not* decrypt. It
   inspects only the **SNI** in the TLS ClientHello to route, then forwards the
   raw encrypted stream to a backend that terminates. The private key stays on
   the backend and the middle box never sees content — but it also can't cache,
   inspect, add headers, or do L7 routing.
3. **Re-encryption (TLS bridging / end-to-end).** The component terminates
   (decrypts, inspects, routes, whatever it needs plaintext for), then opens a
   **new, separate** TLS connection to the backend. Two handshakes, two cert
   sets, encryption on both hops. This is how you get L7 features *and* an
   encrypted backend link.

The interview tell: candidates who only know "termination" think TLS is a switch.
Knowing passthrough and re-encryption shows you understand it's a spectrum of
who-sees-what.

---

## The full menu of termination points

### 1. Load balancer — the default answer
The single most common place. Put the cert on the LB (AWS ALB/NLB, GCP LB, an
HAProxy/nginx LB), and it terminates TLS for the whole fleet: one cert to manage,
crypto offloaded from application servers, and the LB can do L7 routing on the
decrypted request. Backends receive plaintext by default; you can re-encrypt to
them if the LB↔backend network isn't trusted. **Use when:** anything with more
than one backend, which is almost everything.

### 2. Web / app server directly
The application server (or its embedded server — a Java keystore, a Node HTTPS
server, Kestrel, etc.) holds the cert and terminates. Truly end-to-end encrypted
to the app, simplest topology (no extra hop). **Catch:** every server needs the
cert and its renewal, there's no central offload, and TLS crypto competes with
app CPU. **Use when:** a single server, a simple internal service, or a
requirement that nothing but the app ever sees plaintext.

### 3. Reverse proxy (nginx, Caddy, Envoy, HAProxy)
Terminate at a proxy that *also* caches, compresses, rate-limits, and routes.
This is what the exercise builds. It's the LB pattern at a smaller scale, or a
layer *behind* the LB. **Use when:** you want termination co-located with caching
/ routing / TLS policy in one well-understood component. Caddy is notable for
automatic Let's Encrypt certs.

### 4. API gateway (AWS API Gateway, Kong, Apigee)
Terminate at the gateway that fronts many endpoints/microservices: **one cert
instead of one per endpoint**, plus centralized auth, API keys, throttling, and
request transformation on the decrypted traffic. **Catch:** the gateway is a
critical choke point and (managed ones especially) can be a real cost line.
**Use when:** you have many APIs behind one front door and want policy centralized.

### 5. CDN edge (CloudFront, Cloudflare, Fastly, Akamai)
Terminate at the CDN's global edge, physically close to the user. The client's
TLS handshake completes at a nearby POP instead of your distant origin, cutting
latency, and clients **never reach origin directly** (DDoS absorption, origin
hiding). The CDN→origin hop is usually re-encrypted. **Catch:** the CDN holds
your cert/key (or you use their managed cert), so you're extending trust to the
CDN. **Use when:** global user base, static/cacheable content, or you want the
edge to shield origin.

### 6. HSM (Hardware Security Module)
Not a full terminator so much as *where the private key lives and does its work*.
The TLS terminator offloads the private-key operations to an HSM (CloudHSM, a
network HSM, or hardware appliances) so the **key never leaves tamper-resistant
hardware** — it's used but never exportable. **Use when:** ultra-high-security or
compliance (PCI-DSS, FIPS 140-2/3, root CAs, payment systems) demands provable
key custody. **Catch:** cost, latency per signing op, and operational complexity.

### 7. Client-side / mTLS
**Mutual TLS**: the server presents a cert *and* the client presents one, each
verifying the other. "Termination" happens at both ends and identity is
cryptographic on both sides. This is the backbone of **zero-trust service meshes**
(Istio, Linkerd) where every service-to-service call is mTLS. **Catch:**
distributing, rotating, and revoking client certs across every peer is real
operational work (meshes automate it with a sidecar + control plane). **Use
when:** service-to-service auth, zero-trust networks, or B2B APIs needing strong
client identity.

---

## The security trade-off: the plaintext backend hop

The moment you terminate, whatever is behind the terminator sees **plaintext**.
That's fine or fatal depending on the network:

- **Trusted segment (single VPC, private subnet).** Terminating at the LB and
  sending plaintext to backends is standard and widely accepted — the internal
  network is considered trusted, and you save the cost of a second handshake.
- **Untrusted or shared path** (traffic crosses the public internet, a shared
  datacenter, region-to-region links, or a multi-tenant environment). Plaintext
  here is exposed to anyone on the wire. You **re-encrypt** so the backend hop is
  TLS too.
- **Compliance regimes.** PCI-DSS, HIPAA, and similar often mandate encryption
  *in transit end-to-end* — which forbids a plaintext internal hop and forces
  re-encryption or passthrough regardless of how "trusted" you consider the
  network.

Modern practice is trending toward **zero-trust**: treat the internal network as
hostile too, and encrypt every hop (mTLS mesh, or re-encrypt at each terminator).
The old "hard shell, soft interior" model — TLS at the edge, plaintext inside —
is increasingly seen as a liability.

---

## How to choose (the reasoning an interviewer wants)

- **Default:** terminate at the load balancer. One cert, offloaded crypto, L7
  routing. Re-encrypt to backends if the internal hop isn't trusted or compliance
  requires it.
- **Add a CDN** and terminate at the edge when users are global and you want
  latency + origin protection.
- **Add an API gateway** as the termination point when you're fronting many APIs
  and want centralized auth/rate-limiting.
- **Push termination to the app / use mTLS everywhere** when the security model
  is zero-trust or nothing internal may see plaintext.
- **Bring in an HSM** when key custody must be provable for compliance.
- **Use passthrough** when a downstream must own its own cert/key and no middle
  box may ever decrypt (some regulated or end-to-end-encrypted designs).

---

## What NOT to do

- Don't assume "TLS at the edge" means you're encrypted everywhere — name the
  backend hop and decide termination vs re-encryption deliberately.
- Don't scatter certs across every server if you can offload to a LB/proxy;
  cert renewal sprawl is a top outage cause (expired certs). Centralize and
  automate (ACM, Let's Encrypt, cert-manager).
- Don't confuse passthrough with re-encryption — passthrough means the middle box
  can't see the traffic *at all*, re-encryption means it can (it decrypts, then
  re-encrypts). They have opposite inspection properties.
- Don't forget SNI when one terminator serves multiple hostnames/certs — without
  it the terminator can't pick the right cert.
- Don't hardcode weak protocols — disable TLS 1.0/1.1, prefer 1.2/1.3 (the lab
  pins `TLSv1.2 TLSv1.3`).

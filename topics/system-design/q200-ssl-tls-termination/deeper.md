# Q200 — Interviewer follow-ups

**Q: What's the difference between TLS passthrough and re-encryption? They both
sound like "the backend gets TLS."**
Opposite inspection properties. In **passthrough** the middle box never decrypts —
it forwards the raw encrypted stream (routing only on the unencrypted SNI field
at L4), and the backend does the one and only termination. The middle box can't
cache, read, or modify anything. In **re-encryption** the middle box *does*
terminate (decrypts, can inspect/route/add headers on plaintext), then opens a
brand-new TLS connection to the backend. Two separate TLS sessions vs one
end-to-end session. Choose passthrough when nothing but the backend may ever see
the content; choose re-encryption when you need L7 features *and* an encrypted
backend hop.

**Q: If terminating at the load balancer leaves the backend hop plaintext, why is
that the default? Isn't that insecure?**
It's a deliberate trust decision. Inside a single VPC / private subnet, the
internal network is treated as trusted, and skipping the second handshake saves
latency and CPU at scale. It's insecure only if that assumption is wrong — a
shared network, traffic crossing the public internet or regions, a multi-tenant
environment, or a compliance regime (PCI/HIPAA) that mandates end-to-end
encryption. Then you re-encrypt to the backend. The zero-trust trend is pushing
even internal hops toward always-encrypted (mTLS mesh), so "trusted internal
network" is an assumption you should state explicitly, not assume.

**Q: What does terminating at the load balancer actually buy you versus at each
app server?**
Three things: **one cert to manage and renew** instead of one per server (cert
sprawl and expiry are top outage causes); **crypto offload** so TLS handshakes
don't compete with application CPU on every box; and **L7 visibility at a central
point** for routing, WAF, and observability. The cost is that the LB→backend hop
is plaintext unless re-encrypted, and the LB becomes a component you must secure
and scale. Terminating at each app server gives true end-to-end encryption and a
simpler topology, at the price of managing certs everywhere and no central
offload.

**Q: Where does SNI fit in, and why does it matter for termination?**
SNI (Server Name Indication) is the hostname the client puts in the TLS
ClientHello, *before* the handshake completes — it's how one IP / one terminator
can host many sites with different certs: the terminator reads SNI, picks the
matching cert, and completes the handshake. It's also the only thing a
**passthrough** L4 proxy can route on, since it can't see anything encrypted. The
catch: classic SNI is sent in cleartext, so it leaks which host you're visiting —
which is what **Encrypted Client Hello (ECH)** is designed to fix.

**Q: What's mTLS and when would you actually use it over regular TLS?**
Mutual TLS: both sides present and verify certificates, so the *server*
authenticates the *client* cryptographically, not just the reverse. You use it
for service-to-service auth in a **zero-trust** architecture (every internal call
is mutually authenticated and encrypted — the model service meshes like Istio and
Linkerd implement with sidecars), and for high-assurance B2B APIs where you need
strong client identity instead of just an API key. The hard part is operational:
issuing, distributing, rotating, and revoking client certs at scale — which is
exactly why meshes automate it with a control plane and short-lived certs.

**Q: How does terminating at the CDN edge change the security and trust picture?**
The client's TLS handshake completes at a CDN POP near the user, so clients never
reach your origin directly — great for latency, DDoS absorption, and origin
hiding. But it means **the CDN holds (or issues) your cert and sees your
plaintext** at the edge, so you're extending your trust boundary to the CDN
provider. The CDN→origin hop is typically re-encrypted (and you can lock it down
so origin only accepts connections from the CDN). It's the right call for global,
cacheable traffic; the trade is trusting a third party with decryption.

**Q: What's an HSM doing in a list of termination points — it doesn't terminate
connections, does it?**
Correct — it's not a connection terminator, it's where the **private key lives and
performs its operations**. The TLS terminator (LB, server, whatever) offloads the
private-key signing/decryption step to the HSM, so the key is used but can never
be exported from tamper-resistant hardware. You reach for it under compliance or
high-assurance requirements — PCI-DSS, FIPS 140-2/3, root/intermediate CAs,
payment systems — where you must *prove* the key was never extractable. The cost
is latency per operation, money, and operational complexity.

**Q: What actually breaks when a cert expires, and how does termination placement
affect blast radius?**
An expired cert makes clients refuse the handshake — instant, total outage for
everything behind that terminator. Placement determines blast radius and fix
speed: a cert on a central LB is one thing to renew (and automate via ACM /
cert-manager / Let's Encrypt), so it's easy to keep current but a single expiry
takes down the whole fleet. Certs scattered across many app servers spread the
risk out but multiply the number of things that can silently expire. Either way,
the fix is **automated renewal + expiry monitoring/alerting** — expired certs are
one of the most common and most preventable outages there is.

**Q: The client curl rejected your self-signed cert until you passed `--cacert`.
What's happening, and how does production avoid that step?**
The client validates the server cert against a set of trusted CAs. A self-signed
cert's issuer is itself, which is in nobody's trust store, so validation fails
(`self-signed certificate`). `--cacert server.crt` tells curl to trust that
specific cert as a CA; `-k` skips validation entirely (never in production). Real
systems avoid the whole dance by using a cert signed by a CA the client *already*
trusts — a public CA like Let's Encrypt for internet-facing services, or an
internal CA whose root is distributed to clients for internal ones. Then
validation just passes.

**Q: TLS 1.2 vs 1.3 — does it matter which the terminator negotiates?**
Yes. TLS 1.3 drops the insecure/legacy cipher suites, mandates forward secrecy,
and cuts the handshake to **one round trip** (vs two in 1.2) — with 0-RTT
resumption for repeat connections — which is a real latency win at the terminator.
It also encrypts more of the handshake. You disable TLS 1.0/1.1 entirely (long
deprecated, PCI-forbidden) and prefer 1.3, falling back to 1.2 for older clients.
The lab pins exactly `TLSv1.2 TLSv1.3` for this reason.

# Q200 — Where can you terminate SSL/TLS?

## The scenario (as an interviewer would pose it)

> "A client makes an HTTPS request to your service. Somewhere along the path,
> that TLS connection gets decrypted. Where can that happen? Give me every place
> you could terminate TLS, not just 'the web server' — I want the full menu and
> the trade-offs.
>
> Then tell me the difference between terminating TLS, passing it through
> untouched, and re-encrypting it — and what it means for security when the hop
> behind your terminator is plaintext."

## What the interviewer is really probing

1. Do you know **TLS termination is a placement decision**, not a fixed fact —
   the cert and the decryption can live at many layers, each with different
   operational and security consequences?
2. Can you list the real options — **load balancer, web/app server, reverse
   proxy, API gateway, CDN edge, HSM, and client-side/mTLS** — and say when each
   is the right call?
3. Do you understand the three handling modes — **termination, passthrough,
   re-encryption (end-to-end)** — and the plaintext-backend-hop trade-off?
4. Have you actually held a cert and a private key in your hands, or only read
   about it? (The exercise is where you prove it.)

## The core distinction: what happens to the encrypted connection

There are exactly three things a component in the path can do with TLS:

- **Termination** — the component decrypts here. It holds the certificate and
  private key, does the TLS handshake with the client, and hands **plaintext**
  to whatever is behind it. Cheapest and most common; the backend hop is
  unencrypted.
- **Passthrough** — the component does *not* decrypt. It forwards the raw
  encrypted bytes (usually by routing on the TLS SNI field at L4) to a backend
  that terminates. The terminator's key never leaves the backend; the middle box
  can't see or cache the content.
- **Re-encryption (end-to-end / TLS bridging)** — the component terminates
  (decrypts), does its job on plaintext, then opens a **new** TLS connection to
  the backend. Two separate TLS sessions. You get inspection/routing *and* an
  encrypted backend hop, at the cost of two handshakes and two cert sets.

The security question that hangs over all of it: **once you terminate, the hop
behind the terminator is plaintext unless you re-encrypt.** Inside a trusted VPC
that's often accepted; across untrusted networks, or under compliance regimes
that mandate encryption in transit end-to-end, it isn't.

## The full menu of termination points

| Where you terminate       | Why you'd do it                                      | The catch                                             |
|---------------------------|------------------------------------------------------|-------------------------------------------------------|
| **Load balancer**         | Most common. One cert on the LB, backends offloaded  | Backend hop plaintext unless you re-encrypt           |
| **Web / app server**      | Simplest topology, true end-to-end to the app        | Cert + renewal on every server; no central offload    |
| **Reverse proxy** (nginx) | Terminate + cache + compress + route in one place    | Another hop to run and secure                         |
| **API gateway**           | One cert for many endpoints, plus auth/rate-limit    | Gateway becomes a critical, sometimes costly, choke   |
| **CDN edge**              | Terminate near the user; clients never reach origin  | CDN holds your key/cert; origin trust model needed    |
| **HSM**                   | Private key never leaves tamper-resistant hardware   | Cost and complexity; for high-assurance key handling  |
| **Client-side / mTLS**    | Both sides present certs; zero-trust service mesh    | Cert distribution/rotation to every client is real work|

## The mental model

TLS termination is about **where trust boundaries sit**. You terminate as close
to the edge as you can for performance (offload crypto, terminate near the user),
and you re-encrypt or push termination deeper when the network between hops isn't
trusted or compliance demands it. The load balancer is the default answer; the
*interesting* answer is knowing the other six and when each earns its keep.

## Deliverables

- `exercise.md` — a runnable lab: generate a self-signed cert, stand up **nginx
  in Docker terminating TLS in front of a plaintext backend**, `curl` through it,
  and inspect the handshake with `openssl s_client`.
- `solution.md` — the full written answer: every termination point, the three
  handling modes, and the security trade-offs.
- `deeper.md` — the follow-up questions an interviewer asks next.

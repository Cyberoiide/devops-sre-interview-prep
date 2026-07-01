# Q200 — Hands-on lab: terminate TLS at a reverse proxy

You'll generate a self-signed certificate, stand up **nginx (in Docker)
terminating TLS in front of a plaintext HTTP backend**, prove that clients speak
HTTPS to nginx while nginx speaks plaintext to the backend, and inspect the
handshake byte-by-byte with `openssl s_client`. This is the load-balancer /
reverse-proxy termination pattern, built with your own hands.

---

## 0. Prerequisites

- Docker.
- `openssl` and `curl` (both ship with essentially every Unix; you already have them).
- A working directory: `mkdir tls-lab && cd tls-lab`.

---

## 1. Generate a self-signed certificate

One command produces a 2048-bit RSA key and a matching self-signed cert valid
for a year, with `localhost` as both the subject and a Subject Alternative Name
(modern clients require the SAN, not just the CN):

```bash
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout server.key -out server.crt -days 365 \
  -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost"
```

- `-x509` → make a self-signed cert, not a signing request.
- `-nodes` → don't encrypt the private key (no passphrase), so nginx can start
  unattended. In production the key is protected; here we're teaching.

Inspect what you made:

```bash
openssl x509 -in server.crt -noout -subject -issuer -ext subjectAltName
```

You'll see `subject` and `issuer` are **the same** — that's what "self-signed"
means, and it's why browsers/curl will complain until you tell them to trust it.

---

## 2. Start a plaintext backend

`http-echo` is a tiny server that returns a fixed string over **plain HTTP** — it
speaks no TLS at all, which is exactly the point: the encryption stops at nginx.

```bash
docker network create tls-lab-net

docker run -d --name tls-backend --network tls-lab-net \
  hashicorp/http-echo:latest \
  -text="hello from the PLAINTEXT backend (port 5678, no TLS)"
```

It listens on port 5678 by default, HTTP only.

---

## 3. Configure nginx to terminate TLS and proxy to the backend

Create `nginx.conf`:

```nginx
events {}
http {
    server {
        listen 443 ssl;
        server_name localhost;

        # nginx holds the cert + key and does the TLS handshake with the client
        ssl_certificate     /etc/nginx/certs/server.crt;
        ssl_certificate_key /etc/nginx/certs/server.key;
        ssl_protocols       TLSv1.2 TLSv1.3;

        location / {
            # ...then forwards PLAINTEXT HTTP to the backend. This hop is
            # unencrypted — the whole point of "termination".
            proxy_pass http://tls-backend:5678;
            proxy_set_header Host $host;
        }
    }
}
```

Start nginx, mounting the config and the cert/key, publishing 443 as host 8443:

```bash
docker run -d --name tls-proxy --network tls-lab-net -p 8443:443 \
  -v "$PWD/nginx.conf:/etc/nginx/nginx.conf:ro" \
  -v "$PWD/server.crt:/etc/nginx/certs/server.crt:ro" \
  -v "$PWD/server.key:/etc/nginx/certs/server.key:ro" \
  nginx:1.27-alpine

sleep 2 && docker ps --format '{{.Names}}\t{{.Status}}' | grep tls
```

---

## 4. Prove the client speaks HTTPS

```bash
# -k accepts the self-signed cert. You get the backend's plaintext response,
# delivered to *you* over TLS.
curl -sk https://localhost:8443/
```

```
hello from the PLAINTEXT backend (port 5678, no TLS)
```

Now drop `-k` and watch curl **refuse** — a real client won't trust a
self-signed cert:

```bash
curl https://localhost:8443/
# curl: (60) SSL certificate problem: self-signed certificate
```

Tell curl to trust *this specific* cert as the CA, and it works cleanly with
full verification:

```bash
curl --cacert server.crt https://localhost:8443/
```

That's the whole trust model in three commands: reject unknown → explicitly trust
→ verified success. In production you skip this by using a cert signed by a CA the
client already trusts (Let's Encrypt, your cloud's ACM, an internal CA).

---

## 5. Inspect the handshake with `openssl s_client`

This is the tool you reach for when TLS misbehaves in production — it shows the
negotiated protocol, cipher, and the cert chain:

```bash
echo "Q" | openssl s_client -connect localhost:8443 -servername localhost 2>/dev/null \
  | grep -E "Protocol|Cipher|subject=|issuer=|Verify"
```

```
subject=CN=localhost
issuer=CN=localhost
Verification error: self-signed certificate
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Protocol: TLSv1.3
```

Read it: TLS **1.3** was negotiated, an AES-256-GCM cipher chosen, the cert is
`CN=localhost` — and the verification *error* is precisely because subject ==
issuer (self-signed). `-servername localhost` sends **SNI**, the hostname in the
handshake that lets one IP serve many certs; drop it and a multi-site terminator
wouldn't know which cert to present.

---

## 6. See that the backend hop really is plaintext

The backend itself proves it: it only ever spoke HTTP. Confirm nothing but nginx
is exposed, and that the backend has no TLS listener at all:

```bash
# The backend port is NOT published to the host — only nginx (8443) is reachable.
curl -s http://localhost:5678/ ; echo "  <- nothing here, backend isn't exposed"

# Inside the docker network, the backend answers plain HTTP with no TLS:
docker run --rm --network tls-lab-net curlimages/curl:latest \
  -s http://tls-backend:5678/
```

The second command returns the greeting over plain HTTP. That traffic between
nginx and the backend is unencrypted — fine inside a trusted network segment, a
finding to fix if that segment isn't trusted (see `solution.md` on
re-encryption).

---

## 7. Cleanup

```bash
docker rm -f tls-proxy tls-backend
docker network rm tls-lab-net
```

---

## What you proved

- **You** terminated TLS at a reverse proxy: the client did a real TLS 1.3
  handshake against a cert you minted, and nginx handed plaintext HTTP to a
  backend that can't speak TLS at all.
- The self-signed cert is rejected until explicitly trusted — the exact trust
  mechanic a real CA short-circuits.
- `openssl s_client` reads out protocol, cipher, and cert chain — your first move
  when diagnosing a TLS problem.
- The nginx→backend hop is plaintext. That's the security trade-off of
  termination in one observable fact: acceptable inside a trusted VPC, something
  you re-encrypt when it isn't.

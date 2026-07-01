# q226 — What causes a 502 Bad Gateway from an Ingress controller?

## The interview scenario

> "Your service sits behind an nginx-ingress controller. Users report
> intermittent **502 Bad Gateway** errors, and they get noticeably worse
> during deploys. You are not seeing 503s or 504s — specifically 502s.
>
> Walk me through what a **502** means coming from the ingress, how it differs
> from a **503** and a **504**, what the most likely root causes are, and how
> you would debug and fix it."

This is one of the most common real-world Kubernetes networking questions,
because "502 during deploys" is an almost universal symptom of a missing
graceful-shutdown pattern. A strong candidate can (a) precisely distinguish
502 vs 503 vs 504, (b) name the concrete causes and the mechanism behind each,
(c) read the nginx-ingress log lines that identify the cause, and (d) state the
fix for each.

## What you should be able to answer

- **The status-code anchor.** Why does the ingress return 502 specifically,
  and how is that different from 503 (no backend available) and 504 (backend
  reached but timed out)? What does 502 tell you that the other two do not?
- **The "502s during every deploy" root cause.** Why does rolling a Deployment
  produce a burst of 502s, and what is the endpoint-removal-vs-connection race
  underneath it? How do `preStop`, `terminationGracePeriodSeconds`, and
  readiness probes eliminate it?
- **The keepalive-mismatch cause.** Why can idle keepalive connections between
  nginx and your backend cause 502s, and what is the timeout-ordering rule that
  fixes it?
- **The controller-starvation cause.** How can the ingress controller pod
  itself (CPU/memory/OOM/fd limits) be the source of 502s?
- **The "wrong port / wrong protocol" causes.** How do a misconfigured Service
  `targetPort`, an app not listening, or an HTTP/HTTPS mismatch produce 502s
  with `Connection refused` / `Connection reset` in the logs?
- **The debugging workflow.** Which logs, `kubectl` commands, and nginx
  annotations you reach for, and the exact log signatures that point at each
  cause.

## Files in this topic

- `README.md` — this file (the question and what to master).
- `exercise.md` — a runnable lab (kind + ingress-nginx) that reproduces 502s
  from a wrong `targetPort` and from pod termination during a rollout, then
  fixes each.
- `solution.md` — the full explanation: the 502/503/504 anchor, each cause with
  its mechanism, log signatures, and fixes, plus a differential table.
- `deeper.md` — follow-up interview questions with concise answers.

# q210 — CI/CD Rollout: Pods Are "Ready" but the App Can't Reach the Database

## The scenario (as an interviewer would pose it)

> Your CI/CD pipeline deploys a new version of a stateless web service to
> Kubernetes. The rollout completes cleanly: `kubectl rollout status` reports
> success, every new pod is `Running` and `1/1 Ready`, and both the liveness
> and readiness probes are passing. There are no `CrashLoopBackOff` pods and no
> restarts.
>
> But in production the service is broken. Every user request returns HTTP 500.
> The application logs show the pods cannot open a connection to the backend
> PostgreSQL database. The previous version worked fine minutes ago; nothing
> changed on the database side.
>
> **Walk me through this.** Why did the rollout succeed while the service is
> down? What is the most likely root cause, how do you confirm it, and how do
> you change the system so this class of failure cannot silently ship again?

This is a very common real-world incident: a deploy that is *green* in
Kubernetes but *down* in reality. The interesting part is not "the DB is
unreachable" — it's *why Kubernetes happily declared the pods healthy and sent
them live traffic anyway*, and what that tells you about how probes should be
designed.

## What you should be able to answer

- **Why the failure timing is diagnostic.** It coincides *exactly* with the
  deploy, so it is almost certainly a config change shipped by the new version
  (a rotated/wrong Secret, a wrong DB host/port/name in a ConfigMap or env var),
  not random infrastructure failure.
- **Why the pods were marked Ready anyway.** The readiness probe checks a
  *shallow* signal (HTTP 200 on `/healthz`, or a TCP check on the app's own
  port) that does not exercise the database dependency. Ready ≠ "can actually
  serve".
- **The precise difference between the three probes** — liveness, readiness,
  and startup — and exactly what Kubernetes does when each one fails
  (restart vs. remove-from-endpoints vs. gate-the-other-two).
- **The probe-design fix:** a deep `/readyz` that verifies real serving
  capability (including a DB check) so bad pods never become Ready, which makes
  the rolling update halt itself while old pods keep serving.
- **The critical caveat:** a shared hard dependency in readiness can turn a
  transient DB blip into a self-inflicted, cluster-wide outage (cascading
  unreadiness). You must be able to argue *both* schools of thought and the
  mitigations.
- **The prevention layers beyond probes:** validating config/Secrets in CI,
  progressive delivery (canary / blue-green gated on `kubectl rollout status`
  with automatic rollback), a post-deploy smoke test that actually exercises the
  DB path, and secret-rotation drift detection.

## Files in this folder

| File | Purpose |
|------|---------|
| `README.md` | This file — the scenario and what you should be able to answer. |
| `exercise.md` | A runnable hands-on lab (kind + kubectl + docker) that reproduces the bug and then fixes it with a deep readiness probe. |
| `solution.md` | The full written answer: diagnosis reasoning, triage commands, the probe-design fix, the cascading-unreadiness caveat, and prevention layers. |
| `deeper.md` | Follow-up interview questions with concise, correct answers. |

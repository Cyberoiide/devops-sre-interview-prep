# q215 — Handling a Game Server Pod Crash Without Ruining the Player Experience

## The Interview Question

> "You run real-time multiplayer game servers on Kubernetes. Each pod hosts
> live matches, and the authoritative game state lives in the pod's memory.
> A game server pod crashes mid-match (OOM, node failure, or a bug).
> How do you keep the crash from turning into a ruined match — lost progress,
> disconnected players, angry reviews?"

This is a **stateful, latency-sensitive workload** problem dressed up as a
Kubernetes question. The interviewer wants to see that you understand *why
vanilla Kubernetes primitives (Deployments, HPA, liveness probes, rolling
updates) are actively hostile to long-lived stateful sessions*, and that you
can design around the crash so it becomes a **short reconnect blip** instead of
a lost match.

## What You Should Be Able to Answer

- **The three recovery pillars**, and why each one matters for perceived player
  experience:
  1. **Externalize state** — keep authoritative/recent state out of the
     crashing pod's memory (in-memory cache like Redis + periodic snapshots).
  2. **Session directory + redirect** — a matchmaking/session service knows
     *which pod* hosts each player; on a crash the client is redirected to a
     healthy pod that rehydrates state from the cache.
  3. **Action-replay / sync logs** — snapshots are always a few seconds stale;
     buffer the player's recent input events and replay them on top of the
     restored snapshot to close the gap.
- **Why the snapshot is always stale**, and how the action log closes the
  "staleness gap" so the player sees a tiny resync instead of a rollback.
- **StatefulSet vs Deployment vs Agones** — what stable identity buys you, and
  why real-time game fleets reach for a dedicated orchestrator (Agones) rather
  than plain Deployments + HPA.
- **Graceful shutdown mechanics** — `SIGTERM` handling, `preStop` hooks, and
  `terminationGracePeriodSeconds`; how a *graceful drain* produces a fresh
  snapshot while a *crash* leaves a stale one.
- **PodDisruptionBudget** — how it caps *voluntary* disruptions (node drains,
  upgrades) so you don't evict a whole fleet of active-match pods at once.
- **Voluntary vs involuntary disruptions** — which strategy defends against
  which class of failure.

## Files in This Topic

| File | Purpose |
|------|---------|
| `README.md` | This file — the scenario and the checklist of what to master. |
| `exercise.md` | A runnable kind/minikube lab that *simulates* a game server: Redis for state, buffered actions, a crash, rehydration + replay, graceful shutdown, StatefulSet variant, and a PDB drain demo. |
| `solution.md` | The full written walkthrough of the three pillars, the staleness gap, orchestration choices, graceful shutdown, and PDBs — with the *why* behind each. |
| `deeper.md` | Follow-up interview questions (idempotency, cache failure, node vs process failure, cluster upgrades) with concise correct answers. |

## The One-Sentence Answer

> "Don't trust the pod's memory: externalize recent state to a fast cache with
> periodic snapshots, keep a session directory so reconnecting clients land on a
> healthy pod that rehydrates, buffer recent player actions to replay on top of
> the snapshot, run graceful shutdown so planned terminations flush a fresh
> snapshot, and use a dedicated game-server orchestrator (Agones) plus a
> PodDisruptionBudget because vanilla Deployments/HPA assume fungible stateless
> pods and will happily evict an active match."

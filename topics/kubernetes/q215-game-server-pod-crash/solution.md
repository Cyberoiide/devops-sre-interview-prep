# q215 — Solution: Handling a Game Server Pod Crash

## Reframing the Problem

The naive mental model is "the pod crashed, Kubernetes restarts it, we're fine."
That model is correct for **stateless request/response services** and completely
wrong for a **real-time game server**. A game server is:

- **Stateful** — the authoritative match state (positions, scores, physics,
  inventory) lives in the pod's memory.
- **Long-lived and non-fungible** — "this pod is hosting *this* match with
  *these* players." You cannot transparently swap it for another replica the way
  you can with a stateless web pod.
- **Latency-sensitive** — players feel every stutter, rollback, and
  disconnect.

So when the pod crashes, the default Kubernetes behavior (kill, reschedule, new
empty pod) means **the match's memory is gone**. If that memory was the only
copy of the state, the match is lost. The entire solution is about ensuring the
crash costs a *short reconnect and a tiny resync*, not a lost match.

There are two independent goals:

1. **Survive the crash you didn't plan for** (OOM, node failure, bug) — the
   three pillars below.
2. **Make the disruptions you *do* control cheap** (rollouts, drains,
   scale-down) — graceful shutdown + PodDisruptionBudget.

---

## Pillar 1 — Externalize State

**Rule: never let the pod's memory be the only copy of authoritative state.**

The crashing pod's RAM is volatile and can disappear at any instant with no
warning (that is what "involuntary disruption" means). So you continuously push
state *out* of the pod:

- **Fast in-memory cache (e.g. Redis):** stores the *recent* game/session state
  so a replacement pod can **rehydrate quickly** — sub-millisecond reads, which
  matters because you're trying to get a player back into a live match, not
  restore a backup overnight. Redis is the canonical choice because it is
  in-memory (fast), supports the data structures you need (strings for
  snapshots, lists/streams for action logs), and is easy to run HA.
- **Periodic checkpoints/snapshots:** every N seconds (or every game tick
  boundary) you serialize the match/player state and write it to the cache
  (and, for durability, to a persistent store behind it). This is the "known
  good state" the replacement loads.

Why this matters for player experience: rehydrating from a snapshot is the
difference between "you reconnected and you're roughly where you left off" and
"the match reset to zero." It converts a catastrophic loss into a recoverable
one.

The cost you accept: snapshots are **periodic**, so at the moment of the crash
the latest snapshot is always **a few seconds stale**. Pillar 3 exists to pay
down exactly that cost.

---

## Pillar 2 — Session Directory + Redirect

**Rule: something must know *where* each player's session lives, so a
reconnecting client can be sent to the right healthy pod.**

A **session/matchmaking server** (a "session directory") maintains the mapping:

```
player alice   -> gameserver instance / pod  (host:port)
match #4821    -> gameserver-sts-0
```

When a pod crashes:

1. The client's connection drops; the client attempts to reconnect.
2. It asks the session directory "where do I go now?"
3. The directory routes it to a **new healthy pod** (either the rescheduled
   instance, or a different one that has capacity).
4. That pod **loads the player's last known state from the cache** (Pillar 1)
   and the player is back in the match.

This is necessary because a game server is **not** behind a normal round-robin
Service the way a stateless web app is. You can't send player alice to "any"
game pod — she has to land on a pod that either already hosts her match or can
rehydrate it. The session directory is the component that makes the redirect
deterministic instead of random.

This is also where **stable identity** (StatefulSet) or an **allocation model**
(Agones — see below) comes in: they make the "which pod hosts this match"
question answerable and stable across restarts.

Why it matters for player experience: without a directory, the reconnecting
client has no idea where to go, and load balancing would scatter it to a pod
with no knowledge of its match. With one, reconnect is a targeted redirect to a
pod that can immediately rehydrate.

---

## Pillar 3 — Action-Replay / Sync Logs (Closing the Staleness Gap)

**Rule: buffer the player's recent actions so you can replay them on top of the
restored snapshot and erase the gap between "last snapshot" and "moment of
crash."**

The problem: snapshots are periodic, so the restored snapshot is stale by up to
one snapshot interval. If you restore *only* the snapshot, the player is rolled
back those few seconds — they lose the shot they fired, the door they opened,
the ground they gained. That rollback is jarring and is exactly what ruins the
feel of a recovery.

The fix — an **append-only action log** (a stream of the player's inputs/events)
that is written far more frequently than snapshots, ideally on **every tick**:

```
snapshot @ tick 12          <- periodic, stale
actions:  seq13, seq14      <- fine-grained, up to the moment of crash
```

On recovery, the replacement pod:

1. Loads the snapshot (tick 12).
2. **Replays** the buffered actions after tick 12 (seq13, seq14) deterministically
   on top of it.
3. Arrives at the state as of the crash — the gap is closed.

The result: the crash becomes a **minor blip** — a short reconnect plus a tiny,
fast resync — rather than a visible rollback or a lost match. This is the
piece that turns "acceptable" recovery into "the player barely noticed."

**Critical correctness concern — idempotency.** Replay must not double-apply an
action (e.g. "fire weapon" applied twice, or a coin credited twice). You make
actions **idempotent via monotonic sequence numbers**: each action carries a
`seq`, the state tracks the last-applied `seq`, and replay skips anything
`<= last_applied`. This makes replay safe even if it overlaps with actions that
were already partially applied before the snapshot. (See `deeper.md`.)

---

## Why Vanilla Kubernetes Is Hostile to This Workload

This is the part interviewers most want to hear, because it shows you understand
Kubernetes' *assumptions*, not just its API.

Kubernetes' core workload primitives assume pods are **stateless, fungible, and
request/response**:

- **Deployment**: pods are interchangeable replicas with **random names** that
  change on every restart. There is no notion of "this replica owns that
  match."
- **HorizontalPodAutoscaler (HPA)**: scales replicas up/down based on
  CPU/metrics. When it scales *down*, it **picks a pod and kills it** — with no
  awareness that the pod is hosting an active match. That's a lost match.
- **Liveness probes**: if a probe fails (e.g. the server is busy in a long game
  tick and slow to respond), Kubernetes **restarts the container**, killing the
  match. Liveness probes assume "unresponsive == broken," which is often false
  for a busy game server.
- **Rolling updates**: a Deployment rollout terminates old pods to bring up new
  ones — again, evicting active matches on a schedule that has nothing to do
  with when matches end.

The common thread: **none of these primitives model "this pod is hosting an
active match, do not evict it."** They're designed to treat pods as cattle.

### StatefulSet vs Deployment

A **StatefulSet** fixes *some* of this:

- **Stable network identity:** `gameserver-0`, `gameserver-1`, ... each with a
  **stable DNS name** via a headless Service. A re-created pod keeps its
  identity, so a session directory mapping "match -> gameserver-0" stays valid.
- **Stable per-pod storage:** each ordinal gets its own PersistentVolumeClaim
  that is re-attached when the pod is rescheduled — useful if a pod persists
  local state to disk.
- **Ordered, controlled rollout:** pods are updated one at a time in order,
  rather than churned in bulk.

Use a **StatefulSet** when sessions are **pinned to a specific instance** and
you benefit from stable identity/storage. Use a **Deployment** for the
stateless supporting services (the session directory frontend, matchmaker API,
etc.) that *are* fungible.

**But** StatefulSets still don't understand "active match." A StatefulSet
rollout or scale-down will still terminate a pod that's hosting a live game. It
gives you *identity and ordering*, not *match-aware lifecycle*.

### Why Agones (a Dedicated Game-Server Orchestrator)

This is why modern real-time game fleets run **Agones** on top of Kubernetes.
Agones adds game-server-native concepts that plain Kubernetes lacks:

- **`GameServer` CRD:** models a single dedicated game-server process with a
  **lifecycle that includes game states** — notably `Ready` (idle, available)
  vs **`Allocated`** (actively hosting a match). This is the missing concept:
  Kubernetes has no idea a pod is "in a match"; Agones does.
- **`Fleet`:** a set of warm `GameServer`s kept `Ready` so a match can be
  assigned instantly (players don't wait for a cold pod to boot).
- **Allocation model:** the matchmaker asks Agones to **allocate** a `Ready`
  GameServer for a new match; Agones marks it `Allocated`. Crucially, Agones
  and its integrations **protect `Allocated` game servers from being scaled
  down or disrupted** — the fleet autoscaler and drains skip busy matches and
  only reclaim `Ready` ones.
- **SDK:** the game binary calls the Agones SDK (`Ready()`, `Allocate()`,
  `Health()`, `Shutdown()`) so the server itself signals its true state, rather
  than Kubernetes guessing from a generic liveness probe.

In short: **Agones exists precisely because Deployments/HPA/liveness/rolling
updates assume fungible stateless pods and will happily evict an active match.**
Agones teaches the platform the concept "this pod is busy hosting players."

---

## Graceful Shutdown Mechanics

The three recovery pillars handle the crashes you *can't* prevent. Graceful
shutdown handles the terminations you *can* control (rollouts, node drains,
scale-down) so they cost even less than a crash.

### The lifecycle and ordering

When Kubernetes decides to terminate a pod:

1. The pod is marked `Terminating` and **removed from Service endpoints** (it
   stops receiving new traffic).
2. If a **`preStop` hook** is defined, it runs **first, to completion**.
3. **Only after `preStop` finishes**, Kubernetes sends **`SIGTERM`** to the
   container's main process (PID 1).
4. The app has until **`terminationGracePeriodSeconds`** (default 30s) — a
   budget that covers **both** the `preStop` hook **and** the post-SIGTERM
   window — to shut down cleanly.
5. If that budget elapses, Kubernetes sends **`SIGKILL`** and the pod dies
   abruptly.

### What each piece does for a game server

- **`preStop` hook:** the pre-death cleanup that runs before the signal. Use it
  to **flush a final, fresh snapshot to the cache** and **mark the instance as
  draining** in the session directory (stop assigning new players / matches to
  it).
- **Handle `SIGTERM` in the app:** the server must trap `SIGTERM` and, instead
  of dying instantly, **drain** — finish the current game tick, tell connected
  clients "reconnect elsewhere," flush remaining state, then exit. If you don't
  handle SIGTERM, the process may die immediately (or only at SIGKILL), wasting
  the grace period.
- **`terminationGracePeriodSeconds`:** must be **long enough** for the server to
  finish a tick, snapshot, and notify clients — but not so long that rollouts
  crawl. For game servers you often set this generously (tens of seconds to
  minutes) because you may want to wait for the *current match to reach a safe
  boundary*.

### Crash vs graceful drain — why the distinction matters

- **Crash (involuntary):** no `preStop`, no `SIGTERM` window. The latest
  snapshot is whatever was last written periodically — **stale**. Recovery needs
  the full snapshot + action-replay dance (Pillar 3).
- **Graceful drain (voluntary):** `preStop` flushes a **fresh** snapshot and the
  app drains cleanly. The restored state is essentially **current** — near-zero
  rollback, minimal or no replay needed.

So graceful shutdown turns a "reconnect + resync" into "reconnect only" for
every disruption you initiate. That's a strictly better player experience, and
it's entirely within your control. (Best of all: with an Agones-style model you
can often avoid the disruption during an active match altogether.)

---

## PodDisruptionBudget (PDB)

A **PDB** limits how many pods of a workload can be taken down by **voluntary**
disruptions at once. You declare either `minAvailable` (keep at least N running)
or `maxUnavailable` (take down at most N). The **eviction API** enforces it:
`kubectl drain` and cluster-autoscaler use eviction, and eviction is **refused**
if it would violate the budget.

For game servers the value is obvious: during a **node drain or cluster
upgrade**, without a PDB the drain would evict *every* game pod on that node
simultaneously — potentially killing many active matches at once. With
`minAvailable` set appropriately, the drain is forced to proceed **slowly, one
pod at a time**, waiting for rescheduling and never dropping below the budget.
Combined with graceful shutdown, each evicted pod drains cleanly before the next
is touched.

**The crucial limitation:** a PDB governs **only voluntary disruptions**. It
does **nothing** for involuntary ones — OOM kills, kernel panics, node hardware
failure, or a crashing process bypass the eviction API entirely. A node that
dies takes its pods with it regardless of any PDB.

---

## Voluntary vs Involuntary Disruptions — the Defense Matrix

| Disruption | Class | Example | Primary defense |
|-----------|-------|---------|-----------------|
| Node drain / cluster upgrade | Voluntary | `kubectl drain`, rolling node pool | **PDB** (rate-limit) + **graceful shutdown** (fresh snapshot) |
| Deployment rollout | Voluntary | new image version | **Graceful shutdown** + Agones (avoid disrupting `Allocated`) |
| Autoscaler scale-down | Voluntary | HPA/cluster-autoscaler removing capacity | **PDB** + Agones (never reclaim active matches) |
| Process crash / bug | Involuntary | segfault, unhandled panic | **Externalized state + action replay** (Pillars 1 & 3) |
| OOM kill | Involuntary | memory leak / spike | **Externalized state + action replay**; also set memory limits/requests |
| Node hardware failure | Involuntary | kernel panic, hardware fault | **Externalized state + session redirect** to a pod on a healthy node |

The takeaway to deliver: **you need both families of defense.** The three
pillars keep involuntary crashes survivable; graceful shutdown + PDB keep
voluntary disruptions cheap and controlled. Neither alone is sufficient — a PDB
won't save you from an OOM, and a snapshot cache won't stop a drain from
nuking your whole fleet at once.

---

## Putting It All Together — The Complete Answer

1. **Externalize state** to a fast cache (Redis) with periodic snapshots so a
   replacement pod can rehydrate instead of starting from zero.
2. **Run a session directory** so a reconnecting client is redirected to a
   healthy pod that can load that player's state.
3. **Buffer recent actions** and **replay** them on the restored snapshot to
   close the staleness gap — making replay **idempotent via sequence numbers**.
4. **Handle SIGTERM + use a `preStop` hook + a generous
   `terminationGracePeriodSeconds`** so planned terminations flush a fresh
   snapshot and tell clients to reconnect — turning drains into near-seamless
   handoffs.
5. **Use a PodDisruptionBudget** to rate-limit voluntary disruptions so upgrades
   and drains never evict a whole fleet of active matches at once.
6. **Prefer a game-aware orchestrator (Agones)** — or at least a StatefulSet for
   stable identity — over plain Deployments + HPA, because vanilla Kubernetes
   treats pods as fungible and will evict an active match without a second
   thought.

Net effect: a crashing game server pod becomes a **short reconnect and a tiny
resync**, not a ruined match.

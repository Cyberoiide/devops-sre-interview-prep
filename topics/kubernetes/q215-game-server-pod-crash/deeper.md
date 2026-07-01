# q215 — Deeper: Follow-Up Interview Questions

Concise, correct answers to the follow-ups an interviewer is likely to fire
after your main answer.

---

### 1. Deployment or StatefulSet — when does each make sense for game servers?

- **Deployment**: use for the **stateless, fungible** parts of the platform —
  the matchmaker/session-directory API, web frontends, metrics exporters. Pods
  have random names that change on restart and are interchangeable behind a
  Service. Fine when "any replica can serve any request."
- **StatefulSet**: use when a session is **pinned to a specific instance** and
  you benefit from **stable network identity** (`gameserver-0` with a stable DNS
  name via a headless Service) and **stable per-pod storage** (each ordinal
  keeps its own PVC across reschedules), plus **ordered, one-at-a-time
  rollouts**.
- **Honest caveat**: a StatefulSet gives you identity and ordering, **not
  match-awareness**. A rollout or scale-down will still terminate a pod hosting
  a live match. For real-time fleets, that's why Agones (next question) usually
  beats a raw StatefulSet.

---

### 2. What is Agones and why not just use Kubernetes?

**Agones** is an open-source, game-server-native orchestrator that runs on top
of Kubernetes. It adds concepts Kubernetes lacks:

- A **`GameServer` CRD** whose lifecycle includes game-specific states —
  especially **`Ready`** (idle/available) vs **`Allocated`** (actively hosting a
  match).
- A **`Fleet`** of warm `Ready` game servers so matches start instantly.
- An **allocation model**: the matchmaker allocates a `Ready` server for a new
  match; Agones marks it `Allocated` and **protects `Allocated` servers from
  scale-down and disruption**.
- An **SDK** (`Ready()`, `Allocate()`, `Health()`, `Shutdown()`) so the game
  binary reports its true state instead of Kubernetes guessing via a generic
  liveness probe.

**Why not just Kubernetes:** Deployments, HPA, liveness probes, and rolling
updates all assume pods are **stateless and fungible**. They have no concept of
"this pod is hosting an active match, don't evict it." HPA scale-down or a
rollout will kill a live match; a slow game tick can trip a liveness probe and
trigger a restart. Agones teaches the platform "busy vs idle," which is exactly
the missing concept.

---

### 3. How do you avoid replaying an action twice?

Make replay **idempotent** using **monotonic sequence numbers**:

- Every action carries a strictly increasing `seq` (per player/session).
- The authoritative state records the **last-applied `seq`**.
- On replay, apply an action only if its `seq > last_applied`; skip anything
  already applied.

This makes replay safe even if the buffered log overlaps actions already folded
into the snapshot, or if recovery runs more than once. Without sequence numbers
you risk double-applying effects — firing a weapon twice, crediting a coin
twice, moving a player two steps for one input. (Deduplication by unique action
ID is an equivalent alternative; sequence numbers also give you ordering for
free.)

---

### 4. What if the cache (Redis) itself fails?

The cache is now a single point of failure for recovery, so you harden it:

- **Run it HA**, not as a single pod: **Redis Sentinel** (automatic failover
  between primary/replicas) or **Redis Cluster** (sharded + replicated), or a
  managed HA offering.
- **Enable durability**: Redis **AOF** (append-only file) and/or **RDB**
  snapshots so a full restart doesn't lose everything, backed by a
  PersistentVolume or managed persistence.
- **Layer the state**: recent/hot state in Redis for speed, periodic durable
  checkpoints to an object store or database behind it, so a total cache loss
  still lets you restore from the last durable checkpoint (with more rollback).
- **Degrade gracefully**: if the cache is briefly unavailable, the game server
  can keep its in-memory state authoritative and buffer writes, accepting that a
  crash during that window loses more. The cache reduces the blast radius of a
  pod crash; you must not let it *become* a bigger blast radius.

---

### 5. How does `terminationGracePeriodSeconds` interact with the game tick?

`terminationGracePeriodSeconds` is the total budget (covering the `preStop` hook
**and** the post-`SIGTERM` window) before `SIGKILL`. It must be **long enough**
for the server to reach a **safe boundary**:

- On `SIGTERM`, a well-behaved server finishes the **current tick** (so state is
  internally consistent), flushes a final snapshot, and tells clients to
  reconnect — rather than dying mid-tick with half-applied state.
- Set the grace period comfortably larger than one tick plus flush time. If it's
  too short, `SIGKILL` interrupts mid-tick and you're back to a crash-quality
  (stale, possibly inconsistent) recovery.
- If it's too long, rollouts and drains crawl. Some studios set it to span the
  time needed to reach a natural match boundary; others drain at the next tick
  boundary and notify clients. The right value is "one safe checkpoint boundary
  plus flush," not an arbitrary default.

Also note: `preStop` runs **before** `SIGTERM` and eats into the same budget, so
account for both.

---

### 6. How do you handle node failure vs a process crash differently?

Both are **involuntary**, but the recovery path differs:

- **Process crash** (segfault, panic, OOM): the **node is healthy**, so the
  Deployment/StatefulSet/Agones fleet reschedules a replacement quickly, often
  on the **same node**. Fast rehydrate from the local-region cache; short blip.
  For OOM specifically, also fix the root cause — set correct memory
  requests/limits and profile the leak, or it recurs.
- **Node failure** (kernel panic, hardware fault, network partition): the node
  and *all* its pods vanish at once. Kubernetes only notices after the node
  goes `NotReady` and a timeout elapses, so **detection and rescheduling are
  slower**. Replacements must land on **other healthy nodes**, and the session
  directory redirects affected players there. Defenses: spread pods across nodes
  and zones (anti-affinity / topology spread) so one node's death doesn't take a
  whole game region, and keep the cache HA and off the failed node.

In both cases the externalized state + session redirect + action replay is what
saves the match; the difference is *recovery latency* and *blast radius*, which
you manage with spreading and fast failure detection.

---

### 7. How does a PDB help during a cluster upgrade?

A cluster upgrade drains nodes one by one, and `kubectl drain` uses the
**eviction API**, which **respects PodDisruptionBudgets**. With a PDB
(`minAvailable`/`maxUnavailable`), the drain is **forced to evict pods slowly**,
never dropping the workload below the budget — so an upgrade can't nuke a whole
fleet of active matches at once. It waits for each evicted pod to be rescheduled
and healthy before taking the next. Combined with **graceful shutdown**, each
evicted game server drains cleanly (fresh snapshot, clients told to reconnect)
before the next node is touched. Without a PDB, the upgrade evicts everything on
each node immediately and you lose every match on it.

**Caveat to state:** the PDB only constrains **voluntary** disruptions like the
drain. If a node dies *during* the upgrade (involuntary), the PDB doesn't help —
that's what the state/replay pillars are for.

---

### 8. Why not just use a liveness probe to restart a crashed game server?

A liveness probe **restarts the container** when it fails — which for a game
server means **killing the match**. Two problems:

1. **False positives**: a busy server deep in a heavy game tick may be slow to
   answer the probe. Kubernetes reads "unresponsive" as "broken" and restarts a
   perfectly healthy match. That's a self-inflicted outage.
2. **It doesn't preserve state**: even a legitimate restart via liveness is just
   a crash from the match's perspective — memory is gone.

Better: use liveness conservatively (generous thresholds), rely on the app's own
health signal (e.g. the Agones SDK `Health()` heartbeat, which is designed for
this), and make survival depend on externalized state + replay rather than on
"restart and hope." Restarting is not recovery.

---

### 9. Where do WebSocket/UDP connections fit — why is a reconnect even needed?

Real-time games hold **long-lived stateful connections** (often UDP for latency,
or WebSocket/TCP). When the pod dies, that connection dies with it — there is no
way for the socket to "follow" the state to a new pod, because the transport is
tied to the pod's network endpoint. So the client **must reconnect**, and the
session directory tells it **where**. This is fundamentally different from a
stateless HTTP request, which a Service can transparently load-balance to any
replica. It's the core reason the whole session-directory + redirect machinery
exists: you can't hide a game-server pod behind a plain round-robin Service and
pretend nothing happened.

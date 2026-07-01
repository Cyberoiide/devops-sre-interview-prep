# q215 — Hands-On Lab: Surviving a Game Server Pod Crash

Running a real game engine in a lab is impractical, so we **simulate** one. The
essential game-server behavior we care about is:

1. It holds **per-player state** (score, position, inventory) that would be lost
   if the pod's memory vanished.
2. It receives a **stream of player actions** (inputs/events).
3. On crash, a replacement must **rehydrate** the last snapshot and **replay**
   the buffered actions to catch up.

We model all of that with **Redis** (external state store) and tiny `redis-cli`
loops running in standard `redis` / `busybox` images — **no custom image
builds**. That keeps the lab focused on the Kubernetes and state-recovery
mechanics rather than on game code.

---

## Prerequisites

```bash
# Any one of these clusters works. Examples use kind.
kind create cluster --name gameservers
# or: minikube start

kubectl config current-context   # confirm you're pointed at the lab cluster
kubectl get nodes
```

All resources go into a dedicated namespace so cleanup is trivial:

```bash
kubectl create namespace game
kubectl config set-context --current --namespace=game
```

---

## Part 1 — Externalize State: Deploy Redis

This is our "in-memory data store for fast state retrieval." In production you'd
use a managed/HA Redis (or a persistent snapshot store behind it); here a single
pod is enough to demonstrate the pattern.

`redis.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: game
spec:
  replicas: 1
  selector:
    matchLabels: { app: redis }
  template:
    metadata:
      labels: { app: redis }
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: game
spec:
  selector: { app: redis }
  ports:
    - port: 6379
      targetPort: 6379
```

```bash
kubectl apply -f redis.yaml
kubectl rollout status deploy/redis
```

Sanity check:

```bash
kubectl run redis-probe --rm -it --restart=Never --image=redis:7-alpine -- \
  redis-cli -h redis ping
# -> PONG
```

---

## Part 2 — The "Game Server" Pod: Snapshots + Action Buffer

Our simulated game server does two things in a loop against Redis:

- Every few seconds it writes a **snapshot** of player state
  (`player:alice:snapshot`) — this is intentionally *periodic*, so it is always
  a little stale.
- On **every tick** it appends the player's action to an **action log**
  (`player:alice:actions`, a Redis list) with a monotonically increasing
  sequence number — this is the fine-grained stream we replay to close the gap.

`gameserver-deploy.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gameserver
  namespace: game
spec:
  replicas: 1
  selector:
    matchLabels: { app: gameserver }
  template:
    metadata:
      labels: { app: gameserver }
    spec:
      # Short grace period for Part 1-3; we raise it in Part 4.
      terminationGracePeriodSeconds: 5
      containers:
        - name: gameserver
          image: redis:7-alpine          # gives us redis-cli + sh, no build
          command: ["/bin/sh", "-c"]
          args:
            - |
              PLAYER=alice
              echo "[gameserver] $(hostname) starting; rehydrating state..."

              # --- REHYDRATE on startup ---
              SNAP=$(redis-cli -h redis get player:$PLAYER:snapshot)
              if [ -n "$SNAP" ]; then
                echo "[gameserver] restored snapshot: score=$SNAP"
                # Replay any actions newer than the snapshot (see Part 3).
                echo "[gameserver] replaying buffered actions..."
                redis-cli -h redis lrange player:$PLAYER:actions 0 -1
              else
                echo "[gameserver] no prior state; new player"
                redis-cli -h redis set player:$PLAYER:snapshot 0
              fi

              # --- MAIN LOOP: tick, append action, periodic snapshot ---
              TICK=0
              while true; do
                TICK=$((TICK+1))
                # every tick: append an action with a sequence number
                redis-cli -h redis rpush player:$PLAYER:actions "seq=$TICK move=+1" >/dev/null
                # every 3rd tick: write a fresh (still slightly stale) snapshot
                if [ $((TICK % 3)) -eq 0 ]; then
                  redis-cli -h redis set player:$PLAYER:snapshot $TICK >/dev/null
                  echo "[gameserver] snapshot @ tick $TICK"
                fi
                sleep 1
              done
```

```bash
kubectl apply -f gameserver-deploy.yaml
kubectl rollout status deploy/gameserver
kubectl logs -f deploy/gameserver &   # watch it tick; Ctrl-C to stop following
```

Let it run ~15 seconds so it builds up state, then inspect Redis:

```bash
kubectl exec deploy/redis -- redis-cli get  player:alice:snapshot   # e.g. "12"
kubectl exec deploy/redis -- redis-cli llen player:alice:actions    # e.g. 14 (2 ticks ahead of snapshot)
```

Note the **staleness gap**: the snapshot is at tick 12 but the action log has 14
entries. The snapshot alone would roll the player back two ticks; the extra
actions are exactly what we replay to avoid that.

---

## Part 3 — Simulate the Crash and Observe Recovery

A crash is any abrupt loss of the pod. We force one by deleting it (equivalent
in effect to an OOM kill or node loss: no chance to flush a final snapshot).

```bash
# Record state right before the crash
kubectl exec deploy/redis -- redis-cli get  player:alice:snapshot
kubectl exec deploy/redis -- redis-cli llen player:alice:actions

# CRASH: kill the pod abruptly (no graceful drain)
kubectl delete pod -l app=gameserver --grace-period=0 --force
```

The Deployment immediately schedules a replacement. Watch the new pod's logs:

```bash
kubectl logs -f deploy/gameserver
```

You will see it:

1. `restored snapshot: score=<N>` — it rehydrated the last snapshot from Redis
   instead of starting from zero.
2. `replaying buffered actions...` followed by the action list — it replays the
   inputs that happened after the snapshot, closing the staleness gap.

**Key takeaway to state in the interview:** because state lived in Redis and not
only in pod memory, the crash cost the player a short reconnect plus a tiny
resync — *not* a lost match. Without Redis, the new pod would have started the
player at score 0.

---

## Part 4 — Graceful Shutdown: preStop + terminationGracePeriodSeconds

A **crash** is involuntary and gives you no chance to flush. A **planned**
termination (rollout, scale-down, node drain) is *voluntary* — Kubernetes sends
`SIGTERM` first and waits `terminationGracePeriodSeconds` before `SIGKILL`. We
use that window to flush a **fresh** snapshot and tell clients to reconnect
elsewhere.

**Ordering (important for the interview):** when a pod is told to terminate,
Kubernetes runs the `preStop` hook **first**, and *only after it completes* sends
`SIGTERM` to the container's main process (PID 1). `terminationGracePeriodSeconds`
is the total budget covering **both** the `preStop` hook **and** the post-SIGTERM
drain; if it elapses, the container gets `SIGKILL`.

`gameserver-graceful.yaml` (replaces the Part 2 Deployment):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gameserver
  namespace: game
spec:
  replicas: 1
  selector:
    matchLabels: { app: gameserver }
  template:
    metadata:
      labels: { app: gameserver }
    spec:
      terminationGracePeriodSeconds: 30      # generous drain budget
      containers:
        - name: gameserver
          image: redis:7-alpine
          lifecycle:
            preStop:
              exec:
                # Runs BEFORE SIGTERM. Flush a fresh snapshot + mark "draining"
                # so the session directory stops sending new players here.
                command:
                  - /bin/sh
                  - -c
                  - |
                    redis-cli -h redis set player:alice:draining 1
                    LAST=$(redis-cli -h redis llen player:alice:actions)
                    redis-cli -h redis set player:alice:snapshot $LAST
                    echo "[preStop] flushed fresh snapshot @ $LAST, marked draining"
          command: ["/bin/sh", "-c"]
          args:
            - |
              PLAYER=alice
              # Trap SIGTERM so the app can react (drain in-flight ticks).
              trap 'echo "[gameserver] SIGTERM: draining, will exit"; exit 0' TERM
              SNAP=$(redis-cli -h redis get player:$PLAYER:snapshot)
              echo "[gameserver] $(hostname) up; restored snapshot=$SNAP"
              TICK=${SNAP:-0}
              while true; do
                TICK=$((TICK+1))
                redis-cli -h redis rpush player:$PLAYER:actions "seq=$TICK move=+1" >/dev/null
                [ $((TICK % 3)) -eq 0 ] && redis-cli -h redis set player:$PLAYER:snapshot $TICK >/dev/null
                sleep 1 &
                wait $!          # `wait` lets the TERM trap fire promptly
              done
```

```bash
kubectl apply -f gameserver-graceful.yaml
kubectl rollout status deploy/gameserver
sleep 10
```

Now trigger a **graceful** termination and compare against the Part 3 crash:

```bash
# Graceful: Kubernetes runs preStop, THEN sends SIGTERM, THEN waits up to 30s.
POD=$(kubectl get pod -l app=gameserver -o name)
kubectl delete $POD          # default grace period; preStop runs

# Inspect: snapshot should equal the action-log length (no staleness gap)
kubectl exec deploy/redis -- redis-cli get  player:alice:snapshot
kubectl exec deploy/redis -- redis-cli llen player:alice:actions
```

**Compare the two paths:**

| Path | What happened | State freshness |
|------|---------------|-----------------|
| Part 3 crash (`--grace-period=0 --force`) | No preStop, no SIGTERM window | Snapshot lags the action log — replay needed |
| Part 4 graceful (`kubectl delete`) | preStop flushed, SIGTERM drained | Snapshot == action log — no gap, near-zero rollback |

This is the concrete demonstration of *why graceful shutdown matters*: for the
disruptions you control, you convert a "reconnect + resync" into a
"reconnect only."

---

## Part 5 — StatefulSet Variant (Stable Identity)

If sessions are *pinned* to a specific instance, you want a **stable network
identity** (a fixed DNS name that survives a restart) and optionally
**stable per-pod storage**. That's what a StatefulSet provides:
`gameserver-0` always resolves to the same stable DNS record via a headless
Service, and its PersistentVolumeClaim is re-attached to the re-created pod.

`gameserver-sts.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gameserver-hl
  namespace: game
spec:
  clusterIP: None          # headless -> stable per-pod DNS
  selector: { app: gameserver-sts }
  ports:
    - port: 6379
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: gameserver-sts
  namespace: game
spec:
  serviceName: gameserver-hl
  replicas: 2
  selector:
    matchLabels: { app: gameserver-sts }
  template:
    metadata:
      labels: { app: gameserver-sts }
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: gameserver
          image: redis:7-alpine
          command: ["/bin/sh", "-c"]
          args:
            - |
              # Each pod owns a session keyed by its STABLE ordinal identity.
              ME=$(hostname)                       # gameserver-sts-0 / -1
              echo "[$ME] stable identity; hosting session $ME"
              redis-cli -h redis set session:$ME:host $ME
              while true; do
                redis-cli -h redis incr session:$ME:tick >/dev/null
                sleep 2
              done
```

```bash
kubectl apply -f gameserver-sts.yaml
kubectl rollout status statefulset/gameserver-sts

# Stable identity survives a restart: same name, same DNS, session key intact.
kubectl exec deploy/redis -- redis-cli get session:gameserver-sts-0:host   # gameserver-sts-0
kubectl delete pod gameserver-sts-0
kubectl wait --for=condition=Ready pod/gameserver-sts-0 --timeout=60s
kubectl exec deploy/redis -- redis-cli get session:gameserver-sts-0:host   # still gameserver-sts-0
```

The point: `gameserver-sts-0` came back with the **same identity**, so a session
directory that mapped "player alice -> gameserver-sts-0" still routes correctly.
A Deployment gives you random pod names (`gameserver-7d9f...`) that change on
every restart, which is why plain Deployments need an external session directory
to track *where* a player lives.

> Note: This lab keeps the StatefulSet stateless-to-disk for simplicity. In a
> real StatefulSet you'd add a `volumeClaimTemplates:` block for stable
> per-pod storage.

---

## Part 6 — PodDisruptionBudget vs `kubectl drain`

A **PodDisruptionBudget (PDB)** caps *voluntary* disruptions. During a node
drain (upgrade, maintenance), the eviction API refuses to evict a pod if doing
so would push the workload below the PDB's `minAvailable`. This stops an
operator or autoscaler from evicting a whole fleet of active-match pods at once.

`pdb.yaml`:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: gameserver-pdb
  namespace: game
spec:
  minAvailable: 2                 # never let voluntary eviction drop below 2
  selector:
    matchLabels: { app: gameserver-sts }
```

```bash
kubectl apply -f pdb.yaml
kubectl get pdb gameserver-pdb
# ALLOWED DISRUPTIONS should be 0 (we have 2 pods, minAvailable=2)
```

Demonstrate the PDB blocking a mass eviction. On a single-node kind cluster,
draining the only node tries to evict both StatefulSet pods:

```bash
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')

# --dry-run first to see the plan without touching anything
kubectl drain "$NODE" --ignore-daemonsets --delete-emptydir-data --dry-run=client

# Real drain with a short timeout so it doesn't hang forever.
kubectl drain "$NODE" --ignore-daemonsets --delete-emptydir-data --timeout=30s
```

The drain **stalls / errors** with a message like
`Cannot evict pod as it would violate the pod's disruption budget` for the
gameserver pods — the PDB refused to let both active-match pods die at once.
Without the PDB, both would be evicted immediately. (Uncordon afterward:
`kubectl uncordon "$NODE"`.)

> On a multi-node cluster this is more realistic: the drain evicts pods one at a
> time, waiting for rescheduling, and only as long as `minAvailable` stays
> satisfied — exactly the "roll the fleet slowly, never nuke all active matches"
> behavior you want during a cluster upgrade.

**Interview framing:** a PDB does **nothing** for involuntary disruptions (OOM,
kernel panic, node hardware failure) — those bypass the eviction API entirely.
It only governs *voluntary* disruptions. That's why you need *both* the
state-externalization/replay strategy (for involuntary crashes) **and** the PDB
(for planned drains/upgrades).

---

## Cleanup

```bash
kubectl delete namespace game
# and, if you spun up a throwaway cluster:
kind delete cluster --name gameservers
# or: minikube delete
```

---

## What This Lab Proved

- State in Redis (not pod memory) means a crash → rehydrate, not restart-at-zero.
- The **action log** closes the snapshot **staleness gap** on recovery.
- **Graceful** shutdown (`preStop` before `SIGTERM`, bounded by
  `terminationGracePeriodSeconds`) flushes a *fresh* snapshot — a strictly
  better outcome than a crash, for the disruptions you control.
- A **StatefulSet** gives stable identity/DNS so a re-created pod keeps its
  session mapping; a Deployment does not.
- A **PodDisruptionBudget** blocks mass *voluntary* eviction during drains but
  is powerless against involuntary crashes — hence you need both defenses.

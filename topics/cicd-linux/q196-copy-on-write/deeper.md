# Deeper — Interviewer Follow-up Questions

Concise, correct answers to the pushbacks that separate memorizers from people
who understand how CoW is actually implemented.

### Q: Walk me through exactly what happens on the first write to a CoW page.

The page-table entry for that page is marked **read-only**, so the CPU's MMU
raises a **page fault** and traps into the kernel. The fault handler recognizes a
CoW page (the VMA says the page *should* be writable, but the PTE was deliberately
marked read-only, and its refcount is > 1). It allocates a new physical page,
copies the shared page's contents into it, repoints *this* process's PTE at the
new page and marks it writable, drops the old page's refcount, and **restarts the
faulting instruction** — which now writes to the private copy. Only that one page
is copied.

### Q: Does fork() copy anything at all, then?

Yes — the cheap stuff. It duplicates the process's metadata and page-table
structures (which are small relative to the mapped data) and marks writable pages
read-only in both processes. It does **not** copy the actual data pages; those are
shared CoW. So the cost of `fork()` scales with the size of the page tables, not
the size of the resident memory. (Modern kernels optimize even the page-table
duplication.)

### Q: Why is fork()+exec() the case that makes CoW pay off so much?

Because the child throws its inherited address space away almost immediately.
`exec()` replaces the process image with a brand-new program, discarding all the
parent's pages. If `fork()` had eagerly copied gigabytes, every one of those bytes
would be copied and then instantly freed — pure waste. With CoW the child barely
writes anything before `exec()`, so almost no pages are ever actually copied. This
is the overwhelmingly common pattern: shells launching commands, servers spawning
workers.

### Q: If fork() shares pages, how does the child avoid corrupting the parent?

CoW isolation. The moment either process *writes* a shared page, that writer gets
its own private copy and the other keeps the original. Reads stay shared; writes
diverge. So the two address spaces are logically independent even though they
physically share every unwritten page. The lab's `parent-after` line proves it:
the parent's data is untouched after the child overwrites its whole buffer.

### Q: My parent and child each show 256 MB RSS in `top`, but the box only has 256 MB used for them. Why?

`ps`/`top` **RSS double-counts shared pages** — each process is credited with the
full shared region even though there's one physical copy. The accurate metric is
**Pss (Proportional Set Size)** in `/proc/<pid>/smaps` / `smaps_rollup`, which
divides each shared page's size by the number of sharers. Sum the two Pss values
and you get the real ~256 MB. Always use Pss (not RSS) for capacity planning on
forked or shared-memory workloads.

### Q: What's the connection between CoW and memory overcommit?

CoW (and demand paging) mean allocated-but-unwritten pages cost **zero** physical
RAM. That's what makes overcommit safe *enough* to be the default: the kernel
hands out more virtual memory than it has, betting most of it won't be written. A
`fork()` of a 4 GB process instantly "allocates" 4 GB more of address space while
backing almost none of it. The policy is `/proc/sys/vm/overcommit_memory`
(`0`=heuristic default, `1`=always, `2`=strict cap at
`swap + RAM*overcommit_ratio%`).

### Q: And what does the OOM killer have to do with it?

Overcommit is a promise the kernel might not be able to keep. If enough processes
eventually write their promised (CoW or freshly allocated) pages and physical RAM
plus swap runs out, there's no graceful way to fail a store instruction — so the
kernel invokes the **OOM killer** to free RAM by killing a process. It scores
candidates (roughly by memory footprint, adjustable per process via
`/proc/<pid>/oom_score_adj`) and kills the highest. So the chain is: CoW/demand
paging make pages cheap → Linux overcommits → sometimes the promises can't be met
→ OOM killer.

### Q: How does a fork bomb exploit this?

`fork()` is cheap precisely because CoW avoids copying. That cheapness lets a
process spawn children faster than the system can meaningfully account for them;
each fork is nearly free, so `:(){ :|:& };:` explodes exponentially and exhausts
PIDs / process-table slots / memory almost instantly. The defense isn't disabling
CoW — it's limits: `RLIMIT_NPROC` (`ulimit -u`), cgroup `pids.max`, and
systemd `TasksMax`.

### Q: Is CoW only a memory thing, or does it show up elsewhere?

Same idea, different layer. **btrfs and ZFS** are CoW filesystems: a write goes to
a new block and metadata is repointed rather than overwriting in place, which
makes **snapshots** nearly free (share all blocks, diverge only on new writes).
**overlayfs** and **Docker/OCI image layers** use CoW too: containers share
read-only image layers and a write copies the file up into a writable layer
(`copy_up`). It's the same "share read-only, copy on first write" trade at
block/file granularity instead of memory-page granularity.

### Q: Are there downsides or surprises with CoW?

A few worth naming:
- **Latency spikes:** the copying isn't free, it's *deferred*. A workload that
  forks a huge process and then writes broadly (e.g. a garbage collector touching
  many pages, or Redis during a background save) pays a burst of page faults and
  copies right when it starts writing — latency, not saved, just moved.
- **Memory surprise:** a `fork()`ed child that looked cheap can suddenly balloon
  in RSS as it writes and splits pages, potentially triggering OOM well after the
  fork "succeeded."
- **Accounting confusion:** the RSS double-counting above misleads monitoring.
- **Reference-counted languages defeat it:** in CPython, merely *reading* an
  object touches its refcount header (a write), so forked pages get copied anyway
  — which is why `fork()`-based preforking servers don't share Python heap memory
  as well as you'd hope.

### Q: How would you *observe* CoW on a running system?

`/proc/<pid>/smaps` (or the summarized `smaps_rollup`) breaks each mapping into
`Shared_Clean`, `Shared_Dirty`, `Private_Clean`, `Private_Dirty`, and `Pss`.
Pages shared CoW show up as `Shared_*` with a fractional `Pss`; after a write they
flip to `Private_Dirty` and `Pss` rises to the full amount. Watching that flip
across a `fork()`+write (exactly the lab) is the most direct way to *see* CoW
happen.

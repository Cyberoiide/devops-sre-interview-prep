# Solution — Copy-on-Write in the Linux Kernel

## The short definition

**Copy-on-Write (CoW)** is an optimization where multiple parties share the same
physical resource (here, memory pages) as long as they only *read* it. The copy
is deferred until someone *writes*. On that first write the kernel intervenes,
makes a private copy for the writer, and lets the write proceed against the copy.
Everyone who never writes keeps sharing the original — so you never pay to copy
data that is never modified.

The payoff is twofold: **memory saved** (one physical page instead of N copies)
and **speed** (operations like `fork()` return without copying gigabytes).

## How it actually works, at the page level

Memory is managed in **pages** (typically 4 KB). Each process has page tables
mapping its virtual addresses to physical page frames, and each mapping carries
permission bits (read/write). CoW is implemented with those bits plus the page
fault mechanism:

1. **Set up sharing.** When pages are to be shared CoW (e.g. across `fork()`),
   the kernel points *both* processes' page-table entries at the *same* physical
   pages and marks those entries **read-only** — even for pages that were
   originally writable. It also bumps a per-page reference count and flags the VMA
   so the fault handler knows this is a CoW mapping, not a genuine permission
   error.

2. **Reads are free.** As long as both sides only read, nothing happens. They
   share one physical copy. This is the whole point — no copying occurred.

3. **A write traps.** When a process tries to *write* a read-only CoW page, the
   CPU's MMU raises a **page fault** and traps into the kernel's fault handler.

4. **The kernel copies one page.** The handler sees this is a CoW page (writable
   in the VMA, but the PTE was deliberately marked read-only, refcount > 1). It:
   - allocates a fresh physical page,
   - copies the contents of the shared page into it,
   - repoints *this* process's page-table entry to the new page and marks it
     writable,
   - decrements the shared page's reference count.
   The faulting instruction is then **restarted** and the write lands on the
   private copy. (If the refcount had already dropped to 1 — nobody else shares it
   — the kernel just flips it back to writable in place, no copy needed.)

5. **Only that page is copied.** Untouched pages stay shared. The copy is **lazy
   and per-page**, driven entirely by which pages actually get written. Part 2 of
   the lab shows this directly: writing half the buffer copies exactly half the
   pages.

So "Copy-on-Write" is literal: the copy happens *on* the *write*, in the page
fault handler, for that single page.

## The canonical use case: fork()

`fork()` creates a child process that is a near-duplicate of the parent, including
its entire address space. Copying a multi-gigabyte address space eagerly would be
catastrophically slow and wasteful — especially because of the dominant real-world
pattern, **`fork()` + `exec()`**: a shell forks and the child immediately calls
`exec()` to replace its whole image with a new program, throwing away the copied
address space it never touched. Copying it first would be pure waste.

CoW makes `fork()` cheap:
- The kernel does **not** copy the parent's pages. It copies the page *tables*
  (small) and marks all writable pages read-only in both processes.
- Child and parent now **share every page CoW**. `fork()` returns in microseconds
  regardless of how big the parent is.
- Only when one of them *writes* a page does that page get duplicated. For
  `fork()`+`exec()`, the child writes almost nothing before `exec()` blows the
  address space away, so almost no copying ever happens.

The lab's Part 1 shows this: right after `fork()`, the child's `Pss` (its
proportional share of physical memory) is half its `Rss`, and the pages show up
as `Shared_Dirty`. The address space wasn't copied — it was shared. Only after
the child writes every page does its memory become `Private_Dirty`.

(Aside: today even the page tables aren't fully duplicated eagerly on modern
kernels, and `vfork()`/`posix_spawn()` optimize the fork+exec path further by not
even duplicating the address space setup. But CoW is the foundational mechanism.)

## Other important use cases

- **Containers, shared libraries, and memory-mapped files.** When many processes
  `mmap` the same file (a shared library like libc, or a data file), the kernel
  backs them all with the *same* physical pages, read-only, shared. Fifty
  containers running the same image share one physical copy of libc's text pages
  instead of fifty copies. Writable data mapped `MAP_PRIVATE` is CoW: a process
  only gets its own copy of a page when it writes to it. This is a big reason
  container density is viable.

- **Memory overcommit.** Because CoW (and demand paging generally) means
  allocated-but-unwritten pages cost **no physical RAM**, Linux is willing to
  hand out more virtual memory than it physically has — **overcommit**. A
  `fork()` of a 4 GB process "allocates" another 4 GB of address space
  instantly, but backs almost none of it. The policy lives in
  `/proc/sys/vm/overcommit_memory`:
  - `0` — heuristic (default): allow reasonable overcommit, reject the wildly
    unreasonable.
  - `1` — always overcommit: never refuse (useful for sparse allocators,
    dangerous otherwise).
  - `2` — strict: cap total commit at `swap + RAM * overcommit_ratio/100`;
    allocations beyond that fail up front at `malloc`/`fork`.
  The catch: overcommit is a *promise*. If enough processes eventually write their
  promised pages and physical RAM runs out, the kernel can't back out gracefully —
  it invokes the **OOM killer**, which picks a process (scored, tunable per
  process via `/proc/<pid>/oom_score_adj`) and kills it to reclaim RAM. CoW's
  cheapness is exactly why overcommit and therefore the OOM killer exist.

## Related concept: CoW at the filesystem / snapshot layer

The same "share until written, then copy the changed block" idea is used one
layer up, on disk rather than in RAM:

- **btrfs and ZFS** are copy-on-write filesystems: writing a block doesn't
  overwrite in place, it writes a new block and repoints metadata. This makes
  **snapshots** nearly free — a snapshot just shares all existing blocks and only
  diverges as new writes create new blocks.
- **overlayfs and Docker/OCI image layers** use CoW semantics: containers share
  read-only image layers, and a write to a file copies it up into a writable
  layer (`copy_up`) rather than modifying the shared layer. Many containers share
  one base image on disk; each only stores what it changes.

These are conceptually the same trade — share read-only, copy on first write — at
block/file granularity instead of memory-page granularity.

## The trap this creates for observability

Because CoW-shared pages are counted against *every* sharer, **`ps`/`top` RSS
double-counts them**: the parent and child in the lab each report ~256 MB RSS,
implying 512 MB used, when the box actually spent ~256 MB. The honest number is
**Pss** (Proportional Set Size) in `/proc/<pid>/smaps` / `smaps_rollup`, which
divides each shared page's cost by the number of processes sharing it. When
capacity-planning forked/shared-memory workloads, sum Pss, not RSS.

## The answer you give out loud

"Copy-on-Write means several parties share the same physical pages read-only and
the kernel defers copying until someone writes. Mechanically: the shared pages'
page-table entries are marked read-only; a write raises a page fault; the fault
handler sees it's a CoW page, allocates a new page, copies just that one page's
contents, repoints the writer's page table to the private copy and marks it
writable, then restarts the instruction. Untouched pages stay shared, so you
never copy what's never modified. The classic use is `fork()`: the child shares
the parent's whole address space CoW instead of copying it, which is why fork is
fast and why fork+exec is nearly free — the child replaces its image before
writing much. The same principle lets containers share read-only libc pages and
lets Linux overcommit memory, since unwritten pages cost nothing — which is in
turn why the OOM killer exists. And the same idea shows up on disk in btrfs/ZFS
snapshots and overlayfs/Docker layers. One gotcha: `ps` RSS double-counts shared
pages, so use Pss from `/proc/<pid>/smaps` for true usage."

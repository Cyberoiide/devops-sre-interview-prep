# Q196 — Copy-on-Write (CoW) in the Linux Kernel

## Scenario (as posed in the interview)

> "Explain Copy-on-Write in the Linux kernel. What is it, how does it actually
> work at the page level, and give me a good use case where it matters."

## What the interviewer is really testing

Three things:

1. **Do you understand the mechanism, not just the phrase?** "It delays copying"
   is the slogan. The real answer is a page-level story: shared physical pages
   marked read-only, a write fault (trap into the kernel), the kernel duplicating
   *that one page*, and the writer getting its own private copy. If you can walk
   the page fault, you understand it.

2. **Can you connect it to something you actually use?** The canonical case is
   `fork()`: the child doesn't get a fresh 2 GB copy of the parent's address
   space — it shares every page CoW and only pays for pages that get written.
   This is why `fork()` is cheap and why `fork()`+`exec()` (the shell spawning a
   command, a server spawning a worker) is nearly free even from a huge parent.

3. **Do you see the systemic consequences?** CoW is *why* Linux can overcommit
   memory (unwritten pages cost nothing), which is *why* there's an OOM killer,
   which is *why* a fork bomb or a badly sized JVM can take a box down. The same
   idea shows up one layer up in filesystem snapshots (btrfs/ZFS/overlayfs) and
   container image layers.

## The one sentence you must get right

> **Copy-on-Write lets multiple parties share the same physical memory pages
> read-only; the kernel defers the actual copy until someone writes, at which
> point a page fault traps into the kernel, it copies just that page, and gives
> the writer a private copy. You never pay to copy pages that are never
> modified.**

## Why it matters operationally

CoW is the reason `fork()` on a 4 GB process returns in microseconds instead of
copying 4 GB, and the reason a container host can pack dozens of containers that
share the same read-only libc pages. It's also the reason `free` and per-process
RSS can be *misleading* — the same physical pages are counted against multiple
processes until a write splits them (this is exactly what PSS vs RSS in
`/proc/<pid>/smaps` exists to disentangle). Understand CoW and a whole class of
"where did my memory go?" and "why did the OOM killer fire?" questions stop being
mysteries.

Work through `exercise.md` to fork a process holding a big buffer and *watch*
the pages stay shared until the child writes, then read `solution.md` for the
full page-fault walkthrough and `deeper.md` for the follow-ups.

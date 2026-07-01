# Exercise — Watch Copy-on-Write Happen Across a fork()

You will run a program that allocates and touches a large buffer, then `fork()`s.
Immediately after the fork the child's pages are **shared** with the parent
(read-only, CoW) — so the child's *proportional* memory is tiny even though it
"has" the whole buffer. Then the child **writes** to the buffer, the kernel traps
each write into a page fault, copies the page, and the child's pages become
**private**. You will see this flip in `/proc/<pid>/smaps_rollup`: `Shared_Dirty`
and low `Pss` before the write, `Private_Dirty` and full `Pss` after.

Estimated time: 15–25 minutes. Everything is local; you need `gcc`, `python3`,
and a real Linux kernel (the `/proc/.../smaps` and overcommit knobs are
Linux-only — WSL2 or a Linux VM is fine).

```bash
mkdir -p ~/cow-lab && cd ~/cow-lab
```

---

## Part 1 — A fork() that shares pages, then writes

The key measurement is **Pss** (Proportional Set Size). If a 256 MB region is
shared by 2 processes, each is charged 128 MB of Pss even though each shows
256 MB of Rss. When a write splits a page, it moves from `Shared_Dirty` to
`Private_Dirty` and Pss climbs back to the full amount. That flip *is* CoW.

```bash
cat > cow_fork.c <<'EOF'
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/wait.h>

#define MB (1024UL * 1024UL)
static size_t BUF = 256UL * MB;

/* Print this process's memory from the kernel's own accounting. */
static void report(const char *who) {
    FILE *f = fopen("/proc/self/smaps_rollup", "r");
    if (!f) { perror("smaps_rollup"); return; }
    char line[256];
    long rss = 0, pss = 0, shared_dirty = 0, private_dirty = 0, v;
    while (fgets(line, sizeof line, f)) {
        if      (sscanf(line, "Rss: %ld kB", &v) == 1)            rss = v;
        else if (sscanf(line, "Pss: %ld kB", &v) == 1)            pss = v;
        else if (sscanf(line, "Shared_Dirty: %ld kB", &v) == 1)   shared_dirty = v;
        else if (sscanf(line, "Private_Dirty: %ld kB", &v) == 1)  private_dirty = v;
    }
    fclose(f);
    printf("[%-18s pid=%d] Rss=%ldMB Pss=%ldMB Shared_Dirty=%ldMB Private_Dirty=%ldMB\n",
           who, getpid(), rss/1024, pss/1024, shared_dirty/1024, private_dirty/1024);
    fflush(stdout);
}

int main(void) {
    char *buf = malloc(BUF);
    if (!buf) { perror("malloc"); return 1; }
    memset(buf, 'A', BUF);              /* touch every page: now resident, private to parent */
    report("parent-before");

    pid_t pid = fork();                 /* address space is NOT copied - shared CoW */
    if (pid < 0) { perror("fork"); return 1; }

    if (pid == 0) {                     /* child */
        report("child-after-fork");     /* pages SHARED with parent, read-only */
        memset(buf, 'B', BUF);          /* WRITE -> page fault per page -> kernel copies each */
        report("child-after-write");    /* pages now PRIVATE to the child */
        _exit(0);
    }

    wait(NULL);
    report("parent-after");             /* parent unchanged: still its own private pages */
    free(buf);
    return 0;
}
EOF

gcc -O0 -o cow_fork cow_fork.c
./cow_fork
```

Expected output (numbers will be close to this):

```
[parent-before     pid=...] Rss=257MB Pss=256MB Shared_Dirty=0MB   Private_Dirty=256MB
[child-after-fork  pid=...] Rss=256MB Pss=128MB Shared_Dirty=256MB Private_Dirty=0MB
[child-after-write pid=...] Rss=257MB Pss=256MB Shared_Dirty=0MB   Private_Dirty=256MB
[parent-after      pid=...] Rss=257MB Pss=256MB Shared_Dirty=0MB   Private_Dirty=256MB
```

**Read those four lines carefully — this is the whole lesson:**

- `parent-before`: 256 MB, all `Private_Dirty`. The parent owns its pages.
- `child-after-fork`: the child's `Rss` is still 256 MB (it *can* see all of it),
  but `Pss` is only **128 MB** and it's all `Shared_Dirty` — the physical pages
  are shared with the parent. **The fork did not copy 256 MB.** No copy happened.
- `child-after-write`: after writing every page, `Shared_Dirty` drops to 0 and
  `Private_Dirty` jumps to 256 MB. `Pss` climbs to the full 256 MB. **Each write
  triggered a page fault and the kernel copied that page** into a private one.
- `parent-after`: the parent is untouched — its data is still `'A'`, still
  private. CoW isolated the two.

---

## Part 2 — Prove the copy is lazy and per-page

Change the child so it writes only the **first half** of the buffer, and watch
only half the pages go private:

```bash
sed 's/memset(buf, .B., BUF);/memset(buf, 0x42, BUF\/2);/' cow_fork.c > cow_half.c
gcc -O0 -o cow_half cow_half.c
./cow_half
```

Now `child-after-write` shows roughly `Shared_Dirty=128MB Private_Dirty=128MB`:
the untouched half is *still shared* with the parent, the written half was
copied. CoW copies **only the pages you actually write**, one page fault at a
time — never the whole region.

---

## Part 3 — Observe from the outside with ps / smaps

You don't always control the program. Watch a parent's shared memory from another
shell. Start a process that forks a child which just sleeps (child shares the
parent's pages and never writes):

```bash
cat > cow_sleep.c <<'EOF'
#define _GNU_SOURCE
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <stdio.h>
int main(void){
    size_t n = 256UL*1024*1024;
    char *b = malloc(n); memset(b, 'A', n);
    printf("parent pid=%d\n", getpid()); fflush(stdout);
    if (fork()==0){ printf("child  pid=%d (sharing, never writes)\n", getpid()); fflush(stdout); }
    pause();                 /* both sit forever until you kill them */
    return 0;
}
EOF
gcc -O0 -o cow_sleep cow_sleep.c
./cow_sleep &
sleep 1
```

In the same shell:

```bash
# Both processes each show ~256MB RSS (RSS double-counts shared pages!):
ps -o pid,rss,cmd -C cow_sleep

# But Pss tells the truth - each is charged ~half, because pages are shared:
for p in $(pgrep cow_sleep); do
  echo -n "pid $p Pss: "; grep '^Pss:' /proc/$p/smaps_rollup
done
```

**Key observation:** `ps`/`top` RSS **double-counts** CoW-shared pages — add up
the RSS of the parent and child and you get ~512 MB, but the box only spent
~256 MB of physical RAM on that buffer. `Pss` in `smaps_rollup` divides shared
pages by the number of sharers, so the two `Pss` values sum to the real ~256 MB.
This is the single most common way CoW confuses capacity planning.

Clean up:

```bash
pkill cow_sleep
```

---

## Part 4 — CoW is why Linux can overcommit memory

Because unwritten shared pages cost nothing, the kernel lets you *allocate* far
more virtual memory than you have RAM, betting you won't touch it all. Check the
policy:

```bash
cat /proc/sys/vm/overcommit_memory    # 0 = heuristic (default), 1 = always, 2 = strict
cat /proc/sys/vm/overcommit_ratio     # only used in mode 2
```

- **0 (default):** the kernel uses a heuristic and generally lets you allocate
  more than RAM. A `fork()` of a 4 GB process "allocates" another 4 GB of virtual
  address space instantly — but thanks to CoW almost none of it is real until
  written.
- **2 (strict):** allocation is capped at `swap + RAM*overcommit_ratio%`;
  over-allocation fails at `malloc`/`fork` time instead of later.

Demonstrate the overcommit that CoW enables — allocate 8 GB you never touch on a
box that doesn't have 8 GB free, and watch it succeed because no pages are backed
until written:

```bash
python3 - <<'EOF'
import ctypes, os
libc = ctypes.CDLL("libc.so.6", use_errno=True)
libc.malloc.restype = ctypes.c_void_p
GB = 1024**3
p = libc.malloc(ctypes.c_size_t(8*GB))   # virtual allocation only
print("malloc(8GB) ->", "OK" if p else "FAILED")   # OK under default overcommit
# We never write to it, so RSS barely moves:
with open(f"/proc/{os.getpid()}/status") as f:
    print(next(l.strip() for l in f if l.startswith("VmRSS")))
    f.seek(0)
    print(next(l.strip() for l in f if l.startswith("VmSize")))
EOF
```

`VmSize` (virtual) jumps by 8 GB; `VmRSS` (resident, real RAM) stays tiny. The
gap is memory the kernel *promised* but hasn't had to *deliver* — the same
deferred-cost principle as CoW. When too many processes eventually cash in those
promises and RAM runs out, the **OOM killer** picks a victim (tunable per process
via `/proc/<pid>/oom_score_adj`). A fork bomb is this taken to the extreme: cheap
CoW forks let you spawn processes faster than the box can account for them.

---

## What you should be able to explain after this lab

- After `fork()`, parent and child **share** every page read-only; the child's
  `Pss` is a fraction of its `Rss` (Part 1, `child-after-fork`).
- The first **write** to a shared page triggers a **page fault**, the kernel
  copies **just that page**, and it becomes `Private_Dirty` — copies are lazy and
  per-page, not whole-region (Parts 1 and 2).
- `ps`/`top` **RSS double-counts** shared pages; `Pss` in `/proc/<pid>/smaps`
  reflects true physical usage (Part 3).
- CoW's "cost nothing until touched" is the same principle that lets Linux
  **overcommit** memory, which is why the **OOM killer** exists and why a fork
  bomb is dangerous (Part 4).

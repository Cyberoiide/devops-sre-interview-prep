# Solution — Diagnosing Hundreds of CLOSE_WAIT Connections

## First, what `netstat`/`ss` is showing you

`netstat -tan` and `ss -tan` list every TCP connection the kernel knows about,
along with the **TCP state** each one is in. Every TCP connection is a state
machine; the state tells you exactly where in its lifecycle a connection sits.
Seeing hundreds of connections in one state — here `CLOSE_WAIT` — is a signal
that connections are getting *stuck* at one specific transition.

(`netstat` is the old tool from `net-tools`; `ss` from `iproute2` is the modern,
faster replacement. `ss -tan` = TCP, all, numeric. Prefer `ss`.)

## The TCP connection-close handshake

Closing a TCP connection is a **four-way** exchange (two FINs, each ACKed). Either
side can initiate. Call the side that closes first the *active closer* and the
other the *passive closer*.

```
   Active closer                                 Passive closer
   (the one who calls close() first)             (the one who closes later)

   ESTABLISHED                                   ESTABLISHED
        |  --- FIN --------------------------->      |
   FIN_WAIT_1                                         |   receives FIN
        |  <-- ACK ----------------------------      |   kernel auto-ACKs
   FIN_WAIT_2                                    CLOSE_WAIT   <-- waits for the
        |                                             |         LOCAL APP to
        |                                             |         call close()
        |                                        (app calls close())
        |  <-- FIN ----------------------------      |
        |  --- ACK -------------------------->   LAST_ACK
   TIME_WAIT                                          |
        |  (waits 2*MSL, then)                        |  receives ACK
   CLOSED                                        CLOSED
```

Walk it through:

1. The **active closer** calls `close()`, sends **FIN**, enters `FIN_WAIT_1`.
2. The **passive closer**'s *kernel* receives the FIN and automatically sends
   **ACK**. The passive side now enters **`CLOSE_WAIT`**. Crucially, the kernel
   has done everything it can — the next step is for the *local application* on
   the passive side to notice the connection is half-closed and call `close()`
   itself.
3. When the passive-side app finally calls `close()`, it sends its own **FIN**
   and moves to `LAST_ACK`.
4. The active closer ACKs that FIN and sits in **`TIME_WAIT`** for `2*MSL`
   (typically ~60s on Linux) to absorb any stray delayed packets, then goes to
   `CLOSED`.

## So what does `CLOSE_WAIT` actually mean?

`CLOSE_WAIT` = **"my peer has closed its half of the connection (I got its FIN and
ACKed it), and I am now waiting for *my own application* to call `close()`."**

The name is literally "waiting to close (locally)." The kernel cannot move the
connection forward on its own — sending the final FIN is the application's job,
triggered by the application calling `close()` on the socket. Until the app does
that, the connection stays in `CLOSE_WAIT` **forever**. There is no kernel timer
that reaps it.

Therefore:

> **Hundreds of `CLOSE_WAIT` sockets = a local application bug.** The application
> is not closing sockets after the remote peer hangs up.

This is the accuracy point interviewers probe. `CLOSE_WAIT` is *not* the network's
fault and *not* the remote's fault — the remote did the right thing by closing.
It is the **local** app failing to reciprocate.

## Common code-level causes

- **Missing `close()` on error paths.** The happy path closes the socket, but an
  exception/early-return skips it. (Use `try/finally`, `with`, RAII,
  `defer conn.Close()`.)
- **Leaked file descriptors.** The socket object goes out of reference tracking
  but was never explicitly closed, or is held in a collection forever (exactly
  the bug in the lab).
- **Connection-pool bugs.** A pooled client keeps the connection but the pool
  never detects the peer closed it, so it never reaps/closes it. Or the pool is
  configured to keep more idle connections than it ever validates.
- **A blocked/slow event loop or worker** that never gets around to reading the
  socket, so it never observes the peer's FIN (`recv()` returning 0) and never
  closes.

## Contrast with TIME_WAIT (don't confuse them)

| | `CLOSE_WAIT` | `TIME_WAIT` |
|---|---|---|
| Which side | The **passive** closer (peer closed first) | The **active** closer (you closed first) |
| Meaning | Waiting for the **local app** to call `close()` | Waiting `2*MSL` to absorb stray packets |
| Clears on its own? | **No** — only when the app closes | **Yes** — expires after ~60s automatically |
| A large pile means | **App bug**: leaking sockets | Usually normal; huge counts can mean lots of short-lived outbound conns (tune with connection reuse, not by disabling it) |

`TIME_WAIT` is a healthy, designed-in state. `CLOSE_WAIT` that grows without bound
is a defect.

## How to find the offending process

1. **Attribute sockets to a PID:**
   ```bash
   ss -tanp state close-wait          # -p shows the owning process (sudo for others)
   ```
2. **Confirm with lsof:**
   ```bash
   lsof -nP -iTCP -sTCP:CLOSE_WAIT     # all CLOSE_WAIT sockets and their owners
   lsof -nP -p <pid> | grep TCP        # everything that one process holds
   ```
3. **Inspect the leaked fds directly:**
   ```bash
   ls -l /proc/<pid>/fd | grep socket | wc -l   # count of socket fds held
   ```
4. **Check the fd ceiling it is racing toward:**
   ```bash
   cat /proc/<pid>/limits | grep 'open files'   # per-process soft/hard limit
   ulimit -n                                     # your shell's soft limit
   ```

## Why this is dangerous: file-descriptor exhaustion

Each `CLOSE_WAIT` socket holds a **file descriptor**. FDs are limited per process
(`ulimit -n`, i.e. `RLIMIT_NOFILE`) and system-wide (`fs.file-max`). A steadily
growing `CLOSE_WAIT` count is a slow leak; when the process hits its fd limit,
every operation needing a new fd fails with **`EMFILE` — "Too many open files"**.
The service can no longer accept connections, open files, or make outbound calls.
So a socket-leak bug presents as a gradual degradation and then a hard outage. The
lab's Part 5 reproduces this on purpose.

## The fix

**Fix the application, not the kernel.** Kernel timers do not reap `CLOSE_WAIT`:
- `tcp_fin_timeout` governs `FIN_WAIT_2` (the *active* closer waiting for the
  peer's FIN), **not** `CLOSE_WAIT`.
- `SO_KEEPALIVE`/keepalive tuning can eventually tear down truly dead peers, but
  it is a band-aid, slow, and does not address the bug: the app should close
  promptly on its own.

Real fixes:
- **Always `close()` the socket on every path**, including errors — `try/finally`,
  context managers (`with`), `defer`, RAII.
- **Read to EOF:** a `recv()` returning 0 bytes is the peer's FIN; that is the
  cue to close.
- **Fix the connection pool:** ensure it validates/reaps connections the peer has
  closed and enforces sane idle limits.
- As a *stopgap* to survive while shipping the code fix: **raise `ulimit -n`** to
  buy time, and/or restart the process to reclaim the leaked fds — but understand
  these only delay the inevitable until the code is corrected.

## The answer you give out loud

"`ss` is showing me the TCP state of every connection. `CLOSE_WAIT` means the
remote peer already closed its half — it sent a FIN, my kernel ACKed it — and the
socket is now waiting for *my* application to call `close()` and send its own FIN.
The kernel won't clear it; only the app can. So hundreds of them is a local
application bug: it's leaking sockets, not closing them after the peer hangs up —
typically a missing `close()` on an error path or a broken connection pool. That
matters because each one is a file descriptor, and as they pile up the process
hits its `ulimit -n` and starts failing with 'too many open files.' I'd find the
guilty PID with `ss -tanp` / `lsof -iTCP -sTCP:CLOSE_WAIT`, confirm the fd growth
in `/proc/<pid>/fd`, raise the fd limit or restart as a stopgap, and fix the code
to always close the socket. This is distinct from `TIME_WAIT`, which is normal, on
the side that closed first, and clears itself after ~60 seconds."

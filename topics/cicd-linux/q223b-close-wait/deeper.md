# Deeper — Interviewer Follow-up Questions

Concise, correct answers to the pushbacks that separate memorizers from people
who understand the TCP state machine.

### Q: Whose fault is CLOSE_WAIT — local or remote?

**Local.** The remote peer did the correct thing: it closed its side and sent a
FIN, which the local kernel dutifully ACKed. The connection is now stuck waiting
for the *local application* to call `close()`. A growing `CLOSE_WAIT` count is a
local socket-leak bug, full stop. Blaming the network or the remote is the classic
wrong answer.

### Q: How is CLOSE_WAIT different from TIME_WAIT?

They are on opposite sides of the close. `TIME_WAIT` is on the **active closer**
(the side that called `close()` first) and is a *normal* state — it waits `2*MSL`
(~60s) to absorb delayed/stray packets and prevent an old segment from being
misattributed to a new connection, then clears itself. `CLOSE_WAIT` is on the
**passive closer**, means "waiting for my own app to close," and does **not**
clear on its own. Lots of `TIME_WAIT` is usually fine; growing `CLOSE_WAIT` is a
bug.

### Q: Will tuning `tcp_fin_timeout` clear the CLOSE_WAIT sockets?

No. `tcp_fin_timeout` bounds how long a connection stays in **`FIN_WAIT_2`** — the
*active* closer waiting for the peer's FIN. `CLOSE_WAIT` is a different state on
the other side with **no kernel timer** at all. The only thing that moves a socket
out of `CLOSE_WAIT` is the local application calling `close()`. This is a favorite
trap question.

### Q: Then does the kernel ever clean these up?

Only indirectly. TCP keepalive (`SO_KEEPALIVE` plus `tcp_keepalive_*` sysctls) can
eventually detect and tear down connections whose peer is truly gone, but it is
slow (default probes start after ~2 hours) and it is a workaround, not a fix — the
application should notice the FIN (a 0-byte `recv()`) and close promptly. If the
process exits, the OS reclaims all its sockets and fds, which is why "restart it"
temporarily clears the pile.

### Q: What actually happens to trigger the transition into CLOSE_WAIT?

The peer calls `close()` and sends a FIN. The local kernel receives it and sends
an ACK automatically, moving the socket to `CLOSE_WAIT`. From the local app's
perspective, the next `recv()`/`read()` on that socket returns **0 bytes (EOF)** —
that is the application-visible signal that the peer closed. A correct app treats
that EOF as "close my side now." A buggy app ignores it (or never reads), so the
socket lingers.

### Q: Why do these connections threaten the whole service?

Each socket is a **file descriptor**, and fds are a bounded resource — per process
(`RLIMIT_NOFILE`, seen via `ulimit -n` and `/proc/<pid>/limits`) and system-wide
(`fs.file-max`). A leak that never releases sockets marches the process toward its
fd ceiling; on reaching it, every new `socket()`, `open()`, or `accept()` fails
with **`EMFILE` ("Too many open files")**. The service stops accepting connections
and effectively goes down, even though CPU and RAM look fine.

### Q: How do you find which process owns them?

```bash
ss -tanp state close-wait          # owning process per socket (sudo for others)
lsof -nP -iTCP -sTCP:CLOSE_WAIT    # all CLOSE_WAIT sockets and their owners
lsof -nP -p <pid> | grep TCP       # everything a suspected process holds
ls /proc/<pid>/fd | wc -l          # count of fds the process is holding (watch it grow)
```

The growing `/proc/<pid>/fd` count over time confirms the leak and pins it to a
specific process.

### Q: You need to keep the service alive right now. What's the stopgap vs. the real fix?

**Stopgap:** raise the process's fd limit (`ulimit -n` / systemd `LimitNOFILE`) to
buy headroom, and/or restart the process to reclaim leaked fds — both only delay
the inevitable. **Real fix (in the code):** always `close()` on every path
including errors (`try/finally`, `with`, `defer`, RAII), treat a 0-byte `recv()`
as the cue to close, and fix the connection pool so it validates and reaps
peer-closed connections. Kernel knobs do not fix it.

### Q: Could a load balancer or proxy in front cause CLOSE_WAIT on my box?

It can be the *trigger* but not the *bug*. If an LB/proxy aggressively closes idle
upstream connections (sends FIN), your server will land those sockets in
`CLOSE_WAIT` — and if your app never closes them, they pile up. The peer closing
is normal and expected; your app failing to close in response is the defect. The
fix is still on your side: close promptly when the peer hangs up.

### Q: What's the difference between `netstat` and `ss` here, and why prefer `ss`?

`netstat` is the legacy tool from the `net-tools` package; `ss` is from
`iproute2`. `ss` reads socket info from the kernel via netlink rather than parsing
`/proc/net/tcp`, so it is significantly faster on boxes with many connections
(exactly the situation you're debugging), and it has convenient state filters like
`ss -tan state close-wait`. Same information, better tool.

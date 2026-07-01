# Q223b — Hundreds of CLOSE_WAIT Connections

## Scenario (as posed in the interview)

> "You SSH into a production box because a service is misbehaving. You run
> `netstat -tan` (or `ss -tan`) and you see hundreds of TCP connections stuck in
> the `CLOSE_WAIT` state — and the number keeps climbing. What does `CLOSE_WAIT`
> mean, what is causing it, and how do you fix it?"

## What the interviewer is really testing

Three things:

1. **Do you actually understand the TCP connection-close handshake** — the FIN/ACK
   exchange and the state machine each side walks through? Many candidates can
   recite state names but cannot say *which side* is in *which* state and *why*.

2. **Can you correctly assign blame?** This is the crux. `CLOSE_WAIT` is
   frequently misdiagnosed. The common wrong answer is "the network is dropping
   connections" or "it's the same as TIME_WAIT." The correct answer is that
   `CLOSE_WAIT` is **the local application's fault**: the remote peer closed its
   side, the kernel did its part, and now the socket is waiting for the *local
   application* to call `close()` — and the application never does.

3. **Do you know how to hunt down the offending process and fix it** — with `ss`,
   `lsof`, `/proc/<pid>/fd`, an understanding of file-descriptor limits, and the
   knowledge that the real fix is a code fix, not a kernel tunable.

## The one sentence you must get right

> **`CLOSE_WAIT` means the *remote* peer has closed its half of the connection,
> the *local kernel* has ACKed that, and the socket is now waiting for the *local
> application* to call `close()`. A pile of `CLOSE_WAIT` sockets is a local
> application bug: it is leaking sockets by never closing them.**

Contrast with `TIME_WAIT`, which is normal and appears on the side that closed
*first*, protecting against stray delayed packets. `TIME_WAIT` is healthy and
self-clears; `CLOSE_WAIT` that keeps growing is a leak.

## Why it matters operationally

Every socket stuck in `CLOSE_WAIT` holds a file descriptor. File descriptors are
capped per process (`ulimit -n`) and system-wide. As `CLOSE_WAIT` sockets
accumulate, the process eventually hits its fd limit and starts failing with
`EMFILE` ("Too many open files") — it can no longer accept connections, open
files, or make outbound calls. So a socket-leak bug that looks like a networking
curiosity turns into a hard outage.

Work through `exercise.md` to make `CLOSE_WAIT` pile up on purpose and watch it,
then read `solution.md` for the full TCP-state walkthrough and diagnosis, and
`deeper.md` for the follow-up questions.

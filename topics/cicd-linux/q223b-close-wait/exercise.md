# Exercise — Make CLOSE_WAIT Pile Up, Then Fix It

You will run a deliberately buggy TCP server that accepts connections but never
calls `close()` on them. When clients hang up (send FIN), each socket gets stuck
in `CLOSE_WAIT`. You will watch the pile grow with `ss`, find the offending
process with `lsof` and `/proc/<pid>/fd`, see it hit the file-descriptor limit,
then fix the server and confirm the leak is gone.

Estimated time: 20–30 minutes. Everything is local; only Python 3 and standard
Linux net tools (`ss`, `lsof`) are needed.

```bash
mkdir -p ~/close-wait-lab && cd ~/close-wait-lab
```

---

## Part 1 — The buggy server (leaks sockets)

The bug: it accepts a connection, keeps the socket in a list, and **never closes
it** even after the client disconnects.

```bash
cat > leaky_server.py <<'EOF'
import socket

HOST, PORT = "127.0.0.1", 9099
srv = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
srv.bind((HOST, PORT))
srv.listen(128)
print(f"[leaky] listening on {HOST}:{PORT}, pid={__import__('os').getpid()}")

held = []                      # BUG: we keep every accepted socket forever
while True:
    conn, addr = srv.accept()
    print(f"[leaky] accepted {addr}")
    # We never conn.recv() to notice the FIN, and never conn.close().
    held.append(conn)          # leak: reference kept, socket never closed
EOF
```

Start it (leave this terminal running):

```bash
python3 leaky_server.py
```

Note the printed `pid=` — you will need it. In another terminal:

```bash
SRV_PID=$(pgrep -f leaky_server.py)
echo "server pid=$SRV_PID"
```

---

## Part 2 — A client that connects and immediately hangs up

Each client opens a connection and then closes it right away (sends FIN). Because
the client closed first, the *client* side goes to TIME_WAIT and the *server*
side goes to CLOSE_WAIT — which is what we want to observe.

```bash
cat > hangup_client.py <<'EOF'
import socket, sys, time

n = int(sys.argv[1]) if len(sys.argv) > 1 else 50
for i in range(n):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect(("127.0.0.1", 9099))
    s.close()                  # client closes -> sends FIN to server
    time.sleep(0.01)
print(f"[client] opened and closed {n} connections")
EOF

python3 hangup_client.py 200
```

---

## Part 3 — Observe the CLOSE_WAIT pile-up

On the server side, every one of those closed clients left a socket the server
never closed. Count them:

```bash
# All CLOSE_WAIT sockets, numeric:
ss -tan state close-wait

# Just the count:
ss -tan state close-wait | grep -c 9099

# Watch it climb as you fire more clients:
watch -n1 "ss -tan state close-wait | grep -c 9099"
```

Fire more clients (in the client terminal) and watch the count only ever go *up*
— the server never releases them:

```bash
python3 hangup_client.py 300
```

**Key observation:** these sockets never leave `CLOSE_WAIT` on their own. The
kernel has finished its part (it ACKed the client's FIN). It is now blocked
forever waiting for the *server application* to call `close()`. That is the whole
point: **CLOSE_WAIT is the local app's responsibility, and this app is buggy.**

For contrast, look at the client side — those show `TIME_WAIT` (normal) and drain
away on their own after ~60s:

```bash
ss -tan state time-wait | grep -c 9099
```

---

## Part 4 — Find the offending process and its leaked fds

In the real world nobody tells you which process is guilty. Attribute the sockets
to a PID.

```bash
# ss -p prints the owning process for each socket (may need sudo for others' procs):
ss -tanp state close-wait

# lsof: list the leaky sockets held by the server pid:
lsof -nP -p "$SRV_PID" | grep -i tcp | head

# Count file descriptors the process currently holds — this number keeps growing:
ls /proc/$SRV_PID/fd | wc -l
```

Each `CLOSE_WAIT` socket is a live file descriptor. Confirm the process's fd
ceiling:

```bash
cat /proc/$SRV_PID/limits | grep -i 'open files'   # the "Max open files" soft/hard limits
# or, for your current shell:
ulimit -n
```

---

## Part 5 — Drive it into the wall (fd exhaustion)

Optional but instructive. Lower the limit and watch the leak turn into an outage.
Restart the server under a tiny fd limit:

```bash
# stop the running server (Ctrl-C in its terminal), then:
( ulimit -n 64; python3 leaky_server.py ) &
SRV_PID=$(pgrep -f leaky_server.py)

python3 hangup_client.py 200
```

Once the server has leaked ~60 sockets it can no longer `accept()` — you'll see it
error with `OSError: [Errno 24] Too many open files` (EMFILE). That is exactly how
a socket leak becomes a production incident: the process stops being able to do
anything that needs a new fd.

---

## Part 6 — The fix (close the socket)

The kernel tunables (`tcp_fin_timeout`, etc.) do **not** fix CLOSE_WAIT — that
timer governs FIN_WAIT_2, not CLOSE_WAIT. The only real fix is to make the
application call `close()`. Here is the corrected server that reads until the peer
closes, then closes its own side:

```bash
cat > fixed_server.py <<'EOF'
import socket, os

HOST, PORT = "127.0.0.1", 9099
srv = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
srv.bind((HOST, PORT))
srv.listen(128)
print(f"[fixed] listening on {HOST}:{PORT}, pid={os.getpid()}")

while True:
    conn, addr = srv.accept()
    try:
        data = conn.recv(4096)     # recv() returning b'' means the peer sent FIN
        # (handle request here if there were one)
    finally:
        conn.close()               # FIX: always close, even on error paths
EOF
```

Restart with the fixed version and re-run the clients:

```bash
# Ctrl-C the leaky server, then:
python3 fixed_server.py &
SRV_PID=$(pgrep -f fixed_server.py)

python3 hangup_client.py 500

# CLOSE_WAIT count stays at (or near) zero — sockets are released promptly:
ss -tan state close-wait | grep -c 9099
ls /proc/$SRV_PID/fd | wc -l    # fd count stays flat, no leak
```

Clean up:

```bash
kill "$SRV_PID" 2>/dev/null
```

---

## What you should be able to explain after this lab

- `CLOSE_WAIT` sits on the side whose **peer closed first**, and it will **never
  clear until the local application calls `close()`** — so a growing pile is a
  local socket-leak bug.
- How to attribute the sockets to a PID (`ss -tanp`, `lsof -p`) and see the leaked
  fds (`/proc/<pid>/fd`).
- Why the leak eventually causes `EMFILE` / "Too many open files" and takes the
  service down, and how `ulimit -n` bounds it.
- That the fix is in the code (`close()` on all paths, fix the connection pool) —
  **not** a kernel timeout knob.

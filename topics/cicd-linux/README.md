# CI/CD & Linux

Interview-prep study material for DevOps/SRE roles, covering CI/CD pipeline
troubleshooting and Linux/networking fundamentals. Every exercise runs on a
laptop with only common tools — `bash`, `docker`, `python3`, and standard Linux
net utilities (`ss`, `lsof`) — no cloud account required.

## Questions

| ID | Topic | One-line summary |
|----|-------|------------------|
| [q222](./q222-large-pr-pipeline-failures/) | CI/CD reliability | A pipeline that passes small PRs fails inconsistently on large ones — OOM, timeouts, flaky tests. How do you diagnose and fix it? |
| [q223b](./q223b-close-wait/) | Linux / TCP | Hundreds of `CLOSE_WAIT` connections on a box — what causes them, and whose fault is it? |

## File layout

Each question folder contains exactly four files:

- **README.md** — the scenario, phrased the way an interviewer would pose it.
- **exercise.md** — a hands-on lab with concrete, runnable commands.
- **solution.md** — the detailed written answer and walkthrough.
- **deeper.md** — likely interviewer follow-up questions with concise answers.

## How to use this repo

1. Read the question's `README.md` and try to answer it out loud (whiteboard style).
2. Do the lab in `exercise.md` — actually run the commands, don't just read them.
3. Compare your reasoning against `solution.md`.
4. Drill the follow-ups in `deeper.md` until the answers are reflexive.

## Prerequisites

- Linux (or WSL2 / a Linux VM) — the OOM, cgroup, and `CLOSE_WAIT` labs need a
  real Linux kernel.
- `docker` runnable by your user (for the CI resource-limit labs).
- `python3`, `bash`, and `iproute2` (`ss`) + `lsof` installed.
- No cloud account or credentials needed.

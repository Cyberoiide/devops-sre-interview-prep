# Terraform / Infrastructure as Code

Interview-prep study material for DevOps/SRE roles, focused on Terraform (and
its fork OpenTofu). Every exercise runs **without any cloud credentials** — it
uses only the `local`, `null`, `random`, `time`, and `local_file` providers, so
you can practice the mechanics of state, drift, and recovery on a laptop.

> All commands below work identically with `terraform` or `tofu`. Where you see
> `terraform`, substitute `tofu` if you use OpenTofu.

## Questions

| ID | Topic | One-line summary |
|----|-------|------------------|
| [q219](./q219-state-recovery/) | State recovery | Resources deleted **and** remote state corrupted at the same time — how do you recover? |
| [q211](./q211-db-migration-vm-to-k8s/) | DB migration VM → Kubernetes | Move a database from a VM to a K8s StatefulSet with minimal downtime. |
| [q223b](./q223b-phantom-resource-rebuild/) | Phantom resource rebuild | `terraform apply` wants to recreate a resource nobody touched — why, and how to stop it? |

## Folder layout

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

- `terraform` >= 1.5 **or** `tofu` >= 1.6
- A shell (`bash`/`zsh`), `jq` is handy but optional.
- No cloud account, no credentials, no network dependency beyond the initial
  `terraform init` (which downloads providers from the registry once).

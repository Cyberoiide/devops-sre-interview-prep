# q197 — Hands-on lab: upgrade Terraform locally, the safe way

This lab runs entirely locally. No cloud credentials, no backend beyond the
one-time `terraform init` (which downloads the `local` provider). We upgrade the
Terraform *binary* with a version manager, pin it per-project, and prove a real
config still initializes and plans against the new version.

Works with `terraform` or `tofu` — substitute the binary name as needed. The
version-manager section uses `tfenv`; the `tenv` equivalents are noted inline.

> **Version numbers change.** Every concrete version below (`1.9.8`, `1.13.x`,
> etc.) is an *example*. Run `tfenv list-remote` (or check
> releases.hashicorp.com) and substitute whatever the current releases are when
> you do this.

---

## 0. Record where you are (do this first, always)

```bash
terraform version
```

This prints the core version, the platform, **and** each provider's version, and
tells you if a newer release is available, e.g.:

```
Terraform v1.5.7
on linux_amd64
+ provider registry.terraform.io/hashicorp/local v2.4.0

Your version of Terraform is out of date! The latest version
is 1.13.x. You can update by downloading from https://www.terraform.io/downloads.html
```

Write that number down — it's your rollback target if anything goes sideways.

---

## 1. Read the release notes before you touch anything

Not a command, but the step that separates a clean upgrade from a surprise.
Before installing, skim:

- The **CHANGELOG** between your version and the target
  (`https://github.com/hashicorp/terraform/blob/main/CHANGELOG.md`).
- The **upgrade guide** for the target minor (HashiCorp publishes one per
  minor, e.g. "Upgrading to Terraform 1.13").

Look for removed/deprecated features. Real examples: the standalone
`terraform taint`/`untaint` commands were deprecated in favor of `-replace`;
various HCL syntaxes have been deprecated across minors. For a big jump (e.g.
1.5 → 1.13), the cautious path is **one minor at a time** so each upgrade guide
applies cleanly.

---

## 2. Set up a real project and check its constraints

```bash
mkdir -p /tmp/q197-upgrade-demo && cd /tmp/q197-upgrade-demo
cat > main.tf <<'EOF'
terraform {
  # The new binary MUST satisfy this, or `terraform` refuses to run.
  required_version = ">= 1.5.0, < 2.0.0"

  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.4"
    }
  }
}

resource "local_file" "hello" {
  filename = "${path.module}/hello.txt"
  content  = "config still works on the new version\n"
}
EOF

terraform init
terraform apply -auto-approve
```

Now inspect what you depend on — core constraint and providers:

```bash
grep required_version main.tf     # what core versions are allowed
terraform providers               # the provider dependency tree + constraints
terraform version                 # core + resolved provider versions
```

If the target binary falls *outside* `required_version`, you either bump the
constraint in `main.tf` or pick a target that satisfies it. Same logic for
provider constraints in step 6.

---

## 3. Back up state BEFORE running any new binary (the key gotcha)

Newer Terraform can **upgrade the state format** the first time it writes state,
and older binaries then **cannot read it** — this is one-way. With a remote
backend, object versioning covers you. Locally, copy the file yourself:

```bash
cp terraform.tfstate terraform.tfstate.pre-upgrade-backup
ls -la terraform.tfstate*
```

If the upgrade goes wrong and you need to drop back to the old binary, this copy
is what lets the old version read state again.

---

## 4. Install the new binary — Path A: version manager (recommended)

A version manager lets you hold many versions side by side and switch per
project. `tfenv` is the classic Terraform-only tool; `tenv` is the modern
successor (also manages OpenTofu / Terragrunt / Atmos).

### Install tfenv

```bash
# Option 1: git clone (Linux/macOS, no package manager needed)
git clone --depth=1 https://github.com/tfutils/tfenv.git ~/.tfenv
export PATH="$HOME/.tfenv/bin:$PATH"        # add this line to ~/.bashrc or ~/.zshrc

# Option 2: macOS Homebrew
# brew install tfenv
```

Confirm it shims `terraform`:

```bash
which terraform          # should now resolve under ~/.tfenv/ (or the brew shim)
tfenv --version
```

### Install and select versions

```bash
tfenv list-remote | head           # all installable versions (newest first)
tfenv install 1.9.8                # install a specific version (substitute current)
tfenv install latest               # install the newest stable release
tfenv use latest                   # make it the active version globally
tfenv list                         # local versions; '*' marks the active one
terraform version                  # confirms the active version
```

> **`tenv` equivalents** if you chose the modern tool instead:
> `tenv tf list-remote`, `tenv tf install 1.9.8`, `tenv tf install latest`,
> `tenv tf use latest`, `tenv tf list`. `tenv` reads the same
> `.terraform-version` files as `tfenv`.

### Pin this project with `.terraform-version`

Drop a file in the project root and the version manager auto-selects it whenever
you `cd` in — this is how you keep a whole team on the same core:

```bash
cd /tmp/q197-upgrade-demo
echo "1.9.8" > .terraform-version    # substitute your chosen version
terraform version                    # now follows the file, not the global default
```

Commit `.terraform-version` to the repo so everyone resolves the same binary.

---

## 5. Install the new binary — Path B: fully manual (no version manager)

Use this in an exam/restricted environment where you can't install `tfenv`.
This is the download-unzip-replace dance on Linux (Windows is the same idea:
unzip and put `terraform.exe` on your `PATH`).

```bash
# Substitute the current version and your platform.
VER=1.9.8
cd /tmp
curl -fsSLO "https://releases.hashicorp.com/terraform/${VER}/terraform_${VER}_linux_amd64.zip"

# (Optional but recommended) verify the checksum against the published SHA256SUMS.
curl -fsSLO "https://releases.hashicorp.com/terraform/${VER}/terraform_${VER}_SHA256SUMS"
sha256sum -c --ignore-missing terraform_${VER}_SHA256SUMS

unzip -o "terraform_${VER}_linux_amd64.zip"     # produces ./terraform
chmod +x terraform
sudo mv terraform /usr/local/bin/terraform      # replace the binary on your PATH
```

> On macOS the one-liner is `brew upgrade hashicorp/tap/terraform`
> (or `brew upgrade terraform`).

---

## 6. Verify, then bump providers within constraints

```bash
terraform version                  # confirms the new CORE version
```

Core and providers version independently. Now pull providers up to the newest
allowed by your constraints and rewrite the lock file:

```bash
cd /tmp/q197-upgrade-demo
terraform init -upgrade            # upgrades providers within constraints...
                                   # ...and updates .terraform.lock.hcl
cat .terraform.lock.hcl            # note the version + hashes that got pinned
```

Commit `.terraform.lock.hcl` — it's what makes provider versions reproducible
for the rest of the team and CI.

---

## 7. Plan to prove nothing broke

```bash
terraform plan
```

You want to see **`No changes. Your infrastructure matches the configuration.`**
That confirms the new core parses your config, reads (and, on first write, may
have upgraded) the state, and produces no unexpected diff. If you *do* see a
surprise diff after only a version bump, that's drift or a provider behavior
change — investigate before you `apply`.

---

## 8. (Optional) OpenTofu is analogous

If you're on OpenTofu instead of Terraform, the whole runbook is the same shape:

```bash
tofu version                       # record current
tenv tofu install latest           # tenv also manages OpenTofu
tenv tofu use latest
tofu init -upgrade && tofu plan
```

Or install OpenTofu from your package manager / the `get.opentofu.org` script.
The state-format and lock-file cautions apply identically.

---

## 9. Clean up

```bash
rm -rf /tmp/q197-upgrade-demo /tmp/terraform_*.zip /tmp/terraform_*_SHA256SUMS

# Only if you want to remove the version manager itself:
# rm -rf ~/.tfenv          # and remove the PATH line from your shell rc
```

## What you should take away

- **Order matters:** `version` → read notes → check `required_version` →
  **back up state** → install → `version` → `init -upgrade` → `plan`.
- The **state format is a one-way upgrade** — back up (or version) state before
  the first run of a newer binary, or an old binary can't read it anymore.
- **Core and providers are separate upgrades**: the binary is the download;
  `terraform init -upgrade` handles providers and the lock file.
- A **version manager** (`tfenv`/`tenv`) plus a committed `.terraform-version`
  and `.terraform.lock.hcl` keeps your whole team — and CI — on identical
  versions with zero manual binary juggling.

# 06 · Lab Setup

> **Level:** ★ foundation
>
> **You will learn:** how to install Terraform the production way (from HashiCorp's signed package repository) in your **WSL2 Ubuntu**, how to pin or switch versions (`apt` pinning and the **tenv** version manager), shell autocomplete, the VS Code extension, a shared **provider cache** so labs don't re-download providers, and how to check that Docker (needed from topic 26) works. Optional: OpenTofu side by side.
>
> **You need:** topic 05 (PATH, environment variables, `.gitignore`). Your WSL2 Ubuntu with Docker Engine from [Docker topic 04](../../Docker/notes/04-installing-docker.md).
>
> **Part 0 · Foundations** · Previous: [05 · Linux, Git and cloud API recap](05-linux-git-and-api-recap.md) · Next: [07 · Your first Terraform project](07-first-terraform-project.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Terraform is one binary. Install it from **HashiCorp's apt repository** so updates come through `apt` and packages are signature-checked. Then check it with `terraform version`.

**Picture it.** Like installing Docker Engine from Docker's own repo instead of a random script: the official source, verified, updatable.

**The one thing to know.** In WSL2 Ubuntu:

```bash
sudo apt-get update && sudo apt-get install -y gnupg curl lsb-release
curl -fsSL https://apt.releases.hashicorp.com/gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update && sudo apt-get install -y terraform
terraform version
```

**Try it now:** run it, then `terraform -help`.

**You've got it if you can:** see `Terraform v1.17.x` and explain why you installed from a repo instead of downloading a random zip.

> 📖 **That's the core.** Everything below is the **full reference**.

---

## What it is

Your lab for the whole course:

| Piece | Why | From |
|---|---|---|
| **WSL2 Ubuntu** | A real Linux host on your Windows laptop | Docker topic 04 |
| **Terraform 1.17.x** | The tool | This topic |
| **Docker Engine** in WSL2 | The `docker` provider (topics 26–27) and kind | Docker topic 04 |
| **kind** (Kubernetes IN Docker) | The `kubernetes` and `helm` providers (topic 28) | [Kubernetes topic 11](../../CKA-Certification-Course-2025/notes/11-lab-setup-kind-and-minikube.md) |
| **AWS CLI** + free-tier account | Part 4 onward | Topic 29 |
| **VS Code** + WSL + HashiCorp Terraform extensions | Editing with syntax highlighting, completion, formatting | This topic |
| **jq**, **git** | Reading JSON, versioning your code | Topic 05 |

Three ways to install Terraform:

| Way | Use it? |
|---|---|
| **HashiCorp apt/dnf repository** | ✅ **Yes**: signed packages, normal `apt upgrade`, pinnable. |
| **A version manager** (tenv, tfenv) | ✅ Yes, if you work on several projects that need different versions. |
| Manual zip from `releases.hashicorp.com` | Only where you can't use a package manager (e.g. a locked-down CI image). **Verify the checksum.** |
| Snap / distro packages / random scripts | ❌ Often outdated or unofficial. |

---

## Why it exists (why do it this way)

- **Signed repo** = `apt` checks every package against HashiCorp's GPG (GNU Privacy Guard) key, so a tampered download is rejected. Same idea as Docker's repo ([Docker topic 04](../../Docker/notes/04-installing-docker.md)).
- **Version pinning** matters because a team must run the **same** Terraform version: a newer version may upgrade the state file format in small ways, and older versions then refuse to read it (topic 62).
- **A provider cache** avoids downloading the 600 MB+ AWS provider into every lab folder.

---

## Analogy

Setting up a **workshop** before a woodworking course: buy tools from the official store (signed repo), label which tool version each project needs (version pinning), keep one shared shelf of consumables instead of buying a new box per project (provider cache), and check the power socket works (Docker).

---

## How it works inside

What happens when you run `sudo apt-get install terraform`:

```
1. apt reads /etc/apt/sources.list.d/hashicorp.list
      → "packages from https://apt.releases.hashicorp.com <ubuntu codename> main,
         signed by /usr/share/keyrings/hashicorp-archive-keyring.gpg"
2. apt-get update downloads the repo's package index and checks its signature
      against that key. Wrong signature → error, nothing installed.
3. apt-get install downloads terraform_1.17.2-1_amd64.deb, checks its hash
      against the signed index, unpacks /usr/bin/terraform.
4. Your shell finds it through PATH (/usr/bin).
```

Where Terraform keeps things on your machine:

| Path | What |
|---|---|
| `/usr/bin/terraform` | The binary (apt install) |
| `~/.terraformrc` | Your CLI config file (optional) |
| `~/.terraform.d/` | CLI data: credentials for HCP Terraform (`terraform login`), the plugin cache if you configure it |
| `<project>/.terraform/` | Per-project: downloaded providers and modules (topic 10) |

---

## Code / commands

### 1. Install Terraform (Ubuntu in WSL2)

```bash
sudo apt-get update && sudo apt-get install -y gnupg curl lsb-release

# Download HashiCorp's signing key and store it in binary (dearmored) form
curl -fsSL https://apt.releases.hashicorp.com/gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# Check the key's fingerprint against the one published on hashicorp.com/security
gpg --no-default-keyring --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg --fingerprint

# Add the repository, trusted ONLY for packages signed by that key
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt-get update
sudo apt-get install -y terraform
```

Line by line:

- `gpg --dearmor`: converts the text key to the binary format `apt` expects.
- `--fingerprint`: prints the key's fingerprint. Compare it with the one on HashiCorp's security page. If they differ, stop.
- `signed-by=...`: this key is trusted **only** for this repo, not for all of Ubuntu.
- `$(lsb_release -cs)`: your Ubuntu codename (e.g. `noble` for 24.04).

**RHEL / Rocky / Alma** (from your Linux notes' world):

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo dnf install -y terraform
```

### 2. Check it

```bash
terraform version
```

```
Terraform v1.17.2
on linux_amd64
```

- `v1.17.2`: the version. If a newer one exists, Terraform tells you here (`Your version of Terraform is out of date!`).
- `linux_amd64`: your OS and CPU. Providers are downloaded for this platform.

```bash
terraform -help
```

```
Usage: terraform [global options] <subcommand> [args]

Main commands:
  init          Prepare your working directory for other commands
  validate      Check whether the configuration is valid
  plan          Show changes required by the current configuration
  apply         Create or update infrastructure
  destroy       Destroy previously-created infrastructure

All other commands:
  console       Try Terraform expressions at an interactive command prompt
  fmt           Reformat your configuration in the standard style
  ...
```

The five **main commands** are the whole core loop (topic 09). `terraform <command> -help` shows each command's flags.

### 3. Pin the version (team labs)

```bash
apt-cache policy terraform | head -5         # versions available
sudo apt-get install -y terraform=1.17.2-1   # install an exact version
sudo apt-mark hold terraform                 # don't auto-upgrade it
```

`apt-mark unhold terraform` when you're ready to upgrade.

### 4. A version manager: tenv (optional)

If different projects need different versions, a **version manager** installs several side by side and picks the right one per folder. **tenv** handles Terraform and OpenTofu (and Terragrunt, topic 49). The older **tfenv** handles only Terraform.

```bash
# install tenv from its GitHub releases .deb (check the latest version on github.com/tofuutils/tenv)
TENV_VERSION=$(curl -s https://api.github.com/repos/tofuutils/tenv/releases/latest | jq -r .tag_name)
curl -fsSLO "https://github.com/tofuutils/tenv/releases/download/${TENV_VERSION}/tenv_${TENV_VERSION}_amd64.deb"
sudo dpkg -i "tenv_${TENV_VERSION}_amd64.deb"

tenv tf install 1.17.2
tenv tf use 1.17.2
echo "1.17.2" > .terraform-version   # per-project pin: tenv reads this file in this folder
```

> Use **either** apt **or** tenv, not both, or `which terraform` will surprise you. Check with `which -a terraform`.

### 5. Autocomplete and a provider cache

```bash
terraform -install-autocomplete      # adds a line to ~/.bashrc; open a new shell
mkdir -p ~/.terraform.d/plugin-cache
cat > ~/.terraformrc <<'EOF'
plugin_cache_dir = "$HOME/.terraform.d/plugin-cache"
EOF
```

- Type `terraform pl<Tab>` → `terraform plan`.
- `plugin_cache_dir`: `init` downloads each provider version **once** into this folder and links it into each project (topic 10).

### 6. VS Code

In Windows VS Code, install the **WSL** extension, then open your lab with `code ~/tf-lab` from the WSL terminal. In the WSL window, install the **HashiCorp Terraform** extension. You get syntax colours, completion of resource arguments, go-to-definition, and format-on-save:

```json
// settings.json
"[terraform]": { "editor.formatOnSave": true, "editor.defaultFormatter": "hashicorp.terraform" }
```

### 7. Check Docker and jq

```bash
docker version --format '{{.Server.Version}}'
docker run --rm hello-world | head -2
jq --version
```

```
29.0.2

Hello from Docker!
jq-1.7.1
```

If `docker version` says `permission denied ... docker.sock`, fix the `docker` group as in [Docker topic 04](../../Docker/notes/04-installing-docker.md).

### 8. Optional: OpenTofu side by side

```bash
curl --proto '=https' --tlsv1.2 -fsSL https://get.opentofu.org/install-opentofu.sh -o install-opentofu.sh
less install-opentofu.sh                      # read a script before running it
chmod +x install-opentofu.sh && ./install-opentofu.sh --install-method deb
tofu version
```

The installer adds OpenTofu's signed apt repo. `tofu` and `terraform` don't conflict.

---

## Hands-on lab

1. Install Terraform from the repo. Run `terraform version` and `which -a terraform`.
2. Set up autocomplete and the plugin cache. Open a new terminal and test `terraform v<Tab>`.
3. Create the lab folder structure you'll use for the rest of the course:

   ```bash
   mkdir -p ~/tf-lab && cd ~/tf-lab && git init 2>/dev/null
   # add the .gitignore from topic 05 if you haven't
   ls -la
   ```

4. Open it in VS Code (`code .`) and install the Terraform extension in the WSL window.
5. Check Docker with `docker run --rm hello-world`.
6. **Checkpoint:** you should have `terraform` (1.17.x), `docker`, `jq`, `git`, VS Code connected to WSL, and `~/tf-lab` as a Git repo.

---

## Where you'll use it

- **CI runners**: the same install steps go into a Dockerfile or a pipeline step. GitHub Actions has `hashicorp/setup-terraform` (topic 57); Jenkins agents usually bake Terraform into the agent image (topic 58).
- **Team consistency**: `required_version` in code (topic 10) plus a pinned version in CI and `.terraform-version` locally.
- **Air-gapped environments** (no internet): a manual zip with checked checksums, plus a provider mirror (topic 62).

---

## When it breaks

| Symptom | Meaning | Fix |
|---|---|---|
| `NO_PUBKEY AA16FCBCA621E701` on `apt-get update` | The key file is missing or not dearmored. | Redo the `curl ... | gpg --dearmor` step. |
| `E: Unable to locate package terraform` | The repo line wasn't added, or `apt-get update` wasn't run after adding it. | `cat /etc/apt/sources.list.d/hashicorp.list`, then `sudo apt-get update`. |
| `terraform version` shows an old version after upgrading | Another copy earlier in `PATH` (tenv, a manual zip in `/usr/local/bin`). | `which -a terraform`; remove the extra one. |
| `Error: Unsupported Terraform Core version` later | The code's `required_version` doesn't match your install. | Install the needed version (tenv), or update the code deliberately. |
| `terraform init` is very slow in WSL2 | Lab folder on the Windows drive (`/mnt/c/...`), which is slow from WSL2. | Keep labs in the Linux home (`~/tf-lab`). |
| `Error: Failed to install provider ... text file busy` with the cache | Rare race when two `init`s share the cache. | Run inits one at a time, or retry. |

---

## Quick recap

- Install from **HashiCorp's signed apt/dnf repo**; check with `terraform version`.
- Pin versions with `apt-mark hold`, or use **tenv** and a `.terraform-version` file.
- Set `plugin_cache_dir` in `~/.terraformrc` to share providers across projects.
- Keep labs in `~/tf-lab` inside WSL2, not under `/mnt/c`.

### Interview questions

1. **Q: How do you make sure everyone on a team uses the same Terraform version?**
   A: Set `required_version` in the code, pin the version in CI, and use a version manager file like `.terraform-version` locally. Upgrades go through a PR.
2. **Q: How would you install Terraform in a CI image securely?**
   A: From HashiCorp's signed package repo, or by downloading the zip plus the `SHA256SUMS` file and its signature, verifying the signature with HashiCorp's GPG key and the checksum, and pinning an exact version.
3. **Q: What is the plugin cache?**
   A: A folder set by `plugin_cache_dir` (or `TF_PLUGIN_CACHE_DIR`) where `init` stores each downloaded provider once and links it into projects, saving time and bandwidth.

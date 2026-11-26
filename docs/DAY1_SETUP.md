# Day-1 Setup — Custom-distro/fedoraOS

Complete first-day setup for the **fedoraOS** project, from zero to your own OS image running in a VM.

| Item | Value |
|---|---|
| GitHub organization | `Custom-distro` |
| Repository | `Custom-distro/fedoraOS` → https://github.com/Custom-distro/fedoraOS |
| Image name (in `recipe.yml`) | `fedoraos` |
| Published image | `ghcr.io/custom-distro/fedoraos:latest` |
| Base image | `ghcr.io/ublue-os/bazzite` (KDE Plasma, gaming-ready, Fedora-based) |
| Dev environment | VS Code + WSL running **Fedora 45** |
| Local folder | `~/fedoraOS` (inside WSL, not on the C:/E: drive) |

> **Lowercase rule:** container image addresses must be all lowercase. Your GitHub repo can be `Custom-distro/fedoraOS`, but the image is always **`ghcr.io/custom-distro/fedoraos`**. Use lowercase in `recipe.yml`, rebase commands and package names.

**Who does what:** **You** = Samrat · **Friend** = your friend · **Both** = each on your own PC.

---

## Step 0 — Install Fedora 45 in WSL (15 min, Both)

In Windows **PowerShell**:

```powershell
wsl --update
wsl --list --online
```

Find the Fedora entry in the list (for example `FedoraLinux-45`) and install it using that exact name:

```powershell
wsl --install -d FedoraLinux-45
```

When it opens, create a Linux username and password. You need the password for `sudo`; nothing shows while you type it.

Optionally, make Fedora the default WSL distro:

```powershell
wsl --set-default FedoraLinux-45
```

Inside Fedora, update and install the basic tools:

```bash
sudo dnf upgrade -y
sudo dnf install -y git gh tar gzip which nano
git --version
gh --version
```

> If `wsl --list --online` shows no Fedora 45 yet, install the newest Fedora listed. Everything below works the same.

---

## Step 1 — Create the GitHub organization (5 min, You)

1. Sign in on **github.com**.
2. Profile picture (top right) → **Your organizations** → **New organization** → **Free**.
3. Organization name: **`Custom-distro`**. Enter your email, and choose "My personal account" as the owner.
4. Invite your friend: open the org → **People** → **Invite member** → their username → role **Owner**.

**Friend:** accept the invite from email or GitHub notifications.

---

## Step 2 — Create the repo from the BlueBuild template (5 min, You)

1. While signed in, open **https://github.com/blue-build/template**.
2. Green **Use this template** button → **Create a new repository**.
   - If you can't see the button: you're not signed in, or the window is too narrow. Alternative: go to **github.com/new** and pick `blue-build/template` under *Repository template*.
3. Fill in:
   - **Owner:** `Custom-distro`
   - **Repository name:** `fedoraOS`
   - **Visibility:** **Public** (keeps GitHub Actions and the image registry free)
4. Click **Create repository**.

---

## Step 3 — Set up Git in WSL Fedora and clone the repo (10 min, Both)

### 3a. Open VS Code connected to Fedora

1. Open VS Code on Windows (the **WSL** extension must be installed).
2. Click the **><** button in the bottom-left corner → **Connect to WSL using Distro…** → choose **FedoraLinux-45**.
3. The bottom-left now says **WSL: FedoraLinux-45**.
4. Open the terminal with **Ctrl+`**.

### 3b. Git identity

```bash
git config --global user.name "Samrat"
git config --global user.email "your-github-email@example.com"
git config --global init.defaultBranch main
git config --global --list
```

> `main` is the primary branch (older projects call it `master`). This setting only affects new repos you create with `git init`.

### 3c. Log in to GitHub

```bash
gh auth login
```

Answer the prompts:
- Where do you use GitHub? → **GitHub.com**
- Preferred protocol → **HTTPS**
- Authenticate Git with your GitHub credentials? → **Yes**
- How would you like to authenticate? → **Login with a web browser** (copy the 8-character code and approve it in the browser)

```bash
gh auth setup-git
gh auth status          # should say: Logged in to github.com as <your-username>
gh org list             # should show: Custom-distro
gh repo list Custom-distro   # should show: Custom-distro/fedoraOS
```

### 3d. Clone into the WSL home folder

```bash
cd ~
gh repo clone Custom-distro/fedoraOS
cd fedoraOS
ls -la
code .
```

A new VS Code window opens on `~/fedoraOS` (still **WSL: FedoraLinux-45**).

### 3e. Check everything matches

```bash
pwd                     # /home/<your-linux-user>/fedoraOS
git remote -v           # origin https://github.com/Custom-distro/fedoraOS.git
git branch              # * main
gh repo view --json nameWithOwner,url
```

**Friend:** repeat Step 0 and Step 3 on your own PC.

---

## Step 4 — Create the image signing key (10 min, You only)

The image is signed so users' PCs can verify it really came from you.

### 4a. Install cosign

```bash
sudo dnf install -y cosign
cosign version
```

If dnf says no package `cosign`, install the official binary instead:

```bash
curl -LO https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
sudo install -m 755 cosign-linux-amd64 /usr/local/bin/cosign
rm cosign-linux-amd64
cosign version
```

### 4b. Generate the key pair (empty password)

```bash
cd ~/fedoraOS
COSIGN_PASSWORD="" cosign generate-key-pair
ls cosign.*             # cosign.key (PRIVATE) and cosign.pub (public)
```

> The template may already contain a `cosign.pub`. The command replaces it with yours, which is what you want.

### 4c. Upload the private key as a GitHub secret

```bash
gh secret set SIGNING_SECRET --repo Custom-distro/fedoraOS < cosign.key
gh secret list --repo Custom-distro/fedoraOS     # should list SIGNING_SECRET
```

### 4d. Protect and back up the private key

```bash
grep -qx "cosign.key" .gitignore || echo "cosign.key" >> .gitignore
git status              # cosign.key must NOT appear; cosign.pub should
mkdir -p ~/keys-backup && cp cosign.key ~/keys-backup/fedoraos-cosign.key
```

Also copy the backup to Windows (e.g. a password manager or USB drive). In File Explorer, `\\wsl$\FedoraLinux-45\home\<your-linux-user>\keys-backup` opens that folder.

> **Never commit `cosign.key`.** If it leaks or is lost, generate a new pair, update the secret, and everyone has to rebase again.

---

## Step 5 — Set the recipe and run the first build (20–30 min, You)

### 5a. Edit `recipes/recipe.yml`

In VS Code, open `recipes/recipe.yml` and change only the top part:

```yaml
name: fedoraos
description: Fedora-based Linux that runs your Windows apps.
base-image: ghcr.io/ublue-os/bazzite
image-version: stable
```

Leave the `modules:` section as the template has it for now.

> `image-version: stable` follows Bazzite's stable release, which tracks the current Fedora (45). Use a fixed number only if you need to pin a version.

### 5b. Commit and push

(Branch protection isn't on yet, so a direct push to `main` is fine this one time.)

```bash
git add recipes/recipe.yml cosign.pub .gitignore
git commit -m "Set base image to Bazzite, name fedoraos, add signing key"
git push
```

### 5c. Watch the build

```bash
gh run watch --repo Custom-distro/fedoraOS
```

Or on GitHub: repo → **Actions** tab. The first build takes about 15–30 minutes. A green tick means success.

### 5d. Make the image public

1. Open **github.com/Custom-distro** → **Packages** → **fedoraos**.
2. **Package settings** (bottom right) → **Change visibility** → **Public** → confirm.
3. Still in Package settings, check that **Manage Actions access** lists `fedoraOS` with **Write** role. Add it if missing.

Your image is now at `ghcr.io/custom-distro/fedoraos:latest`.

---

## Step 6 — Test VM: switch Bazzite to your image (40–60 min, You)

Always test in a VM, never on your main PC first.

### 6a. Software and ISO

- Install **VirtualBox** or **VMware Workstation Pro** (free for personal use) on Windows.
- From **bazzite.gg**, download the ISO: **Desktop**, **KDE Plasma**, GPU **AMD/Intel** (a VM's virtual GPU counts as non-NVIDIA).

### 6b. Create the VM

| Setting | Value |
|---|---|
| Type | Linux / Fedora (64-bit) |
| RAM | 8 GB (minimum 6 GB) |
| CPUs | 4 |
| Disk | 64 GB |
| Firmware | **EFI enabled** (VirtualBox: Settings → System → Enable EFI) |
| Graphics | VirtualBox: Display → Graphics Controller **VMSVGA**, 128 MB video memory |

Boot the ISO, install Bazzite, and create a user.

### 6c. Rebase onto fedoraOS

Inside the VM, open **Konsole**:

```bash
# 1st: unsigned image — installs your signing policy
rpm-ostree rebase ostree-unverified-registry:ghcr.io/custom-distro/fedoraos:latest
systemctl reboot
```

After the reboot:

```bash
# 2nd: the signed image — what real users will use
rpm-ostree rebase ostree-image-signed:docker://ghcr.io/custom-distro/fedoraos:latest
systemctl reboot
```

### 6d. Verify

```bash
rpm-ostree status
```

The booted entry (marked ●) should show `ghcr.io/custom-distro/fedoraos:latest`.

From now on, `rpm-ostree upgrade` in the VM pulls each new build, and `rpm-ostree rollback` returns to the previous one.

---

## Step 7 — Protect the `main` branch (5 min, You)

1. Repo → **Settings** → **Rules** → **Rulesets** → **New ruleset** → **New branch ruleset**.
2. Name `protect-main`, Enforcement **Active**.
3. **Target branches** → **Add target** → **Include default branch**.
4. Tick:
   - **Require a pull request before merging** → required approvals: **1**
   - **Require status checks to pass** → **Add checks** → select the **build** check (it appears after Step 5 ran once)
   - **Block force pushes**
5. **Create**.

From now on, every change to `main` goes through a pull request.

---

## Step 8 — Add CODEOWNERS through your first PR (10 min, You + Friend)

```bash
cd ~/fedoraOS
git checkout main && git pull
git checkout -b setup/codeowners
mkdir -p .github
nano .github/CODEOWNERS
```

Paste the following with your real GitHub usernames, then save (Ctrl+O, Enter, Ctrl+X):

```
/compat/       @your-github-username
/recipes/      @your-github-username
/files/        @your-github-username
/frontend/     @friend-github-username
/docs/api.md   @your-github-username @friend-github-username
```

```bash
git add .github/CODEOWNERS
git commit -m "Add CODEOWNERS"
git push -u origin setup/codeowners
gh pr create --fill
```

**Friend:** open the PR on GitHub → **Files changed** → **Review changes** → **Approve** → **Submit review**.
Then either of you clicks **Merge pull request** (wait for the build check to pass).

Afterwards, both of you run:

```bash
git checkout main && git pull
```

---

## Step 9 — Labels, project board, first issues (10 min, You)

### 9a. Labels

```bash
cd ~/fedoraOS
gh label create backend    --color 1d76db --description "fedoraos-compat / Wine / Proton"
gh label create frontend   --color 5319e7 --description "KDE GUI (Qt6 / QML / Kirigami)"
gh label create image      --color 0e8a16 --description "recipe.yml / OS image"
gh label create app-recipe --color fbca04 --description "Per-app Windows recipe"
gh label list
```

(`bug` already exists by default.)

### 9b. Project board (website)

1. **github.com/Custom-distro** → **Projects** → **New project** → **Board**.
2. Name it **fedoraOS v0.1**.
3. Set the columns to **To do / Doing / Review / Done**.
4. Project **⋯ menu** → **Settings** → **Manage access**: add your friend as **Admin**. Then in the repo → **Projects** tab → **Link a project** → pick *fedoraOS v0.1*.

### 9c. First Week-1 issues

```bash
gh issue create --title "Run Streamer.bot manually under Wine and document steps" --label backend
gh issue create --title "List 5-8 target apps and check ProtonDB / anti-cheat status" --label backend
gh issue create --title "Kirigami + PySide6 hello world" --label frontend
gh issue create --title "Mockups: Library, App Detail, Install, Settings, Welcome" --label frontend
```

Add them to the board via the project's **+ Add item** button.

---

## Step 10 — Friend's practice PR (10 min, Friend)

```bash
cd ~/fedoraOS
git checkout main && git pull
git checkout -b frontend/readme-note
echo "" >> README.md
echo "- Frontend: fedoraOS Hub (KDE Plasma / Qt6 / Kirigami)" >> README.md
git commit -am "Add frontend note to README"
git push -u origin frontend/readme-note
gh pr create --fill
```

**You:** review → **Approve** → **Merge**. Then both run `git checkout main && git pull`.

---

## Done checklist

- [ ] Fedora 45 WSL installed on both PCs, VS Code connects to it
- [ ] Org **Custom-distro** created, friend is an Owner
- [ ] Repo **Custom-distro/fedoraOS** created from the template, Public
- [ ] Both cloned to `~/fedoraOS`; `git remote -v` points to Custom-distro/fedoraOS
- [ ] `SIGNING_SECRET` saved; `cosign.pub` committed; `cosign.key` **not** committed and backed up
- [ ] First build green; package **fedoraos** set to Public
- [ ] VM `rpm-ostree status` shows `ghcr.io/custom-distro/fedoraos:latest`
- [ ] `main` protected; CODEOWNERS merged via PR
- [ ] Labels, board and first issues created
- [ ] Friend's practice PR merged

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `git: command not found` | `sudo dnf install -y git gh` |
| No **Use this template** button | Sign in to GitHub, or use github.com/new → Repository template |
| `gh repo clone` says not found | Check the exact names: `gh repo list Custom-distro` |
| Build fails at **cosign / signing** | The secret is missing or the key has a password. Regenerate with `COSIGN_PASSWORD=""` and set the secret again. |
| Build fails with **denied / permission** when pushing the image | Org → Settings → Actions → General → Workflow permissions → **Read and write**; in the package settings, give the `fedoraOS` repo **Write** access |
| Build error about **uppercase** / invalid reference | `name:` in `recipe.yml` must be lowercase: `fedoraos` |
| VM rebase: **unauthorized** / **manifest unknown** | Package is still private, the build isn't finished, or there's a typo. It must be `ghcr.io/custom-distro/fedoraos` |
| Signed rebase fails | Do the unverified rebase + reboot first (Step 6c) |
| `git push` to main rejected | Branch protection is working. Use a branch + PR. |
| VS Code can't connect to the Fedora WSL | Run `sudo dnf install -y tar gzip which` in Fedora, then reconnect |

---

**Next:** Week 1 of `PROJECT_PLAN.md`: Streamer.bot under Wine (You) and Kirigami mockups (Friend).

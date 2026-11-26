# fedoraOS — Project Plan

> A Fedora Atomic–based Linux OS that runs a chosen set of Windows-only apps (games, Streamer.bot, and more) in one click, through a native KDE Plasma app.

| | |
|---|---|
| **Repository** | [Custom-distro/fedoraOS](https://github.com/Custom-distro/fedoraOS) |
| **OS image** | `ghcr.io/custom-distro/fedoraos:latest` |
| **Base** | [Bazzite](https://bazzite.gg) (`ghcr.io/ublue-os/bazzite:stable`), Fedora Atomic, KDE Plasma 6 |
| **Build system** | [BlueBuild](https://blue-build.org) + GitHub Actions, images signed with cosign |
| **Target** | v0.1 in 12 weeks |
| **Doc status** | Living document. Update it through pull requests. |

---

## Contents

1. [Goal and scope](#1-goal-and-scope)
2. [Team and ownership](#2-team-and-ownership)
3. [Decisions](#3-decisions)
4. [Architecture](#4-architecture)
5. [Repository layout](#5-repository-layout)
6. [Backend ↔ GUI contract](#6-backend--gui-contract)
7. [App recipes](#7-app-recipes)
8. [Roadmap: week by week](#8-roadmap-week-by-week)
9. [Development workflow](#9-development-workflow)
10. [Testing](#10-testing)
11. [Release and install](#11-release-and-install)
12. [Risks](#12-risks)
13. [Glossary](#13-glossary)
14. [References](#14-references)

---

## 1. Goal and scope

### Goal
Make Windows-only apps work on Linux **reliably and without manual Wine tweaking**: the user opens **fedoraOS Hub**, clicks *Install*, then *Launch*.

### In scope for v0.1
- An OS image built from `recipes/recipe.yml`, auto-built and signed by CI
- `fedoraos-compat`: a CLI that installs, runs, repairs and removes Windows apps using per-app recipes
- **fedoraOS Hub**: a KDE Plasma GUI (Qt 6 / QML / Kirigami) on top of the CLI
- **5–8 target apps** that work well (Streamer.bot first)
- Install by **rebase** (from any Fedora Atomic / Bazzite system) and by **ISO**

### Out of scope (for now)
- Games with **kernel-level anti-cheat** (Valorant, many Call of Duty titles, etc.). Wine/Proton can't run kernel drivers, so these are listed as *unsupported* with the reason, never "fixed".
- Supporting "every" Windows app. Quality over quantity.
- Our own package repository, kernel, or installer. Fedora, Universal Blue and BlueBuild provide these.

### Why Fedora Atomic + BlueBuild (and not Arch + archiso)
| Concern | Arch + archiso | Fedora Atomic + BlueBuild (chosen) |
|---|---|---|
| Updating installed users | Own signed pacman repo + PKGBUILDs | `git push` → CI builds a new image → users get it on their next update |
| Breakage protection | Rolling updates hit users directly | Broken builds fail in CI; users roll back at boot |
| Installer | Maintain Calamares | Rebase, or ISO generated from the image |
| Hosting | ISOs + repo need external hosting | GitHub Actions + GHCR are free for public repos |
| Recurring chore | Chasing rolling breakage, ISO/keyring rot | Bump the Fedora release about every 6 months |

---

## 2. Team and ownership

| Member | Role | Owns |
|---|---|---|
| **Samrat** | Backend + OS image | `compat/`, `recipes/`, `files/`, CI, app recipes, VM/hardware testing |
| **Friend** | Frontend + branding | `frontend/`, Plasma branding, UI/UX |
| **Both** | Shared | `docs/api.md`, this plan, releases |

Ownership is enforced by `.github/CODEOWNERS`: GitHub automatically requests a review from the owner of the changed folder.

---

## 3. Decisions

| # | Topic | Decision | Status |
|---|---|---|---|
| 001 | Base distro | Fedora Atomic via BlueBuild (not Arch + archiso) | ✅ Decided |
| 002 | Base image | `ghcr.io/ublue-os/bazzite`, tag `stable` (KDE, gaming-ready; Steam, Gamescope and drivers handled upstream) | ✅ Decided |
| 003 | Desktop | KDE Plasma 6 | ✅ Decided |
| 004 | GUI toolkit | Qt 6 + QML + **Kirigami**, Python via **PySide6** | ✅ Decided |
| 005 | Backend language | Python 3 | ✅ Decided |
| 006 | GUI ↔ backend | CLI with `--json` output and JSON-lines progress (D-Bus later if needed) | ✅ Decided |
| 007 | Runners | Wine for apps; umu-launcher + Proton for games; per-app override | ✅ Decided |
| 008 | Image visibility | Public on GHCR (required for rebase without login) | ✅ Decided |
| 009 | Dev environment | VS Code + WSL (Fedora), repo in `~/fedoraOS`; test in VMware VM | ✅ Decided |

New decisions get a short file in `docs/decisions/NNN-title.md` (context → decision → consequences) and a row here.

---

## 4. Architecture

```
┌──────────────────────────────────────────────┐
│  fedoraOS Hub (GUI)                — Friend  │
│  QML + Kirigami pages                        │
│  PySide6 layer: models + QProcess wrapper    │
└──────────────────────┬───────────────────────┘
                       │  fedoraos-compat <command> --json
                       │  (stdout: JSON / JSON lines)
┌──────────────────────▼───────────────────────┐
│  fedoraos-compat (CLI)             — Samrat  │
│  recipes · prefixes · runners · logs         │
└──────────────────────┬───────────────────────┘
                       │
        Wine  /  umu-launcher + Proton  /  winetricks
                       │
   ~/.local/share/fedoraos/prefixes/<app-id>/   (one Wine prefix per app)
```

### Where things live on an Atomic system

| What | Path | Notes |
|---|---|---|
| CLI and GUI programs | `/usr/bin/fedoraos-compat`, `/usr/bin/fedoraos-hub` | Shipped in the image (read-only) |
| App recipes, icons, QML | `/usr/share/fedoraos/` | Shipped in the image, updated with it |
| Default system config | `/usr/etc/` → `/etc/` | Image ships defaults; users can override |
| Wine prefixes, downloads | `~/.local/share/fedoraos/` | Per user, writable, survives updates |
| Logs, cache | `~/.cache/fedoraos/` | Per user |
| User settings | `~/.config/fedoraos/` | Per user |

---

## 5. Repository layout

```
fedoraOS/
├── recipes/
│   └── recipe.yml              # BlueBuild OS definition
├── files/
│   ├── system/                 # copied into the image root (e.g. usr/bin, usr/share/fedoraos)
│   └── scripts/                # build-time scripts
├── compat/                     # Backend — Samrat
│   ├── fedoraos_compat/
│   │   ├── cli.py              # argument parsing, --json output
│   │   ├── recipes.py          # load + validate app recipes
│   │   ├── prefix.py           # create / delete / repair Wine prefixes
│   │   ├── runner.py           # Wine and umu/Proton runners
│   │   └── events.py           # JSON-lines progress events
│   ├── apps/                   # one YAML recipe per Windows app
│   └── tests/
├── frontend/                   # GUI — Friend (KDE / Qt 6)
│   ├── fedoraos_hub/
│   │   ├── main.py             # QGuiApplication + QML engine
│   │   ├── backend.py          # QProcess wrapper around fedoraos-compat
│   │   └── models.py           # QAbstractListModel for apps
│   ├── qml/
│   │   ├── Main.qml            # Kirigami.ApplicationWindow + navigation
│   │   ├── LibraryPage.qml
│   │   ├── AppDetailPage.qml
│   │   ├── InstallDialog.qml
│   │   ├── SettingsPage.qml
│   │   └── WelcomePage.qml
│   ├── data/                   # .desktop file, icons
│   ├── mock/                   # fake CLI JSON for UI work without the backend
│   └── tests/
├── docs/
│   ├── PROJECT_PLAN.md         # this file
│   ├── DAY1_SETUP.md           # environment + repo setup guide
│   ├── api.md                  # CLI ↔ GUI contract
│   ├── app-status.md           # per-app support status
│   ├── decisions/              # decision records
│   └── notes/                  # research and manual test notes
├── .github/
│   ├── workflows/              # build.yml (image), tests.yml (lint + tests)
│   └── CODEOWNERS
├── cosign.pub
└── README.md
```

---

## 6. Backend ↔ GUI contract

Full spec lives in `docs/api.md`. **Changing it requires a PR approved by both members.**

### Commands

| Command | Purpose | Output |
|---|---|---|
| `fedoraos-compat list --json` | All apps with status | JSON object |
| `fedoraos-compat info <app> --json` | One app's details | JSON object |
| `fedoraos-compat install <app> --json` | Install | JSON lines (progress) |
| `fedoraos-compat run <app>` | Launch | exit code |
| `fedoraos-compat uninstall <app> --json` | Remove app + prefix | JSON lines |
| `fedoraos-compat repair <app> --json` | Rebuild prefix, keep user data | JSON lines |
| `fedoraos-compat doctor --json` | GPU, Vulkan, drivers, disk | JSON object |

### `list` example
```json
{"apps": [
  {"id": "streamerbot", "name": "Streamer.bot", "status": "installed", "support": "works",
   "icon": "/usr/share/fedoraos/icons/streamerbot.png"},
  {"id": "valorant", "name": "Valorant", "status": "not_installed", "support": "unsupported",
   "reason": "Kernel anti-cheat (Vanguard)"}
]}
```

### Progress stream (one JSON object per line)
```json
{"event": "step", "step": "create_prefix", "progress": 10}
{"event": "step", "step": "install_dotnet", "progress": 45}
{"event": "done", "ok": true}
{"event": "error", "code": "DOWNLOAD_FAILED", "message": "Could not download installer"}
```

### Fixed values
| Field | Values |
|---|---|
| `status` | `installed` · `not_installed` · `installing` · `broken` |
| `support` | `works` · `partial` · `unsupported` |
| Exit code | `0` = success; non-zero = error (details in the last `error` event) |

---

## 7. App recipes

Each supported app is one file in `compat/apps/<id>.yaml`:

```yaml
id: streamerbot
name: Streamer.bot
support: works            # works | partial | unsupported
reason: ""                # required when support is not "works"
runner: wine              # wine | proton
runner_version: default   # pin a version if an update breaks the app
prefix_arch: win64
winetricks: [dotnet48, corefonts]
download:
  url: https://example.com/streamerbot.zip   # real URL in implementation
  type: zip               # zip | exe | msi
executable: "Streamer.bot.exe"
env: {}                   # extra environment variables
notes: "Needs .NET Framework in the prefix."
```

Every recipe PR must include a manual test result in `docs/app-status.md`.

---

## 8. Roadmap: week by week

Each week ends with a **15-minute demo to each other** and merged PRs. Week 1 starts after Day-1 setup ([`DAY1_SETUP.md`](DAY1_SETUP.md)).

### Day-1 setup status
- [x] GitHub org `Custom-distro` + repo `fedoraOS` from the BlueBuild template
- [x] Repo cloned in WSL, `gh` authenticated
- [x] cosign key pair; `SIGNING_SECRET` set; `cosign.pub` committed
- [x] First image build green (`recipe.yml` → Bazzite, name `fedoraos`)
- [ ] Image package set to **Public**
- [ ] VM: Bazzite installed and rebased to `ghcr.io/custom-distro/fedoraos:latest`
- [ ] `SIGNING_SECRET` also added as a Dependabot secret; Dependabot PR merged
- [ ] `main` branch protection, CODEOWNERS, labels, project board
- [ ] Friend: WSL + clone + practice PR

### Phase 1 — Foundations (Weeks 1–3)

| Week | Backend (Samrat) | Frontend (Friend) | Done when |
|---|---|---|---|
| **1** | Target app list (5–8), checked on ProtonDB / Are We Anti-Cheat Yet. Run **Streamer.bot manually** under Wine; document every step in `docs/notes/streamerbot.md`. | Kirigami tutorial + PySide6/QML "hello world". Mockups (KDE HIG): Library, App Detail, Install, Settings, Welcome. | App list and mockups reviewed together. |
| **2** | `fedoraos-compat` skeleton: `list`, `info`, prefix manager, `--json`. pytest. | Scaffold `fedoraos_hub`: `Kirigami.ApplicationWindow`, sidebar navigation, mock backend from `frontend/mock/*.json`. | `docs/api.md` v0.1 merged. CI runs backend tests + frontend lint (ruff, `qmllint`). |
| **3** | Recipe loader + validation. `install` / `uninstall` Streamer.bot (Wine + winetricks) with progress events. | `LibraryPage.qml`: `Kirigami.CardsGridView` + Python list model; status/support chips; unsupported apps greyed out with reason. | Streamer.bot installs from the CLI on a clean VM. |

### Phase 2 — First working loop (Weeks 4–6)

| Week | Backend | Frontend | Done when |
|---|---|---|---|
| **4** | `run`; logs in `~/.cache/fedoraos/logs/`; error codes; umu/Proton runner. | `InstallDialog.qml`: progress from JSON lines, cancel, `Kirigami.InlineMessage` errors + "Show log". | Mock JSON matches real CLI output. |
| **5** | Ship CLI + recipes in the image via `files/system` + `recipe.yml` packages. | `backend.py`: real calls via **QProcess** (non-blocking). Ship the GUI + `.desktop` file in the image. | **🎯 Milestone 1:** in the VM, Plasma menu → fedoraOS Hub → Install Streamer.bot → Launch → it runs. |
| **6** | 2–3 more recipes (≥1 game via Proton). `.desktop` entries + icon extraction for installed apps. | `AppDetailPage.qml`: Launch / Repair / Uninstall actions, "Open prefix folder" (Dolphin). | `docs/app-status.md` covers every target app. |

### Phase 3 — Real-world quality (Weeks 7–9)

| Week | Backend | Frontend | Done when |
|---|---|---|---|
| **7** | `repair`, `doctor`; per-app runner/version override. | `SettingsPage.qml` (Kirigami Addons `FormCard`); "System check" page showing `doctor` output. | Settings reach the backend. |
| **8** | **Real hardware tests** (AMD and/or NVIDIA); GameMode / MangoHud per app; bug fixes. | `WelcomePage.qml` first-run flow; Breeze light/dark; keyboard + large touch targets. | `app-status.md` updated from real hardware. |
| **9** | Image polish: default Flatpaks (e.g. OBS), trim unneeded packages, nightly CI build. | Branding: wallpaper, look-and-feel defaults, Hub icon, SDDM theme / Plymouth (optional). | Nightly builds green 7 days in a row. |

### Phase 4 — Release (Weeks 10–12)

| Week | Backend | Frontend | Done when |
|---|---|---|---|
| **10** | Installer ISO (`bluebuild generate-iso` or bootc-image-builder); rebase guide. | Error/empty states: offline, download failed, missing driver. | **Alpha:** 3–5 testers install and file Issues. |
| **11** | Fix alpha bugs; per-recipe test checklist. | Fix alpha bugs; UX fixes. | All `critical` issues closed. |
| **12** | Freeze recipes; tag `v0.1.0`. | Final polish; README screenshots. | **🚀 v0.1 released.** |

---

## 9. Development workflow

### Environment
- **Editor:** VS Code connected to **WSL (Fedora)**; repo at `~/fedoraOS` (Linux home, not `/mnt/c` or `/mnt/e`).
- **Tools:** `git`, `gh`, `python3`, `cosign` (Samrat only), `skopeo` (optional).
- **Testing OS:** VMware VM running Bazzite → rebased to fedoraOS. Never test new images on your main PC first.
- **Codespaces** is optional for editing, but it can't run the VM or show the GUI.

### Branches and pull requests
- `main` is protected: changes go in only through **pull requests**, with **1 approval** and **green checks**.
- Branch names: `feat/<name>`, `fix/<name>`, `recipe/<app-id>`, `docs/<name>`, `frontend/<name>`, `setup/<name>`.
- One Issue → one branch → one PR. Put `Closes #<issue>` in the PR description.
- Keep PRs small (ideally under 300 changed lines).

```bash
git checkout main && git pull
git checkout -b feat/prefix-manager
# ... work ...
git add -A && git commit -m "compat: add prefix manager"
git push -u origin feat/prefix-manager
gh pr create --fill
```

### Commit messages
`<area>: <what changed>`, where area is one of `compat`, `frontend`, `image`, `recipe`, `docs`, `ci`.
Examples: `compat: add repair command`, `frontend: install progress dialog`, `image: add winetricks`.

### Labels
`backend` · `frontend` · `image` · `app-recipe` · `bug` · `critical`

### Project board
**fedoraOS v0.1**: To do → Doing → Review → Done. Move your card when you start and when you open the PR.

### Weekly rhythm
| When | What |
|---|---|
| Start of week | Pick Issues for the week on the board |
| During week | Small PRs, review each other within 24 h |
| End of week | 15-min demo, update this plan's checkboxes and `app-status.md` |

### Secrets
- `cosign.key` is **never** committed (it's in `.gitignore`). The backup is kept offline.
- `SIGNING_SECRET` exists as an **Actions** secret and a **Dependabot** secret.

---

## 10. Testing

| Level | What | Where |
|---|---|---|
| Unit | Backend logic (`pytest`), frontend models (`pytest-qt`) | CI on every PR |
| Lint | `ruff` (Python), `qmllint` (QML), recipe schema check | CI on every PR |
| Image build | BlueBuild workflow | CI on every PR + nightly |
| VM | Rebase, install/launch each recipe | Samrat's VMware VM |
| Hardware | Performance, GPU drivers, real games | Both PCs (Week 8+) |

### Updating the test VM
```bash
rpm-ostree upgrade && systemctl reboot   # get the newest build
rpm-ostree status                        # confirm the booted image
rpm-ostree rollback && systemctl reboot  # go back if something broke
```

### App test matrix (keep in `docs/app-status.md`)
| App | Runner | VM | AMD PC | NVIDIA PC | Status | Notes |
|---|---|---|---|---|---|---|
| Streamer.bot | Wine | | | | | |
| Game 1 | Proton | | | | | |
| Game 2 | Proton | | | | | |
| App 2 | Wine | | | | | |

---

## 11. Release and install

### Versioning
`vMAJOR.MINOR.PATCH`, starting at `v0.1.0`. Git tags mark releases; the image keeps rolling on `latest`.

### Install by rebase (from any Bazzite / Fedora Atomic system)
```bash
rpm-ostree rebase ostree-unverified-registry:ghcr.io/custom-distro/fedoraos:latest
systemctl reboot
rpm-ostree rebase ostree-image-signed:docker://ghcr.io/custom-distro/fedoraos:latest
systemctl reboot
```

### Install by ISO
Generated from the image (Week 10). Too large for GitHub Releases, so hosted externally (e.g. SourceForge or cloud storage), with the link in the README.

### Verify the image signature
```bash
cosign verify --key cosign.pub ghcr.io/custom-distro/fedoraos:latest
```

### Fedora major upgrades (about every 6 months)
Bazzite's `stable` tag moves to the new Fedora automatically. When it does, watch the CI build; fix or temporarily drop any COPR / third-party package that hasn't caught up.

---

## 12. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Kernel anti-cheat games | Can't run at all | Mark `unsupported` with the reason; don't spend time on them |
| Wine/Proton update breaks an app | App stops working | Pin `runner_version` per recipe; test before bumping |
| CI fails after a Fedora major release | No new images | Fix or drop lagging packages; users stay on the last good image |
| NVIDIA driver issues | Black screen / poor performance | Use the Bazzite `-nvidia` image variant; test on real NVIDIA in Week 8 |
| Lost or leaked `cosign.key` | Can't sign / untrusted images | Offline backup; regenerate and rotate if leaked |
| ISO too large for GitHub | Distribution | Rebase is the main install path; external ISO hosting |
| Scope creep | v0.1 slips | 5–8 well-tested apps; new apps go to the backlog |
| One member unavailable | Work blocks | `api.md` contract + mock JSON let each side keep working independently |

---

## 13. Glossary

| Term | Meaning |
|---|---|
| **Fedora Atomic** | Fedora variants (Silverblue, Kinoite…) where the OS is a read-only image updated as a whole |
| **Bazzite** | Gaming-focused Fedora Atomic image from Universal Blue; our base |
| **BlueBuild** | Tool that turns `recipe.yml` into an OS container image |
| **GHCR** | GitHub Container Registry, where our image is published |
| **rpm-ostree / bootc** | Tools that install, update and roll back image-based Fedora |
| **Rebase** | Switching an installed Atomic system to a different image |
| **cosign** | Tool that signs images so systems can verify they came from us |
| **Wine** | Compatibility layer that runs Windows programs on Linux |
| **Proton / umu-launcher** | Valve's gaming-tuned Wine, and the launcher that runs it outside Steam |
| **Prefix** | A self-contained fake Windows folder (`C:\` drive) for one app |
| **winetricks** | Script that installs Windows components (.NET, fonts, DLLs) into a prefix |
| **Kirigami** | KDE's QML UI framework for native Plasma apps |
| **PySide6** | Official Python bindings for Qt 6 |
| **Dependabot** | GitHub bot that opens PRs to update dependencies such as the BlueBuild action |

---

## 14. References

- BlueBuild docs: https://blue-build.org · template: https://github.com/blue-build/template
- Universal Blue / Bazzite: https://universal-blue.org · https://bazzite.gg
- Kirigami tutorial: https://develop.kde.org/docs/getting-started/kirigami/
- KDE Human Interface Guidelines: https://develop.kde.org/hig/
- Qt for Python (PySide6): https://doc.qt.io/qtforpython-6/
- umu-launcher: https://github.com/Open-Wine-Components/umu-launcher
- winetricks: https://github.com/Winetricks/winetricks
- ProtonDB: https://www.protondb.com
- Are We Anti-Cheat Yet: https://areweanticheatyet.com

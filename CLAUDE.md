# CLAUDE.md — fedoraOS

Context for Claude Code. Read this first, then `docs/PROJECT_PLAN.md` (full plan) and `docs/DAY1_SETUP.md` (setup guide).

## What this project is
fedoraOS is a Fedora Atomic–based Linux OS, built with BlueBuild, that runs a chosen set of Windows-only apps (Streamer.bot first, then 5–8 games/apps) in one click through a native KDE Plasma app called **fedoraOS Hub**.

- Repo: `Custom-distro/fedoraOS` (GitHub org `Custom-distro`)
- Image: `ghcr.io/custom-distro/fedoraos:latest` (**always lowercase** in image names)
- Base: `ghcr.io/ublue-os/bazzite`, tag `stable` (KDE Plasma 6, Fedora Atomic)
- Build: BlueBuild GitHub Action, images signed with cosign (`SIGNING_SECRET`)
- Target: v0.1 in 12 weeks (roadmap in `docs/PROJECT_PLAN.md` §8)

## Team and ownership
- **Samrat**: backend + OS image → `compat/`, `recipes/`, `files/`, CI, app recipes, VM testing
- **Friend**: frontend + branding → `frontend/`, Plasma branding
- **Shared**: `docs/api.md` (needs both approvals), `docs/PROJECT_PLAN.md`

## Decided (don't re-open without asking)
- Fedora Atomic + BlueBuild (not Arch + archiso)
- KDE Plasma 6; GUI = Qt 6 + QML + **Kirigami**, in Python via **PySide6**
- Backend = Python 3 CLI `fedoraos-compat`; GUI calls it through **QProcess** with `--json` output and JSON-lines progress events
- Runners: Wine for apps, umu-launcher + Proton for games, per-app override
- Kernel anti-cheat games (Valorant etc.) are marked `unsupported`, never worked around

## Names (use these, not the old `myos` names)
| Thing | Name |
|---|---|
| CLI command | `fedoraos-compat` |
| Backend package | `compat/fedoraos_compat/` |
| GUI package | `frontend/fedoraos_hub/` |
| Desktop file | `frontend/data/org.fedoraos.Hub.desktop` |
| Shipped data | `/usr/share/fedoraos/` (source: `files/system/usr/share/fedoraos/`) |
| User data | `~/.local/share/fedoraos/`, `~/.cache/fedoraos/`, `~/.config/fedoraos/` |
| App recipes | `compat/apps/<id>.yaml` |

## Atomic system rules
- `/usr` is read-only on installed systems: ship programs and data in the image via `files/system/` and `recipes/recipe.yml`, never by writing to `/usr` at runtime.
- Per-user, writable things (Wine prefixes, logs, settings) go under the user's home.
- Packages are added in `recipes/recipe.yml` (`dnf` module); desktop apps preferably as Flatpaks (`default-flatpaks` module).

## Workflow rules
- `main` is protected: work on a branch (`feat/`, `fix/`, `recipe/`, `docs/`, `frontend/`, `setup/`) and open a PR with `gh pr create --fill`.
- Commit messages: `<area>: <what>` where area ∈ `compat`, `frontend`, `image`, `recipe`, `docs`, `ci`.
- Keep PRs small; link issues with `Closes #N`.
- Changes to `docs/api.md` need both members' approval.

## Safety rules for Claude Code
- **Ask before** pushing, merging PRs, changing GitHub settings/secrets, or deleting files/branches.
- **Never** commit, print or upload `cosign.key` (it's in `.gitignore`; only `cosign.pub` is committed).
- Don't run `rpm-ostree` here: this is a WSL dev environment. OS testing happens in a VMware VM running Bazzite rebased to fedoraOS.

## Environment
- Developers use VS Code connected to WSL (Fedora), repo at `~/fedoraOS`.
- Tools available: `git`, `gh` (logged in), `python3`; `cosign` and `skopeo` may be installed.
- Useful checks:
  - `gh run list --limit 5`: image builds
  - `gh pr list --state all`: PRs
  - `gh api repos/Custom-distro/fedoraOS/rulesets --jq '.[].name'`: branch rules
  - `gh secret list` / `gh secret list --app dependabot`: secrets exist

## Current status (update as you go)
- Done: org + repo, cosign key + `SIGNING_SECRET`, first green build with Bazzite base.
- Open: PR #1 (Dependabot, BlueBuild action 1.13 → 1.14), PR #2 (docs), branch `feat/repo-layout` (rename `myos` → `fedoraos`), CODEOWNERS, branch protection, image public check, VM rebase.
- `recipes/recipe.yml` still contains template example modules (micro, starship, Firefox/Loupe flatpaks); replace them in Week 1.

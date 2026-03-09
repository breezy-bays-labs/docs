---
pipeline: m1.5-workspace-observability
stage: frame
---

# Frame: Soji M1.5 — Workspace Observability

## Source Material

Discovery interview session (2026-03-06) produced clear requirements and locked design decisions for all M1.5 sections. Source documents:

- `docs/workspace/m1.5-workspace-observability/user-stories.md` — 7 journeys: readiness check (A), env debugging (B), secret provisioning (C), GitHub state (D), label health (E), worktrees (F), config inventory (G).
- `~/Github/SECRETS-MANAGEMENT.md` — canonical secrets architecture. `.envrc` is the declared layer; `ks` + macOS Keychain is the secret store; direnv loads vars into shell. `ks ls` uses `<scope>/<category>/<key>` naming.
- `ks ls` output — real keychain structure: `mokumo/` (workspace-scoped), `shared/` (org-level).
- `docs/user-stories.md` — global journey J2 (context switch) and J7 (env debugging) are the primary journeys this milestone serves.
- `docs/vision.md` — Workspace Observability layer: env vars, secrets, config files, git state all belong at the workspace level.
- `~/.claude/projects/-Users-cmbays-Github-soji/memory/MEMORY.md` — locked design decisions: secret management uses `security`/`ks` directly, workspace vs. surface distinction, TUI interaction model.

### What is already built (pre-M1.5)

M1 shipped the TUI shell with stub sections in Workspace Mode:

- **Left panel** — ENV, SECRETS, GIT section headers exist as `NavItem::SectionHeader(SectionKind::*)` in the nav. Not expandable. No data behind them.
- **Right panel** — Section detail views show "coming in M1.5" placeholder text.
- **Domain** — `Workspace` struct has `name`, `cwd`, `git_branch`, `surfaces` only. No env/secrets/git/github/config fields.
- **Sources** — No env, secrets, git, or GitHub source modules exist.
- **Keybindings** — Section headers are already in the nav but `Enter` on them does nothing meaningful (no expansion logic wired).

---

## Problem Statement

M1 shipped an observable process/port layer. But the workspace environment itself — the env vars, secrets, git state, and project status that define whether a workspace is *ready to work in* — is completely invisible.

A developer opening Soji in a workspace today can see their Claude sessions and listening ports. They cannot see:
- Whether `SUPABASE_SERVICE_ROLE_KEY` is loaded or missing
- Which secrets the workspace declared in `.envrc` vs. which are provisioned in the keychain
- The current git branch, dirty status, or open worktrees
- What milestones are active and what issues are left in them
- Whether a `.envrc` or `docker-compose.yml` is even present

The section headers for ENV, SECRETS, and GIT are visible stubs — but they deliver nothing. Journey 2 (context switch) and Journey 7 (env debugging) are entirely unserved.

---

## Outcome Definition

A workspace developer opens Soji in their workspace CWD and, without running any shell commands, can immediately see:

1. **ENV** — Every var declared in `.envrc`, whether it's currently loaded in the shell, whether it's secret-backed (and the `ks` path), and whether anything is missing. The gap between declared and loaded is visible at a glance.

2. **SECRETS** — Every secret the workspace needs (from `.envrc` parsing), whether it exists in the keychain, and whether it's currently loaded. `shared/` secrets visually distinguished from workspace-scoped ones.

3. **GIT** — Current branch, dirty/clean status, open worktrees, stash count.

4. **GITHUB** — Open milestones with issue counts; drill into a milestone to see open issues with their labels; label health view showing what labels are actively used.

5. **CONFIG** — Which config files are present (`.envrc`, `docker-compose.yml`, `soji.toml`, etc.); view `.envrc` contents inline.

The milestone is complete when a developer can fully assess workspace readiness (Journey A) and diagnose a missing-env-var failure (Journey B) entirely within the Soji TUI.

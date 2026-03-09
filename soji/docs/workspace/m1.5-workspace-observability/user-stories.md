---
title: M1.5 User Stories — Workspace Observability
description: 'User journeys and stories scoped to M1.5: env vars, secrets, git state, GitHub integration, and config file inventory.'
category: product
status: draft
milestone: M1.5
last_updated: 2026-03-06
depends_on:
  - user-stories (global)
  - vision
---

# M1.5 User Stories — Workspace Observability

> These stories are scoped to M1.5 only. They refine and extend the global user stories doc for the specific capabilities being built: the ENV, SECRETS, GIT/GitHub, and Config sections of Workspace Mode.

---

## Background: What M1.5 is building

M1.5 extends the Workspace Mode left panel — the ENV, SECRETS, GIT, and CONFIG section headers that exist as stubs in M1 — into real, live data sections. The core value: a developer can open Soji inside a workspace and immediately see the complete environment state without running any commands.

### The secrets architecture (relevant context)

All secrets flow through macOS Keychain via `ks`, surfaced to the shell via `direnv`:

```
Keychain (ks)  →  .envrc (declared, committed to git)  →  Shell env (loaded)
```

Secrets use path-based naming: `<scope>/<category>/<key>`
- `shared/` — account-level, used across multiple workspaces
- `<workspace>/` — workspace-specific infrastructure (e.g. `mokumo/supabase/url`)

The `.envrc` is the source of truth for what a workspace declares it needs. It's committed to git, so it's always present — including in worktrees.

---

## Core Journeys

### Journey A: "Is this workspace ready to work in?"

**Context**: Developer context-switches to a workspace they haven't been in for a few days. Before starting work, they want to verify the environment is healthy.

**Trigger**: Open Soji inside the workspace. Navigate to ENV and SECRETS sections.

**Desired experience**:
1. ENV section shows all vars declared in `.envrc` — with a green check if loaded, red flag if missing.
2. At a glance: `ANTHROPIC_API_KEY` loaded ✓, `MOKUMO_DB_URL` loaded ✓, `STRIPE_SECRET_KEY` ✗ missing.
3. SECRETS section shows the ks entries this workspace needs — exists in keychain and loaded status.
4. Developer can see exactly what's missing before they write a line of code.
5. GIT section shows current branch, whether the worktree is clean or dirty, and stash count.

**User stories**:
- As a developer, I want to see all env vars declared by my workspace and whether they are currently loaded, so I know if my environment is ready before I start work.
- As a developer, I want to immediately see which secrets are missing from my keychain so I can add them before hitting a runtime error.
- As a developer, I want to see git state (branch, dirty, stashes) alongside env state so I have a complete readiness picture in one view.

---

### Journey B: "Why is my code broken?"

**Context**: Developer hits an unexpected runtime error — likely a missing or wrong environment variable. They want to diagnose without dropping to a shell and running multiple commands.

**Trigger**: Error in app. Open Soji → ENV section.

**Desired experience**:
1. ENV section shows the gap clearly: `UPSTASH_REDIS_TOKEN` is declared in `.envrc` but not loaded.
2. The source shows: `$(secret mokumo/upstash/redis-token)` — the ks path is visible.
3. Developer can see the entry doesn't exist in keychain either — that's the root cause.
4. They drop to shell, run `ks add mokumo/upstash/redis-token <value>`, then `direnv allow`.
5. Reload Soji — the entry now shows loaded ✓.

**User stories**:
- As a developer, I want to see not just that a var is missing but *why* — is it absent from the keychain, or is direnv just not loaded?
- As a developer, I want to see the ks path for each secret-backed env var so I know exactly what to add without reading the .envrc manually.
- As a developer, I want the diagnostic flow (identify → fix → verify) to stay inside Soji without multiple terminal commands.

---

### Journey C: "What secrets does this workspace need?"

**Context**: Developer sets up a workspace on a new machine, or provisions a new worktree and needs to ensure secrets are present.

**Trigger**: `cd` into a workspace or worktree. Open Soji → SECRETS section.

**Desired experience**:
1. SECRETS section lists every ks path referenced in `.envrc` — this workspace's full secret dependency list.
2. Each entry shows: path, category (supabase / upstash / auth / etc.), scope (workspace vs shared), exists-in-keychain, loaded.
3. Red entries = not in keychain → developer knows exactly what to provision.
4. Shared secrets (e.g. `shared/ai/anthropic-api-key`) are visually distinguished — they're org-level, not workspace-specific.
5. Developer provisions only the missing entries and re-loads.

**User stories**:
- As a developer, I want to see a complete secret manifest for the workspace — what it needs, what exists, what's missing — without reading the .envrc manually.
- As a developer, I want workspace-scoped secrets and shared/org-level secrets to be visually separated so I understand which keys belong to which tier.
- As a developer, I want the secrets section to serve as a setup checklist when provisioning the workspace on a new machine.

---

### Journey D: "What's the state of work in this repo?"

**Context**: Developer opens their workspace Soji pane to orient themselves — what milestone is active, what's left to do, are there blocked issues?

**Trigger**: Open Soji → GITHUB section. Or navigate there as part of daily orientation.

**Desired experience**:
1. GitHub section shows open milestones for the repo, with a count of open vs closed issues per milestone.
2. Expand a milestone → see its open issues, each with title, assignee, and labels.
3. Developer can see at a glance: "M1.5 has 8 issues, 3 are in-progress, 5 open."
4. Labels on open issues are visible — developer can spot label sprawl (too many labels, duplicates, confusion).
5. Progressive disclosure: workspace → milestones → issues. Not everything at once.

**User stories**:
- As a developer, I want to see my open milestones and the issues inside them without leaving Soji or opening a browser.
- As a developer, I want to see issue labels on open issues so I can assess label health and tell AI agents what labels apply.
- As a developer, I want milestone progress (open/closed issue counts) at a glance so I know how far along a sprint is without counting manually.

---

### Journey E: "What labels are being used — is this a mess?"

**Context**: Developer suspects their repo has label sprawl — too many labels created over time that AI agents find confusing. They want to see what labels are active on open issues.

**Trigger**: GITHUB section → Labels view.

**Desired experience**:
1. Labels view shows only labels that appear on currently-open issues (not all labels ever created).
2. Developer sees: 24 labels in use. 6 are clearly duplicates (`bug` and `Bug` and `bug-fix`).
3. Developer can now clean up via `gh label delete` in a shell — Soji showed them the problem.
4. Future: configure which labels to surface per workspace in `soji.toml`.

**User stories**:
- As a developer, I want to see which labels are actively used on open issues, not every label ever created, so I can see the real scope of label clutter.
- As a developer, I want label sprawl to be visible so I can keep my repo labels clean and useful to AI agents.

---

### Journey F: "What worktrees do I have open, and what state are they in?"

**Context**: Developer has multiple worktrees for a workspace (e.g. soji has a main worktree + several session worktrees). They want to know what's open, what's dirty, what's been abandoned.

**Trigger**: GIT section → worktrees list.

**Desired experience**:
1. GIT section shows all worktrees under the workspace root — path, branch, dirty status.
2. Developer sees: 3 active worktrees, one is dirty with uncommitted changes, one has a detached HEAD.
3. They can identify stale worktrees (no recent activity) as cleanup candidates.
4. Deep-link to lazygit scoped to a worktree is a future enhancement.

**User stories**:
- As a developer, I want to see all worktrees for the current repo and their status (clean/dirty/detached) in one view.
- As a developer, I want to identify stale worktrees without running `git worktree list` manually.

---

### Journey G: "What config files does this workspace have?"

**Context**: Developer does a quick health check on the workspace — does it have a .envrc? A docker-compose? A soji.toml?

**Trigger**: CONFIG section in workspace mode.

**Desired experience**:
1. CONFIG section lists known config files with presence indicators.
2. `.envrc` present ✓, `docker-compose.yml` absent, `soji.toml` absent.
3. Can expand to view file contents for supported types (`.envrc`, `soji.toml` when it exists).
4. No parsing of secrets — just presence + optionally view the declared structure.

**User stories**:
- As a developer, I want to see at a glance which config files are present in my workspace so I know what's set up.
- As a developer, I want to view my .envrc directly from Soji without opening a separate editor.

---

## Workspace vs. Surface Distinction (design principle locked in M1.5)

Emerged from the M1.5 interview. Important to codify:

**Workspace level** — things bound to the repo/project as a whole:
- Env vars (from `.envrc`, direnv)
- Secrets (from ks + keychain)
- Git state (branch, dirty, worktrees, stash count)
- GitHub state (milestones, issues, labels)
- Config files (`.envrc`, `docker-compose.yml`, `soji.toml`, `.env.*`)

**Surface level** — things bound to a specific cmux pane/session:
- Running processes
- Listening ports
- Active worktree (which worktree this surface's CWD is in)
- Recent command history (future, via atuin)
- cmux notifications/alerts

This keeps workspace-level sections clean — no per-surface noise — while surfaces show the live session-specific state.

---

## Scope Notes

- **GitHub via `gh` CLI** — not direct API. `gh` is installed and authenticated; Soji shells out.
- **Repo detection** — inferred from `git remote get-url origin` at workspace CWD.
- **Declared = `.envrc`** — the committed `.envrc` is the source of truth for what vars a workspace expects. No soji.toml needed yet for this purpose.
- **`soji.toml` for label config** — planned but deferred to M2. M1.5 shows all labels on open issues.
- **Values never shown** — Soji shows secret names, ks paths, and load status only. Never the actual value.
- **Graceful degradation** — if `gh` isn't authed, GitHub section shows a clear "gh auth required" message. If direnv isn't installed, ENV section degrades gracefully.

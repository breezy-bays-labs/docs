---
title: Open Questions
description: 'Design and product questions that need answers before we build. Each question blocks something downstream.'
category: product
status: living
last_updated: 2026-03-04
depends_on:
  - vision
  - user-stories
---

# Open Questions

> Tracked here so they don't get lost in conversation. Each question has a priority (what it blocks) and a status.

---

## Critical — Blocks Architecture

### Q1: What is the home screen?

**Resolved**: Context-aware launch — Soji detects its context from CWD and opens the right mode automatically.

**Two modes**:

**Global Mode** — launched from outside any workspace (or `soji --global`). System-level dashboard. Tab nav: Workspaces (home) → Ports (full Port Manager) → Services (shared infrastructure). Shows all workspaces as a summary list; `Enter` to drill into a workspace or launch it.

**Workspace Mode** — launched from inside a workspace's directory tree (or `soji --workspace <name>`). Single-workspace focus. Left panel with collapsible sections (Processes, Ports, Env, Secrets, Git). Right panel with detail pane. Ports section is workspace-scoped only. `← global` in header to jump to Global Mode.

**Detection logic**:
1. Check CWD against known workspace roots
2. CWD inside a workspace → Workspace Mode for that workspace
3. CWD not inside any workspace → Global Mode
4. Override: `soji --global` or `soji --workspace <name>`

**Status**: Resolved.

---

### Q2: How does Soji discover workspaces?

**Question**: What defines a workspace from Soji's perspective?

**Options**:
- A) Whatever cmux workspaces currently exist (current behavior)
- B) Git repos discovered by scanning known directories (`~/Github`, `~/Code`, etc.)
- C) Declared in `soji.toml` — explicit registry
- D) Combination: cmux workspaces + git repo scanning, merged

**Why it matters**: Option A limits Soji to knowing only what cmux knows. Option B discovers inactive repos. Option C is the M2 "organization" goal but requires upfront work. Option D is richest but most complex.

**Lean**: Option D for the full vision, but start with A (current behavior) and add B. Option C (soji.toml) is how M2 formalizes things.

**Status**: Partially answered — cmux-first, git scanning in scope for M1.5.

---

### Q3: Runtime model — CLI, TUI, or daemon?

**Question**: How does Soji stay alive and keep data fresh?

**Options**:
- A) One-shot CLI: run it, see a snapshot, exit. Simple but no live refresh.
- B) Live TUI: stays open, refreshes on a polling interval (e.g., 5s). No background process.
- C) Daemon + TUI client: daemon runs in background, collects data continuously. TUI connects to it. Richer history, faster startup.
- D) Hybrid: live TUI by default; optional daemon for history features (M5).

**Why it matters**: A daemon changes deployment and install complexity significantly. A live TUI is simpler but can't support weekly diff (M5) without running continuously.

**Lean**: Option D. Ship a live TUI for M1. Design the data model so a daemon can be added later for M5 without breaking the TUI.

**Status**: Partially answered — live TUI for M1, daemon deferred to M5.

---

### Q4: Secret management — `ks` vs macOS Keychain as canonical source

**Resolved**: `ks` is the canonical interface. Soji sources from `ks`. macOS Keychain (`security`) is a `ks` implementation detail. If `ks` doesn't expose the interface Soji needs (e.g., list secrets by workspace), that's a `ks` enhancement — not a reason to go around it. Cross that bridge when we reach M1.5.

**Status**: Resolved.

---

## High Priority — Blocks M1 Design

### Q5: What does workspace health mean?

**Resolved**: Start with three signals for the Global Mode summary row — all detectable without `soji.toml`:

| Signal | Healthy | Degraded |
|--------|---------|---------|
| Activity | `● active` (has running processes) | `○ inactive` |
| Integrity | No orphaned/detached processes | `⚠ orphan` indicator |
| Process count | Count shown | High count is informational only |

Everything else (missing secrets, port conflicts, env var state) belongs in the detail pane, not the summary row. When `soji.toml` exists in M2+, we can add declared-vs-running comparison as a fourth signal.

**Status**: Resolved.

---

### Q6: What is the Port Manager's relationship to the Workspace View?

**Question**: Are these two separate TUI panels/tabs, or does the workspace view embed port info inline?

**Options**:
- A) Two tabs: Workspace View + Port Manager. Ports also appear inline in Workspace View for workspace-bound processes.
- B) One view: everything inline in Workspace View. Port Manager is a filter/sort mode, not a separate tab.
- C) Port Manager is a standalone `soji ports` subcommand, separate from the main TUI.

**Lean**: Option A. Workspace View is the daily driver (no noise). Port Manager is the dedicated tool for port debugging (all ports, all tiers). Ports appear in both but with different context.

**Status**: Agreed in design conversation — needs visual confirmation in mockups.

---

### Q7: How aggressive should Soji be about surfacing unaffiliated processes?

**Resolved**: YAGNI. Unaffiliated processes are visible in the Port Manager (Global Mode, Ports tab). They don't appear in the workspace list or workspace views. No badge or counter on the Global Mode home screen unless it becomes a pain point. Revisit if it turns out users miss important processes this way.

**Status**: Resolved.

---

### Q8: What is the minimum viable `soji.toml`?

**Question**: When we build M2, what's the smallest workspace definition that makes `soji up` useful?

**Candidate fields**:
```toml
[workspace]
name = "soji"
root = "~/Github/soji"
cmux_layout = "dev"  # which cmux layout to use

[services]
dev_server = { command = "cargo run", port = 8080 }

[secrets]
provider = "ks"  # or "keychain"
```

**Why it matters**: Over-engineering `soji.toml` means we delay M2 forever. Under-engineering it means it's not actually useful. We need a scope boundary.

**Status**: Needs scoping before M2 design.

---

## Medium Priority — Blocks M2+

### Q9: What does `soji down` do?

**Question**: When tearing down a workspace, what exactly gets shut down?

**Options**:
- A) Only processes Soji started (if Soji manages lifecycle). Leaves other things alone.
- B) All workspace-bound processes — aggressive, regardless of how they started.
- C) Asks the user — shows a checklist of what would be stopped.

**Lean**: Option C for safety. Soji shouldn't silently kill things it didn't start. Show the list, let the user confirm.

**Status**: Needs decision before M2.

---

### Q10: How does Soji handle worktrees?

**Question**: Worktrees live at a different path than the repo root. How does CWD matching handle them?

**Example**: Repo root is `~/Github/soji`. Worktrees are at `~/Github/soji/.claude/worktrees/feature-x/`. A process has CWD `~/Github/soji/.claude/worktrees/feature-x/src/`.

**Answer**: Any CWD that is a descendant of the workspace root (at any depth) is considered workspace-bound for that workspace. This includes all worktrees under `.claude/worktrees/`.

**Note**: Worktrees outside the repo root (e.g., parallel directory worktrees) need to be listed in `soji.toml` or discovered via `git worktree list`.

**Status**: Mostly answered — verify edge cases during M1 implementation.

---

### Q11: Clipboard and shell integration

**Question**: Should Soji be able to push commands or values to the shell? For example, if you're in Soji and you want to run the command you see displayed, can you copy it?

**Options**:
- A) Copy to clipboard via a keybinding (`y` to yank).
- B) No integration — Soji is read-only + kill, nothing else.
- C) Future: "open in shell" action that spawns a new cmux pane.

**Lean**: Option A is easy to add (crossterm + pbcopy/xclip). Option C is powerful but scope-creep for now.

**Status**: Nice-to-have, not blocking.

---

### Q12: Log streaming in the detail pane

**Question**: If a dev server is listed in the Port Manager, can the detail pane show its logs?

**Challenge**: Soji doesn't start processes, so it doesn't hold their stdout/stderr. To stream logs Soji would need to:
- Find log files the process has open (`lsof -p PID` for file descriptors)
- Tail known log locations (`.soji/logs/`, `/tmp/`, etc.)
- In M2+: if Soji starts the process itself, redirect output to a managed log file

**Status**: Deferred to M3. In M1, detail pane shows process metadata but no log stream.

---

## Resolved

| Question | Decision |
|----------|----------|
| Home screen / launch mode | Context-aware: inside workspace CWD → Workspace Mode; outside → Global Mode. Override with `--global` or `--workspace <name>`. |
| Global Mode layout | Tab nav: Workspaces (home) → Ports (full Port Manager) → Services (shared infra). Workspace list with summary rows. |
| Workspace Mode layout | Split pane: left = collapsible sections (Processes, Ports, Env, Secrets, Git); right = detail pane. `← global` in header to escape. |
| Port Manager scope | Global Mode: all ports, all tiers. Workspace Mode: workspace-bound ports only. Same data source, different filter. |
| `Enter` behavior | Always progressive disclosure. Never destructive. |
| Kill keybinding | `k` for kill. `Space` to multi-select. |
| Table navigation | `↑`/`↓` primary, `j`/`k` also supported. |
| Panel switching | `Tab`/`Shift-Tab`. |
| Workspace switching | `1`–`9` keys. |
| Mouse support | Yes — built in from M1 via crossterm `EnableMouseCapture`. |
| "Dev server" as category | Removed. Absorbed into Workspace-bound scope. |
| ProcessScope tiers | Workspace-bound / Shared infrastructure / Unaffiliated. |
| Worktree → workspace rollup | Any CWD under workspace root counts as workspace-bound. |
| Secret values | Never stored, never displayed. Names and load status only. |
| Unaffiliated processes | Port Manager only. No badge on Global home. YAGNI. |
| Secret canonical source | `ks` only. macOS Keychain is a `ks` backend detail. |

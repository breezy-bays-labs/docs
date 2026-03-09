---
title: Surface Types
description: Taxonomy of the 7 surface types that appear in a developer's cmux workspace, with detection signals and noise guidance for Soji.
category: design
milestone: M1
status: final
last_updated: 2026-03-06
---

# Surface Types

A **surface** is a cmux tab or pane — the unit Soji monitors. Not every surface maps cleanly to a single process, and not every signal a surface exposes is worth reading. This document defines the 7 surface types, how Soji detects each, and which signals are actionable vs noise.

The surface type taxonomy matters for M1.5 forward (dynamic Scout Mode body, per-surface-type detail panes). In M1, surface types are detected and stored but not yet used to drive different UI behavior. Invest in getting detection right now so M1.5 can build on it.

---

## 1. Agent / LLM Session

**Description**: A pane running an AI coding agent — Claude Code (`claude`), Cursor agents, or custom LLM runners. These are the highest-value surfaces in an agentic workflow: they consume the most memory, generate the most output, and are the most expensive to accidentally kill.

**Detection**:
- Process name matches: `claude`, `cursor`, any process with `--model` flag in command args
- `--session-id <uuid>` flag present in full command (resolved via `ps -p <pid> -o args=`)
- `--agent-id <id>` flag marks a sub-agent spawned by an orchestrating session
- High RSS memory (>200 MB) combined with a known agent process name is a strong signal
- CWD typically at or under a workspace root, but not always (Claude Code can be launched globally)

**Useful signals**:
- `--session-id` — unique identifier for correlating with JSONL session logs in `~/.claude/projects/`
- `--model claude-opus-4-5` or similar — which model is running (cost and capability signal)
- `--agent-id` — marks sub-agents; parent PID links back to the orchestrating session
- RSS memory from `ps aux` — agents accumulate context and grow over time
- CWD — which workspace owns this session

**Noise / do not use**:
- TTY field — Claude Code often detaches from TTY; absence of TTY does not mean orphaned
- Process uptime alone — a long-running Claude session is not necessarily stale; check if parent session is alive

**cmux surface type value**: `agent` (from `cmux list-pane-surfaces` output)

**Example full command**:
```
/opt/homebrew/bin/node /opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js \
  --session-id a3f8c2d1-4e7b-4a9c-b5f0-1234567890ab \
  --model claude-opus-4-5 \
  --workspace /Users/cmbays/Github/soji
```

---

## 2. Runner / Log Tail

**Description**: A pane running a long-lived background job — `npm run dev`, `cargo watch -x run`, a CI log stream (`gh run watch`), or a background build. These surfaces actively produce output but do not interact with the user.

**Detection**:
- Process names: `node`, `bun`, `cargo`, `python`, `ruby`, `go`, `deno`, `vite`, `next`, `turbo`, `nx`
- Watching patterns in command args: `watch`, `dev`, `serve`, `start`, `run dev`, `--watch`
- Often has a listening port (serves a dev server)
- CWD is highly reliable — runners always start from the project root
- Typically a direct child of a shell (bash/zsh) with no sub-processes except the compiled output

**Useful signals**:
- Listening ports from `lsof -iTCP -sTCP:LISTEN` — if a process owns `:3000`, that's its primary signal
- CWD — this is the cleanest workspace mapping signal for runners
- Full command — reveals what script is running (`package.json` script name, cargo subcommand, etc.)
- RSS memory — build watchers (webpack, cargo watch) can balloon; worth surfacing

**Noise / do not use**:
- Process name alone — `node` could be Claude Code, a dev server, or a one-off script; always resolve full command
- Port presence alone — not all runners serve ports (cargo watch for a CLI tool has no port)

**cmux surface type value**: `runner`

**Example full command**:
```
/opt/homebrew/bin/bun run dev
  (CWD: /Users/cmbays/Github/kata, port :3000)
```

---

## 3. Passive Monitor

**Description**: A pane running a system monitoring TUI — `btop`, `bottom` (`btm`), `htop`, `lazydocker`, `ctop`. These surfaces are display-only: they read system state but have no project affiliation. They should never be classified as workspace-bound.

**Detection**:
- Process name exactly matches: `btop`, `btm`, `htop`, `htop`, `glances`, `lazydocker`, `ctop`, `k9s`
- No listening ports (passive monitors never bind ports)
- CWD is arbitrary — typically the shell's CWD at launch, not a project directory

**Useful signals**:
- Process name — sufficient for identification
- Memory — `btop` and `btm` can use notable memory with large refresh rates; worth showing in global process list

**Noise / do not use**:
- CWD — a monitor launched from `~/Github/kata` is not "affiliated with kata"; it's ambient tooling
- Listening ports — these tools never listen; a port on a monitor process would be anomalous and worth investigating

**cmux surface type value**: `monitor`

**Scope classification**: Always `Unaffiliated` regardless of CWD. Add process names to a static exclusion list in `classify_scope()` so they never get mistakenly tagged as WorkspaceBound.

---

## 4. Directory Navigator

**Description**: A pane running a file system navigator — `yazi`, `ranger`, `lf`, `nnn`, `broot`. The user is browsing the filesystem interactively. CWD changes constantly as the user navigates — it is not a stable workspace signal.

**Detection**:
- Process name matches: `yazi`, `ranger`, `lf`, `nnn`, `broot`
- Interactive TUI (no subprocesses, no ports)
- CWD is highly dynamic — can be at the project root, deep inside a file tree, or in `/tmp`

**Useful signals**:
- Process name — identification only
- Nothing else is stable enough to use

**Noise / do not use**:
- CWD — a yazi session navigating through `~/Github/kata/src/domain/` does not indicate workspace membership; the user may have started from a completely different workspace and is just browsing
- Memory — navigators are lightweight and memory usage is not meaningful

**cmux surface type value**: `navigator`

**Scope classification**: Always `Unaffiliated`. Even if CWD happens to be under a workspace root, the classification is meaningless for a navigator because it changes with every keystroke. Do not use navigator CWD for workspace inference.

**Design note**: In M1.5, when Soji can detect which surface is focused in adjacent panes, the Scout Mode body could show the navigator's current path as informational context ("browsing: kata/src/domain"). But this is display-only — never used for scope or workspace attribution.

---

## 5. Active Work Surface

**Description**: A pane with an editor or IDE open — Neovim (`nvim`), VS Code (`code`), Zed (`zed`), Helix (`hx`), Emacs. This is where the developer writes code. The most reliable surface for workspace attribution via CWD.

**Detection**:
- Process name matches: `nvim`, `vim`, `code`, `zed`, `hx`, `emacs`, `nano`
- CWD is stable — editors open to a project root and stay there
- Language servers (LSP) are child processes: `rust-analyzer`, `typescript-language-server`, `pyright`, `lua-ls`
- Language server ports or socket files may be present (editor-to-LSP communication)

**Useful signals**:
- CWD — the most reliable workspace attribution signal available in M1
- Full command — may include the file or directory being edited (`nvim src/main.rs`), useful for detail pane
- Child processes (language servers) — indicate active development; worth surfacing in workspace detail
- Memory — Neovim with many plugins + active LSP can use significant memory

**Noise / do not use**:
- Language server ports — these are inter-process sockets, not dev server ports; do not list them in Port Manager
- TTY — editors always have a TTY; its presence is expected, not informative

**cmux surface type value**: `editor`

**Example**: `nvim src/tui/app.rs` in CWD `/Users/cmbays/Github/soji` → WorkspaceBound to `soji` workspace.

---

## 6. Quick Terminal

**Description**: An ad-hoc terminal pane used for one-off commands — running `git status`, a quick `curl`, testing a build output, checking logs. Not dedicated to any single purpose. This is the most common surface type and the least predictable.

**Detection**:
- Shell process: `zsh`, `bash`, `fish` with no significant child processes
- Or a short-lived child process (`git`, `curl`, `cargo build`) that changes frequently
- No stable long-running process

**Useful signals**:
- Shell CWD at the moment of snapshot — gives a hint but not a guarantee of workspace affiliation
- If a child process is running at snapshot time, its full command is useful

**Noise / do not use**:
- CWD for workspace attribution — a developer in a quick terminal may have `cd`'d to any directory; the shell's current CWD is a weak signal. Do not use quick terminal CWD as a primary workspace attribution signal. If the shell is idle (no child process), skip attribution entirely.
- Any transient child process — commands that complete in under a second will not be visible at snapshot time

**cmux surface type value**: `terminal`

**Design guidance**: When a quick terminal has no child process running, Soji records it as `ProcessType::Shell` with `scope = Unaffiliated` unless the CWD is the same as the workspace root (not just under it — exactly the root). This conservative stance avoids false positives.

---

## 7. Browser Tab

**Description**: A browser tab managed by cmux's sidebar integration — Arc, Chrome, Safari, Firefox. Soji can detect browser tabs from the cmux sidebar state. There are no process-level signals available for individual tabs — the browser is a single OS process, and tab isolation happens inside it.

**Detection**:
- cmux `sidebar-state` output lists browser tab surfaces with a URL and title
- Browser process (`arc`, `Google Chrome`, `Safari`) is visible via `ps aux`, but individual tab attribution requires the sidebar

**Useful signals**:
- URL from cmux sidebar — can indicate which deployment, localhost port, or documentation page is open
- Title from cmux sidebar — provides context ("Soji TUI — GitHub PR #41")
- localhost URLs (`:3000`, `:8080`) connect browser tabs back to workspace-bound processes

**Noise / do not use**:
- Browser process memory — a single Chrome process represents all tabs across all workspaces; its RSS is not meaningful for a single workspace
- Browser PID — cannot be used to attribute a single tab to a workspace

**cmux surface type value**: `browser`

**Design note**: In M1, browser tabs appear in the Workspace Mode left panel as surface rows but have no expandable process children. The detail pane shows the URL and title. Kill operations are not available for browser tabs. In M2+, when `soji.toml` defines expected tabs for a workspace, Soji can flag missing or unexpected browser surfaces.

---

## Signal Quality Summary

| Surface Type | CWD reliable? | Port signal? | Memory signal? | Kill-safe? |
|---|---|---|---|---|
| Agent / LLM | Yes (mostly) | Rarely | Yes (primary) | Warn — high value session |
| Runner / log tail | Yes (primary) | Yes (often) | Yes | Safe |
| Passive monitor | No | Never | Minor | Safe (but pointless) |
| Directory navigator | No | Never | No | Safe (but pointless) |
| Active work surface | Yes (primary) | LSP only | Moderate | Warn — unsaved edits |
| Quick terminal | Weak | Transient | No | Safe |
| Browser tab | N/A | Via URL | Not useful | Not available |

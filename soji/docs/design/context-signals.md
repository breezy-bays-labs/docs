---
title: Context Signals
description: Inventory of all signals Soji reads per surface type — what each signal is, which surface types expose it, how Soji reads it, and which milestone introduces it.
category: design
milestone: M1
status: final
last_updated: 2026-03-06
---

# Context Signals

A **context signal** is any piece of information Soji reads from the system to build its workspace picture. Signals come from system tools (`ps`, `lsof`, `git`), from cmux, from the environment (`direnv`, `ks`), and from Docker. This document inventories every signal Soji reads, explains its source and reliability, and maps it to the surface types that expose it.

Not all signals are equally reliable. Some (listening ports, full command) are stable and authoritative. Others (CWD for a directory navigator, git branch for a detached process) are weak or context-dependent. Signal quality notes are included for each.

---

## Signal Inventory

---

### CWD (Current Working Directory)

**What it is**: The directory a process has as its current working directory at the moment of the snapshot.

**Why it matters**: CWD is the primary signal for `classify_scope()`. If a process's CWD is at or under a known workspace root, it is classified `WorkspaceBound`. This works because most dev tools (editors, runners, build watchers) are launched from the project root and maintain that CWD for their lifetime.

**How Soji reads it**: `lsof -d cwd -Fn -p <pid>` — returns the CWD as a file path for the given PID. Alternatively, `readlink /proc/<pid>/cwd` on Linux, but macOS uses the lsof approach. In practice, `lsof -d cwd` is called in batch for all tracked PIDs to minimize process spawning.

**Which surface types expose reliable CWD**:
- Runner / log tail — highly reliable; runners start at project root
- Active work surface (editors) — reliable; editors anchor to the opened directory
- Agent / LLM session — mostly reliable; Claude Code starts from the workspace it's given
- Quick terminal — unreliable; the shell changes CWD frequently
- Directory navigator — unreliable; CWD changes with every navigation keystroke
- Passive monitor — irrelevant; monitors are not project-affiliated

**Signal quality**: High for editors and runners. Low for navigators and quick terminals. The classifier should not use navigator or shell CWD for WorkspaceBound classification.

**Worktree handling**: A workspace root of `/Users/cmbays/Github/kata` includes all worktrees at any path under it. The classifier uses a prefix match: `process.cwd.starts_with(workspace.root)`. This means `/Users/cmbays/Github/kata/.claude/worktrees/parallel-task-abc/src/main.rs` correctly maps back to the `kata` workspace.

**Milestone**: M1.

---

### Git Branch

**What it is**: The name of the git branch currently checked out in a workspace directory.

**Why it matters**: Provides workspace context in Scout Mode header and Workspace Mode detail pane. Tells the developer at a glance which feature branch is active, without them having to switch focus to a terminal and run `git branch`.

**How Soji reads it**:
1. Preferred: cmux `sidebar-state` command, which includes the current branch for each surface. One call, no subprocess per workspace.
2. Fallback: `git -C {workspace_cwd} branch --show-current` — runs for each workspace whose branch is not already known from cmux. Spawns one git process per workspace.

The sidebar-state approach is preferred because it batches workspace information. The git fallback is used when cmux is not available or when a workspace has no active cmux surface.

**Which surface types expose it**: Any surface with a stable CWD pointing to a git repository. Editors and runners are the primary sources. The branch is a property of the *workspace*, not the surface — Soji reads it once per workspace, not per surface.

**Signal quality**: High. Git branch is stable and accurate. The only edge case is a detached HEAD state (`HEAD detached at a3f8c2d`) — in this case, Soji shows `detached` in the Scout header.

**Milestone**: M1.

---

### Listening Ports

**What it is**: TCP ports that a process has open in LISTEN state — meaning it is actively accepting connections.

**Why it matters**: The primary signal for dev servers, API servers, and services. Port 3000 with a node process means the Next.js dev server is running. Port 11434 with an `ollama` process means Ollama is serving. Ports drive the Port Manager view and workspace-bound port attribution.

**How Soji reads it**: `lsof -iTCP -sTCP:LISTEN -Fpn` — returns PID and port number for every listening TCP socket. This is a single invocation that returns all listening ports across all processes. Soji parses the output and joins it with the process list by PID.

**Which surface types expose it**:
- Runner / log tail — primary source; dev servers always listen
- Agent / LLM session — occasionally; some agent runtimes expose a status API
- Active work surface — language servers use sockets (typically Unix domain, not TCP) — not captured by lsof TCP filter
- Passive monitor — never
- Directory navigator — never
- Quick terminal — transient processes may listen briefly (e.g., `python -m http.server 8080`)
- Browser tab — shows as a URL, not a port; no direct OS-level port signal

**Signal quality**: High. `lsof -sTCP:LISTEN` is authoritative. Port → PID mapping is reliable.

**Port Manager tiers**: Ports are classified by their owning process's `ProcessScope`:
- `WorkspaceBound` — port exposed under workspace-scoped processes in Workspace Mode and Port Manager
- `SharedInfra` — port shown in Services tab (Ollama :11434, Docker daemon :2375, OrbStack :443)
- `Unaffiliated` — shown in Port Manager only, with a `?` indicator

**Milestone**: M1.

---

### Memory Usage (RSS)

**What it is**: Resident Set Size — the amount of physical RAM the process is currently using, in kilobytes (from `ps aux`, converted to MB for display).

**Why it matters**: The primary signal for identifying memory-hungry processes. In an agentic workflow, Claude Code sessions accumulate context and grow over time. A session using 1.2 GB of memory is a strong signal that it has been running for a long time and may be worth reviewing.

**How Soji reads it**: `ps aux` output, `RSS` column (column 6, in KB). Soji runs `ps aux` in batch and parses the full output. RSS is converted to MB by dividing by 1024.

**Which surface types expose it**: All process-level surface types (Agent, Runner, Monitor, Navigator, Editor, Quick terminal). Browser tabs share a single browser process RSS — not meaningful per-tab.

**Signal quality**: High. RSS from `ps` is accurate at the moment of the snapshot. It does not reflect peak memory or memory growth over time — those require repeated sampling (M5+).

**Memory bar display**: In Scout Mode, memory bars show each process's RSS as a proportion of the highest-RSS process in the current list (not absolute thresholds). This makes bars meaningful at any memory scale.

**Milestone**: M1.

---

### Full Command

**What it is**: The complete command line that spawned a process, including all flags and arguments. This is the resolved, un-truncated command — not the 15-character truncated name that `ps aux` shows in the `COMM` column.

**Why it matters**: Process names alone are ambiguous. `node` could be Claude Code, a Next.js dev server, or a one-off script. `cargo` could be `cargo watch`, `cargo test`, or `cargo build`. Full command resolution turns ambiguous names into actionable information. It is also the data shown in the Port Manager detail pane and Workspace Mode process detail.

**How Soji reads it**: Batch `ps -p <pids joined by comma> -o pid=,args=` — a single invocation that returns PID and full args for all tracked PIDs. This is the command resolution fix in M1 (Part A10 of the shaping plan). The output is parsed by splitting on whitespace after the PID field.

**Which surface types expose it**: All process-level surface types. Browser tabs have no per-tab command.

**Signal quality**: High. `ps -o args=` returns the full argv array joined with spaces — no truncation. Edge case: processes that have called `prctl(PR_SET_NAME, ...)` to rename themselves may show a custom name rather than the original command. Rare in practice.

**Example resolution**:
- `ps aux` shows: `node` (truncated at 15 chars)
- `ps -p 12345 -o args=` shows: `/opt/homebrew/bin/node /opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js --session-id a3f8c2d1 --model claude-opus-4-5`

**Milestone**: M1 (fixing the pre-M1 truncation bug).

---

### Claude Session ID

**What it is**: The UUID passed via `--session-id` flag to the Claude Code CLI. Unique per session, persisted across restarts of the same session.

**Why it matters**: Allows Soji to correlate a running process with its session history in `~/.claude/projects/<workspace>/`. The session ID links the live process to its JSONL log file, which contains the session name, model, cost, and conversation history.

**How Soji reads it**: Parsed from the full command (see above). The full command contains `--session-id a3f8c2d1-4e7b-4a9c-b5f0-1234567890ab`. Soji extracts the flag value with a simple argument parser.

**Which surface types expose it**: Agent / LLM session only. No other surface type uses `--session-id`.

**Signal quality**: High. The session ID is set by Claude Code at launch and never changes during the session's lifetime.

**Milestone**: M1.

---

### Claude Model

**What it is**: The model identifier passed via `--model` flag to the Claude Code CLI. For example: `claude-opus-4-5`, `claude-sonnet-4-6`, `claude-haiku-3`.

**Why it matters**: Model identity is a resource signal — Opus is more expensive than Sonnet, which is more expensive than Haiku. Showing the model in the process detail pane lets the developer see at a glance which tier of model is running in which session. Useful for multi-agent setups where different agents may be assigned different models for cost optimization.

**How Soji reads it**: Parsed from the full command, same as session ID. Extracts the value of `--model` flag.

**Which surface types expose it**: Agent / LLM session only.

**Signal quality**: High.

**Milestone**: M1.

---

### cmux Surface Type

**What it is**: The surface type classification as reported by cmux's `list-pane-surfaces` command. cmux returns a structured output with surface entries that include a `type` field (values: `agent`, `runner`, `monitor`, `navigator`, `editor`, `terminal`, `browser`).

**Why it matters**: cmux's surface type is the authoritative classification for the 7 surface types defined in `surface-types.md`. Soji uses it to populate `Surface.surface_type` and to drive M1.5's dynamic Scout Mode body.

**How Soji reads it**: `cmux list-pane-surfaces` returns JSON or structured text. Soji calls this in `src/sources/cmux.rs` and parses the output into `Surface` domain structs. The surface type field maps directly to Soji's `SurfaceType` enum.

**Which surface types expose it**: By definition, all surface types are reported this way. cmux knows the type of each pane it manages.

**Signal quality**: High when cmux is installed and the workspace is managed by cmux. Falls back to `Unknown` when cmux is not available — Soji degrades gracefully.

**Milestone**: M1.

---

### Environment Variables (Names and Source)

**What it is**: The names of environment variables loaded in a workspace, along with their source (direnv, `.env` file, `ks`, shell export). Values are never read, stored, or displayed.

**Why it matters**: A missing `DATABASE_URL` causes a confusing crash 10 minutes into a coding session. Showing which env vars are loaded and where they come from (direnv loaded ✓, `.env.local` loaded ✓) gives the developer confidence that the environment is correctly configured before they start.

**How Soji reads it**:
- `direnv export json` — returns the env var names exported by `.envrc` for the workspace CWD. Requires direnv to be installed.
- Parse `.env`, `.env.local`, `.env.development` files for variable names only (no values read).
- `ks ls` — lists secret names from the default keychain or workspace-scoped keychain.

Soji applies a heuristic to flag variables as sensitive: names containing `TOKEN`, `KEY`, `SECRET`, `PASSWORD`, `CREDENTIAL` are flagged as sensitive regardless of their source. Flagged variables are shown with a lock icon in the detail pane.

**Which surface types expose it**: Environment variables are a workspace-level signal, not surface-level. All surfaces in a workspace share the same env context (assuming direnv is active for the workspace root).

**Signal quality**: Medium. Depends on direnv being installed and `.envrc` being present. Falls back to empty list if direnv is not available.

**Milestone**: M1.5 (deferred from M1).

---

### Secrets (Names and Load Status)

**What it is**: The names of secrets managed by `ks` (the project's macOS Keychain wrapper) and their load status — whether they are currently accessible in the environment.

**Why it matters**: Secrets not loaded = broken builds. `GITHUB_TOKEN` not present means `gh` CLI calls fail silently. Showing which secrets are loaded by name (never by value) gives the developer a quick verification layer.

**How Soji reads it**: `ks ls` returns a list of secret names from the current keychain. `ks ls -k <workspace>` scopes to a workspace-named keychain. Soji calls this and stores `SecretRef { name, provider: Ks, loaded: bool }`.

**Which surface types expose it**: Workspace-level signal, same as env vars.

**Signal quality**: High when `ks` is installed. Falls back to empty when not available.

**Milestone**: M1.5 (deferred from M1).

---

### Docker Containers

**What it is**: Running Docker containers — their name, image, ports, memory, and status.

**Why it matters**: Docker containers are services that support a workspace's development. A Postgres container serving a dev database, a Redis container for caching, a mock third-party API container — these are workspace-bound infrastructure. Knowing they are running (or stopped) is part of workspace health.

**How Soji reads it**: Two calls:
1. `docker ps --format json` — returns container list with name, image, ports, status
2. `docker stats --no-stream --format json` — returns memory, CPU per container

Both calls require Docker daemon to be running. If Docker is not available, the Docker section is absent from the workspace view (graceful degradation per R15).

**Which surface types expose it**: Docker containers appear as their own item type in the Workspace Mode left panel — separate from process-level surfaces but under the same workspace. They are not associated with a specific cmux surface.

**Signal quality**: High when Docker is running. The `docker ps` and `docker stats` output is authoritative.

**Milestone**: M1.

---

### cmux Surface Focus (FocusGained / FocusLost)

**What it is**: A terminal event emitted by the terminal emulator when the current pane gains or loses focus. Fired as `Event::FocusGained` and `Event::FocusLost` by crossterm.

**Why it matters**: This is the mechanism that drives Scout Mode transitions. Without this signal, Soji cannot know when the user has left the Soji pane or returned to it.

**How Soji reads it**: Crossterm `EnableFocusChange` escape sequence emitted during terminal setup. crossterm then includes `Event::FocusGained` and `Event::FocusLost` in its event stream, which Soji's event loop handles in `src/tui/events.rs`.

**Which surface types expose it**: The Soji surface itself (the pane running the Soji TUI). This signal is about Soji's own focus state, not the focus state of adjacent surfaces.

**Signal quality**: High on supported terminals (Ghostty, Zellij, iTerm2, Alacritty, WezTerm). Silent failure on unsupported terminals (macOS Terminal.app ignores the escape sequence). Soji degrades gracefully — no crash, just no Scout Mode transitions.

**Milestone**: M1.

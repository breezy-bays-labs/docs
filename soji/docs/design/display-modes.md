---
title: Display Modes
description: Precise definitions of Soji's three display modes — Global, Workspace, and Scout — including layouts, navigation, keybindings, and transition rules.
category: design
milestone: M1
status: final
last_updated: 2026-03-06
---

# Display Modes

Soji has three display modes: **Global Mode**, **Workspace Mode**, and **Scout Mode**. Each has a distinct layout, navigation model, and set of interactions. Transitions between modes are deterministic — each transition has a specific trigger and a specific reverse.

This document is the authoritative reference for mode behavior. It governs keybinding assignments, layout structure, and transition logic. The TUI implementation must match this spec exactly.

---

## Global Mode

### When Active

Global Mode is active when:
- `soji` is launched from a directory that is not under any known workspace root
- `soji --global` flag is passed explicitly
- The `g` key is pressed while in Workspace Mode

It represents a **system-level perspective**: all workspaces, all ports, all shared infrastructure. No single workspace's context dominates.

### Layout

```
┌─────────────────────────────────────────────────────┐
│  ⬡ soji  [Workspaces]  Ports  Services     ? q      │  ← Tab bar
├─────────────────────────────────────────────────────┤
│                              │                       │
│  Workspace list              │  Detail pane          │
│  (left panel)                │  (right panel)        │
│                              │  Opens on Enter       │
│  ● kata        4p :3000      │  Closes on Esc        │
│  ● soji        2p :8080      │                       │
│  ⚠ spp         1p orphan     │                       │
│                              │                       │
└─────────────────────────────────────────────────────┘
│  ↑↓ navigate  Tab panels  Enter detail  Space sel  k kill  ? help  q quit  │
```

The tab bar at the top shows the three tabs with the active tab highlighted. The body is a split-pane: left list, right detail. The status bar at the bottom shows active keybindings.

### Tab Structure

| Tab | Key | Content |
|---|---|---|
| Workspaces | `1` | All known workspaces as summary rows with health signals |
| Ports | `2` | Full Port Manager — all listening ports by ProcessScope tier |
| Services | `3` | SharedInfra processes: OrbStack, Ollama, Docker daemon, local databases |

**Tab switching**: `Tab` cycles forward through tabs (Workspaces → Ports → Services → Workspaces). `Shift-Tab` cycles backward. `1`, `2`, `3` jump directly to the corresponding tab. In the Ports and Services tabs, `4`–`9` are unused (they serve workspace jumping, but in Global Mode those keys jump to workspace index in the Workspaces list only when that tab is active).

### Workspaces Tab

Each workspace is a row in the left panel with:
- Health indicator: `●` (green = healthy), `●` (yellow = has Unaffiliated/detached), `⚠` (red = orphan detected or stale Claude session)
- Workspace name
- Process count (`4p`)
- Primary port (`:3000` — the lowest numbered WorkspaceBound listening port)
- Orphan indicator if present

**Enter on a workspace row**: Opens the workspace detail pane on the right. Shows all processes, ports, memory, and git state for that workspace. Does NOT switch to Workspace Mode — the detail pane is read-only summary within Global Mode. To enter interactive Workspace Mode for a workspace, the developer launches a new Soji instance from within that workspace's directory, or uses `soji --workspace <name>`.

This distinction is intentional: Enter in Global Mode always means "tell me more about this workspace." It is not a mode switch. Switching modes is an explicit action.

### Ports Tab (Port Manager)

All listening ports across the system, grouped by ProcessScope tier:

```
WorkspaceBound ─────────────────────────────────
  :3000   node      kata    /Github/kata   18h
  :8080   bun       soji    /Github/soji    2m

SharedInfra ─────────────────────────────────────
  :11434  ollama    Ollama   (inference server)
  :2375   docker    Docker   (daemon socket)

Unaffiliated ────────────────────────────────────
  :9229   node      ?       (inspector protocol)
```

Each row: port, process name, workspace (or service name), CWD or description, age.

**Enter on a port row**: Opens detail pane with full command, PID, memory, CWD, uptime. `k` to kill. SharedInfra ports require extra confirmation.

### Services Tab

SharedInfra processes shown as a health dashboard:

```
  OrbStack     ✓ running    :443 :80    PID 1234   124 MB
  Docker       ✓ running    :2375       PID 5678    88 MB
  Ollama       ✓ running    :11434      PID 9012   312 MB
  PostgreSQL   ✗ not found  :5432       (not detected)
```

Services are detected by process name + port heuristic. Presence in this list does not mean they are workspace-bound — they are shared infrastructure. Kill from Services tab requires extra confirmation (k+k+k, see kill flow in Workspace Mode docs).

### Navigation

| Key | Action |
|---|---|
| `↑` / `↓` | Navigate rows in the active left panel |
| `j` / `k` | Same as `↑` / `↓` (vim aliases) |
| `Tab` | Cycle through tabs (Workspaces → Ports → Services) |
| `Shift-Tab` | Cycle tabs backward |
| `1` | Jump to Workspaces tab |
| `2` | Jump to Ports tab |
| `3` | Jump to Services tab |
| `4`–`9` | In Workspaces tab: jump cursor to workspace at index N-3 |
| `Enter` | Open detail pane for current row |
| `Space` | Toggle process selection (when cursor is on a process row in Port/Services tab) |
| `k` | Kill selected processes (or current row if no selection) |
| `Esc` | Close detail pane / clear selection |
| `g` | No-op (already in Global Mode) |
| `?` | Open help overlay |
| `q` / `Ctrl-C` | Quit Soji |

### Mode Transitions from Global

| Trigger | Destination |
|---|---|
| `FocusLost` event | Scout Mode (saves `prev_mode = Global`) |
| `q` / `Ctrl-C` | Exit |
| There is no key to enter Workspace Mode from Global — Workspace Mode is entered by launching Soji from inside a workspace | — |

---

## Workspace Mode

### When Active

Workspace Mode is active when:
- `soji` is launched from a directory that is under a known workspace root (CWD prefix match)
- `soji --workspace <name>` flag is passed explicitly

It represents a **workspace-level perspective**: one workspace's processes, ports, environment, and git state. Designed to live in a dedicated cmux pane while the developer works.

### Layout

```
┌─────────────────────────────────────────────────────┐
│  ⬡ soji  ← global  git:main  4p :3000     ? q      │  ← Header
├───────────────────────┬─────────────────────────────┤
│  Left panel           │  Right panel (detail)        │
│                       │                              │
│  ▼ Processes          │  PID: 12345                  │
│    claude    487 MB ● │  Command: claude --session.. │
│    node      104 MB   │  CWD: /Github/soji           │
│  ▶ Ports              │  Memory: 487 MB              │
│  ▶ Env                │  Scope: WorkspaceBound       │
│  ▶ Git                │  Port: none                  │
│                       │  Model: claude-opus-4-5      │
│                       │  Session: a3f8c2d1-...       │
└───────────────────────┴─────────────────────────────┘
│  ↑↓ navigate  Tab panel  Enter expand  Space sel  k kill  g global  ? help  │
```

The header shows workspace name, `← global` hint (press `g`), git branch, process count, and primary port. The left panel is a collapsible section list. The right panel is the detail pane — empty until `Enter` is pressed.

### Left Panel Sections

Sections collapse and expand with `Enter` when the cursor is on a section header. A `▶` prefix means collapsed; `▼` means expanded.

| Section | Content |
|---|---|
| **Processes** | All processes classified as WorkspaceBound to this workspace. Subsections by surface type in M1.5. In M1: flat list sorted by memory descending. |
| **Ports** | WorkspaceBound listening ports only. No SharedInfra, no Unaffiliated. Shows port number, process name, age. |
| **Env** | (M1.5) Environment variable names and source (direnv, .env, ks, shell). Not available in M1 — section is present but shows `(coming in M1.5)`. |
| **Secrets** | (M1.5) Secret names from `ks ls`. Not available in M1. |
| **Git** | Current branch, dirty status (modified files count), worktrees if any. Uses cached data from cmux sidebar-state or a fresh `git status --short` call. |

**Process row format**:
```
  claude        487 MB  ●
  node          104 MB
  rust-analyzer  88 MB
```
- Process name (left-aligned, 14 chars max)
- Memory in MB (right-aligned)
- `●` green dot if selected (multi-select state)
- `[k?]` kill-confirm indicator appears after first `k` press

**Docker containers**: If Docker is running, a **Docker** section appears between Ports and Env. Shows containers whose network connects to the workspace (detected by container label or workspace CWD convention). In M1, all running containers appear here — workspace attribution for containers is tightened in M2 with `soji.toml`.

### Right Panel (Detail Pane)

Opens when `Enter` is pressed on any left panel row. The detail pane stays open until `Esc` is pressed.

**Process detail**:
- PID, process name, full command (resolved — no truncation)
- CWD
- Memory (RSS in MB)
- ProcessScope: WorkspaceBound / SharedInfra / Unaffiliated
- Listening ports (if any)
- Uptime
- Claude-specific: session ID, model (if process is a Claude agent)

**Port detail**:
- Port number, protocol (TCP)
- Owning process: PID, full command, CWD
- ProcessScope tier
- Age (how long port has been listening)
- Kill affordance: `k` to kill the owning process

**Section header detail** (e.g., Git section):
- Git: branch name, last commit hash and message, dirty file count, worktrees list

### Navigation

| Key | Action |
|---|---|
| `↑` / `↓` | Navigate rows in the focused panel |
| `j` / `k` | Same as `↑` / `↓` (vim aliases) — EXCEPT: `k` alone on a Process row activates kill, not navigation |
| `Tab` | Switch focus from left panel to right panel (and back) |
| `Enter` | On section header: expand/collapse section. On process/port row: open detail pane. Never a destructive action. |
| `Space` | Toggle current row's PID into/out of `app.selected` (multi-select) |
| `k` | Kill: starts kill confirmation flow (see kill flow below) |
| `Esc` | Close detail pane / clear selection / cancel kill confirmation |
| `g` | Switch to Global Mode |
| `1`–`9` | Jump cursor to process at index N in the process list |
| `?` | Open help overlay |
| `q` / `Ctrl-C` | Quit Soji |

### The k Key — Kill Flow Detail

The `k` key in Workspace Mode is context-sensitive. Its behavior depends on what the cursor is on and what state `kill_confirm` and `selected` are in.

**Normal process (WorkspaceBound or Unaffiliated)**:
1. First `k`: `kill_confirm = Some(pid)`. Row shows `[kill?]` indicator. No action yet.
2. Second `k`: `KillConfirmed(pid)`. Process receives SIGTERM. If still alive after 3 seconds, SIGKILL.
3. `Esc` at any point: `kill_confirm = None`. Kill cancelled.

**SharedInfra process** (OrbStack, Docker daemon, Ollama):
1. First `k`: `infra_kill_confirm = Some(pid)`. Status bar shows warning: `SharedInfra — press k twice more to confirm`.
2. Second `k`: `kill_confirm = Some(pid)`. Status bar: `Are you sure? Press k again to confirm.`
3. Third `k`: `KillConfirmed(pid)`. SIGTERM.
4. `Esc` at any point: all confirm state cleared.

**Multi-select kill** (when `selected` is non-empty):
1. `Space` on each process row toggles it into `selected` (HashSet<u32>).
2. `k` with non-empty `selected`: prompts "Kill N processes? Press k to confirm."
3. Second `k`: `KillBatch(selected.clone())`. All selected processes receive SIGTERM.
4. SharedInfra processes in the selection are called out separately with extra confirmation.

**Critical rule**: `Enter` NEVER triggers a kill action at any point. `Enter` always means "show me more." This rule is enforced in `src/tui/events.rs` and verified in behavioral tests (`src/tests/tui_behavior.rs`).

### Mode Transitions from Workspace

| Trigger | Destination |
|---|---|
| `g` key | Global Mode |
| `FocusLost` event | Scout Mode (saves `prev_mode = Workspace`) |
| `q` / `Ctrl-C` | Exit |

---

## Scout Mode

### When Active

Scout Mode is active when:
- A `FocusLost` event fires while in Global or Workspace Mode (the Soji pane loses focus)

It is not directly launchable — it is always a transition from an existing mode. See `scout-mode.md` for the full Scout Mode specification.

### Layout

```
⬡ soji  ● 4p  :3000 :8080  main
  claude        ████████░░░░  487 MB
  rust-analyzer ████░░░░░░░░  212 MB
  node          ██░░░░░░░░░░  104 MB
  zed           █░░░░░░░░░░░   88 MB
[focus to interact]  q quit
```

Single column. No split pane. No cursor highlight. No keybindings except `q`.

### Navigation

Scout Mode has no navigation. It is read-only. The only interactive key is `q` to quit. All other keys are ignored.

The developer interacts by switching focus back to the Soji pane (FocusGained), which restores the previous full-screen mode.

### Mode Transitions from Scout

| Trigger | Destination |
|---|---|
| `FocusGained` event | Previous mode (`prev_mode` — Global or Workspace) |
| `q` / `Ctrl-C` | Exit |

---

## Mode Transition Summary

```
            soji launched outside workspace CWD
            or --global flag
                    │
                    ▼
              ┌──────────────┐
    ◄──────── │  Global Mode │ ──────────────────────────────────────────┐
    FocusGain └──────────────┘  FocusLost                                │
                    │                │                                   │
              (no direct)            ▼                                   │
              (key to WS)      ┌──────────────┐                         │
                               │  Scout Mode  │                         │
              FocusGain ──────►└──────────────┘◄────────── FocusLost   │
                                                                        │
            soji launched inside workspace CWD                          │
            or --workspace flag                                          │
                    │                                                   │
                    ▼                              g key                │
              ┌──────────────┐ ────────────────────────────────────────►┘
              │ Workspace Md │
              └──────────────┘
```

The critical constraint: there is no direct key to enter Workspace Mode from Global Mode. The modes are parallel launch contexts, not a hierarchy. A developer in Global Mode who wants Workspace Mode for a specific workspace should launch `soji --workspace <name>` in a new pane, or navigate into that workspace's directory and launch `soji` fresh.

This constraint keeps the mode model simple. Global Mode is for cross-workspace overview. Workspace Mode is for deep single-workspace work. Scout Mode is for ambient monitoring. Each has a clear purpose, and the transition rules enforce the purpose boundaries.

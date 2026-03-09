---
title: Scout vs Inline Mode
description: Decision framework for choosing between Scout Mode (dedicated pane TUI) and Inline Mode (prompt-integrated display). Includes workflow targeting, capability boundaries, and implementation timeline.
category: design
milestone: M1 (Scout) / M1.5 (Inline)
status: final
last_updated: 2026-03-06
---

# Scout vs Inline Mode

Soji has two ways to surface ambient workspace information to the developer: **Scout Mode** and **Inline Mode**. These are not competing features — they target different workflows and live in different parts of the terminal environment. This document defines each clearly and explains when to use which.

---

## Scout Mode

**Status**: M1 (shipping)

Scout Mode is the compact ambient layout that Soji renders when its dedicated cmux pane loses focus. It is part of the full-screen TUI — the same ratatui application, the same `App` state, but a different layout branch.

**How it works**: The user dedicates one cmux pane to Soji. Soji runs in that pane continuously. When the user switches focus to another pane (editor, agent, runner), crossterm `FocusLost` fires and Soji renders the compact Scout layout instead of the full interactive TUI. When focus returns, `FocusGained` fires and the full TUI reappears.

**What Scout Mode shows**:
- Workspace header: name, health dot, process count, active ports, git branch
- Top processes by memory with visual bars
- No cursor, no interactive affordances (except `q` to quit)

**What Scout Mode supports**:
- Interactive kill (`k`) — only available in full interactive TUI, not in Scout
- Multi-select (`Space`) — only in full interactive TUI
- Detail pane (`Enter`) — only in full interactive TUI
- Navigation (`↑`/`↓`) — not available; Scout is read-only

**Required setup**: A dedicated pane in the cmux layout. Scout Mode does not make sense in an ad-hoc terminal — it requires the persistent pane structure that cmux provides.

**Best for**:
- Workspace-level observability alongside active development
- Kill operations and port management (via switching back to full TUI)
- Developers who work in a fixed cmux layout with a persistent Soji pane

---

## Inline Mode

**Status**: M1.5 (deferred — not in M1)

Inline Mode renders Soji output directly in the terminal output stream, below the current shell prompt. It does not take over the full screen. It is designed to complement Powerlevel10k by providing richer workspace context at the prompt level.

**How it works**: `soji inline` (or a shell hook) triggers a non-interactive render that writes to stdout, then exits. The output appears between commands, like a status report after each shell command, or on demand.

**What Inline Mode shows**:
- Workspace name and health indicator
- Active process count and top ports
- Git branch and dirty status
- Secrets loaded / not loaded (names only)
- No interactive UI — strictly read-only output

**What Inline Mode does NOT support**:
- Interactive kill — no cursor, no keybindings, no TUI state
- Multi-select
- Detail pane
- Background refresh — each invocation is a fresh snapshot; there is no persistent app state

**Required setup**: None beyond installing Soji. Can be invoked from a shell hook (e.g., `precmd` in Zsh) or manually. Does not require cmux.

**Best for**:
- Developers who work in ad-hoc terminals without a fixed cmux layout
- Prompt-level workspace context alongside Powerlevel10k
- Read-only status checks in situations where opening a full TUI is too heavy

---

## The Core Distinction

| Dimension | Scout Mode | Inline Mode |
|---|---|---|
| Requires dedicated pane | Yes | No |
| Requires cmux | Yes (for transitions) | No |
| Interactive (kill, select) | Yes (in full TUI) | No |
| Persistent / live-updating | Yes (background refresh) | No (point-in-time) |
| Renders in TUI | Yes | No (stdout only) |
| Activates automatically | Yes (FocusLost event) | No (manual / hook) |
| Milestone | M1 | M1.5 |

The key insight: Scout Mode is a *state* within the persistent TUI application. Inline Mode is a *separate render path* — a CLI output mode that produces terminal-friendly text and exits.

---

## What Belongs Where

### Scout Mode capabilities (always interactive TUI features):
- Process list with kill affordance
- Port Manager (full three-tier port view)
- Services health panel
- Multi-select and batch kill
- Detail pane for processes, ports, workspaces
- Background data refresh

### Inline Mode capabilities (read-only summary only):
- Workspace name
- Health dot (single character color indicator)
- Process count (`4p`)
- Primary active port (`:3000`)
- Git branch
- Secret load status (`3 secrets loaded`)
- Nothing that requires user interaction

### Will never be in Inline Mode:
- Kill operations — interactive action, requires full TUI
- Multi-select — requires persistent cursor state
- Detail pane expansion — requires TUI layout
- Global Mode tab switching — full TUI only

---

## Why Inline Was Deferred to M1.5

Three reasons for deferral:

1. **Different workflow targeting**: Inline Mode targets a workflow — ad-hoc terminals without fixed layouts — that the M1 work does not serve directly. The primary M1 user has a cmux layout with a dedicated Soji pane. Inline Mode is a second audience, and building for two audiences at once dilutes focus.

2. **P10k integration complexity**: Making Inline Mode feel native alongside Powerlevel10k requires understanding the prompt rendering pipeline, segment timing, and potential conflicts with existing P10k segments (git, node version, etc.). This is non-trivial and not on the critical path for M1 journeys (J1, J3, J4, J5, J8, J10).

3. **Scout Mode is sufficient for M1**: The M1 user journeys are all served by Scout Mode + the full interactive TUI. Adding Inline Mode in M1 would be over-building.

---

## Decision Guide

Use this to decide which mode applies to a given situation:

**"I want to see my workspace health while I code, without losing screen space to a full TUI"**
→ Use Scout Mode in a dedicated pane. The pane is narrow; Scout Mode gives you what you need when you glance at it.

**"I don't use cmux / I work in a single terminal window"**
→ Wait for Inline Mode (M1.5), or use `soji status` for a point-in-time print. Scout Mode requires a separate pane to be meaningful.

**"I want to kill a process I noticed while glancing at Soji"**
→ Switch focus back to the Soji pane (FocusGained restores full TUI), kill with `k`, switch back. This is the intended Scout → Interactive → Scout flow.

**"I want Soji workspace info in my shell prompt"**
→ This is Inline Mode (M1.5). Not available in M1.

**"I'm an agent querying Soji for workspace state"**
→ Use `soji snapshot --agent` (TOON output) or `soji snapshot --format json`. Neither Scout nor Inline is for agents — agent output is a separate render path entirely.

---

## Relationship to Each Other

Scout Mode and Inline Mode are not mutually exclusive. A developer with a cmux layout can have both:
- A Soji pane in the layout running in Scout/Interactive mode
- Inline Mode hooks in their Zsh config for quick prompt-level glances in ad-hoc windows

The same `App` data model and `gather_snapshot()` function underlies both. The difference is purely in rendering: Scout uses ratatui; Inline writes to stdout.

In M2+, when `soji.toml` defines workspaces canonically, both Scout and Inline will show the same workspace model — they just present it differently.

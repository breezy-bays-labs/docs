---
title: Reference cmux Layout
description: A three-column reference cmux layout spec for using Soji in a live development workflow, with Scout Mode pane integration and tab group design.
category: design
milestone: M1
status: final
last_updated: 2026-03-06
---

# Reference cmux Layout

This document describes the recommended cmux layout for a workspace with Soji as a persistent observer pane. It is a reference, not a requirement — Soji works in any pane configuration. But this layout represents the intended usage pattern: agent session + workspace observer + filesystem navigation simultaneously visible.

---

## Three-Column Layout Overview

```
+----------------------------------+----------+----------------------+
|                                  |          |                      |
|   Column 1: Agent + Terminal     | Column 2 |   Column 3: Tools    |
|                                  |  yazi    |                      |
|   +----------------------------+ |          |  [Tab: Soji]         |
|   |                            | |          |  ┌──────────────────┐|
|   |  Claude Code agent session | |          |  | ⬡ soji  ● 4p     ||
|   |  (top 70%)                 | |          |  | :3000 :8080  main ||
|   |                            | |          |  |                  ||
|   |                            | |          |  |  claude    ████  ||
|   |                            | |          |  |  node      ██░░  ||
|   |                            | |          |  |  zed       █░░░  ||
|   |                            | |          |  |                  ||
|   +----------------------------+ |          |  └──────────────────┘|
|   |                            | |          |                      |
|   |  Quick terminal (bot 30%)  | |          |  [Tab: lazygit]      |
|   |                            | |          |  [Tab: runner/logs]  |
|   +----------------------------+ |          |                      |
+----------------------------------+----------+----------------------+
  ~60-70% of terminal width        ~20%           ~20-25%
```

---

## Column Descriptions

### Column 1: Agent + Terminal (~60–70% width)

The primary work column. Two panes stacked vertically:

**Top pane (70% height): Claude Code agent session**
- Command: `claude` (or `claude --workspace soji`)
- This is where the developer spends most active focus time
- The agent reads, plans, and executes code changes here
- When focus is here, Soji's pane in Column 3 transitions to Scout Mode
- Width: as wide as possible — Claude Code benefits from a wide viewport for displaying diffs and file trees

**Bottom pane (30% height): Quick terminal**
- Command: `zsh` (or whatever the user's shell is)
- Used for one-off commands: `cargo build`, `git log --oneline -5`, `curl localhost:3000/health`
- CWD typically follows the workspace root (maintained via `direnv` or manual `cd`)
- When focus is here, Soji similarly transitions to Scout Mode

**Why this split**: 70/30 is the natural ratio for agentic work. The agent pane needs height to show multi-file diffs. The terminal pane needs just enough height for a few lines of output and a command entry.

---

### Column 2: Directory Navigator (~20% width)

**Full height: yazi**
- Command: `y` (yazi with cd-on-exit shell hook)
- Always visible — provides persistent filesystem context while the agent works
- Width: 20% is comfortable for a file tree (typically 30–40 columns at a 180-column terminal)
- The developer glances here to verify the agent is editing the right files
- When focus is here, Soji transitions to Scout Mode

**Why a full-height navigator column**: In agentic workflows, the developer is frequently checking what files exist, what the directory structure looks like, and verifying that the agent's changes landed correctly. Having yazi always visible eliminates context-switching to check the filesystem.

**Column width note**: At narrow terminals (<120 columns total), Column 2 can be omitted. The layout degrades gracefully to two columns: agent + terminal in Column 1, Soji tools in Column 2.

---

### Column 3: Soji and Tool Tabs (~20–25% width)

A cmux window with three tabs. The developer rarely focuses on this column — it is primarily a reference pane. Switching tabs triggers `FocusLost` in the Soji pane, causing Scout Mode.

**Tab 1: Soji (default/active on workspace open)**
- Command: `soji` or `soji --workspace soji`
- Always visible when this column is focused
- Transitions to Scout Mode when Column 1 or Column 2 takes focus
- Width: 40–50 columns is sufficient for Scout Mode; wider is fine for full interactive TUI

**Tab 2: lazygit**
- Command: `lg` (lazygit alias)
- The developer switches to this tab when they want to review diffs, stage hunks, or manage branches
- Switching to this tab causes `FocusLost` in the Soji pane (Soji goes to Scout)
- Switching back to the Soji tab causes `FocusGained` (full TUI restores)

**Tab 3: Runner / logs**
- Command: `npm run dev` or `cargo watch -x run` — whatever serves the workspace's dev server
- The developer switches here to check build output, server logs, or test runner output
- Same Scout Mode transitions apply

**Why tabs instead of split panes in Column 3**: Column 3 is narrow (~40–50 columns). Stacking panes vertically would give each insufficient height. Tabs give each tool its full column height when focused and keep the other tools running in the background. The `FocusLost`/`FocusGained` Scout transitions work naturally with tab switches — no special configuration needed.

---

## cmux Layout Pseudocode

```
workspace: soji
  window: main
    layout: three-column
      pane:
        split: vertical
        size: 65%
        panes:
          - command: "claude"
            size: 70%
            name: "agent"
          - command: "zsh"
            size: 30%
            name: "terminal"
      pane:
        command: "y"
        size: 18%
        name: "navigator"
      pane:
        size: 17%
        name: "tools"
        tabs:
          - name: "soji"
            command: "soji"
          - name: "git"
            command: "lg"
          - name: "dev"
            command: "bun run dev"
```

This pseudocode is illustrative — actual cmux layout format depends on cmux's automation surface (to be confirmed in M1.5 research spike, issue #2). The intent is clear: three columns, six surfaces, Soji as one tab in the rightmost column.

---

## Scout Mode Transitions in This Layout

The Scout → Interactive transitions happen naturally with normal workflow. The developer never explicitly activates or deactivates Scout Mode.

**Scenario: Agent is writing code**
1. Developer focuses the Claude Code pane (Column 1, top)
2. `FocusLost` fires in the Soji pane (Column 3, Tab 1)
3. Soji renders Scout compact layout:
   ```
   ⬡ soji  ● 4p  :3000  main
     claude        ████████  487 MB
     rust-analyzer ████░░░░  212 MB
     bun           ██░░░░░░  104 MB
   ```
4. Developer works in agent pane; Soji quietly updates every 5 seconds

**Scenario: Developer checks port conflict**
1. Developer notices port `:3000` is occupied from yesterday
2. Developer switches focus to the Soji pane (Tab 1, Column 3)
3. `FocusGained` fires — full interactive TUI restores instantly
4. Developer presses `2` → Ports tab, finds the node process
5. Kills it with `k` + `k` confirm
6. Developer switches back to Column 1 → Scout Mode activates again

**Scenario: Developer switches to lazygit**
1. Developer clicks or `Ctrl-n`s to Tab 2 (lazygit) in Column 3
2. `FocusLost` fires for the Soji tab (Tab 1)
3. Soji enters Scout Mode (even though it's a background tab — it is not visible, but the event fires)
4. Developer finishes git operations, switches back to Tab 1
5. `FocusGained` fires → full TUI restores

---

## Why This Layout Works

**Everything in one viewport**: The developer can see the agent working (Column 1), the filesystem (Column 2), and Soji ambient status (Column 3) simultaneously, without context switching between windows.

**No screen real estate waste**: Column 3 is narrow enough that in Scout Mode, it provides useful information without occupying more than 20% of the terminal width. When the developer needs the full TUI, they simply focus the pane — Scout to Interactive costs one keystroke (a click or a pane-focus shortcut).

**cmux tab design matches focus patterns**: The developer rarely needs lazygit and the dev server simultaneously with Soji — they tab between them. The tab structure encodes this mutual exclusivity explicitly rather than fighting against it.

**Graceful without cmux**: If the developer is not using cmux, Soji runs in a standalone terminal window. Scout Mode still works (FocusLost fires when any other application takes focus), but the transitions are between terminal windows rather than panes, which is less seamless. The layout guidance above is the ideal; it degrades to "useful but less integrated" without cmux.

---

## Customization Notes

This reference layout assumes a single primary Claude session in Column 1. For multi-agent workflows (multiple Claude agents running in parallel on different worktrees), the layout adapts:

- Column 1 may have multiple vertical splits: one per active agent session
- Column 3 Soji tab may be set to `soji --global` to see all workspaces at once rather than being scoped to a single workspace
- In Global Mode, Scout Mode header shows `⬡ global  3 workspaces  12p` instead of a single workspace name

The reference layout is a starting point. The principles (dedicated Soji pane, narrow column, tab grouping for related tools) apply at any scale.

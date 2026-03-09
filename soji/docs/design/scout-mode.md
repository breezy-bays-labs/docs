---
title: Scout Mode
description: Design specification for Scout Mode — Soji's ambient compact layout that activates when the pane loses focus.
category: design
milestone: M1
status: final
last_updated: 2026-03-06
---

# Scout Mode

Scout Mode is Soji's ambient state. When the cmux pane running Soji loses focus — because the user switched to their editor, a runner, or a Claude agent session — Soji transitions from the full interactive TUI to a compact, read-only layout that stays informative without demanding attention. When the user returns to the Soji pane, the full TUI restores exactly where they left off.

The goal: Soji is **always on screen, never in the way**.

---

## Why "Scout"

"Focus Mode" was rejected because Ghostty uses "Focus Mode" for its pane zoom feature, creating a naming collision that would confuse users and documentation. "Ambient Mode" is close but implies passivity — Scout is active: it watches, it monitors, it reports. "Scout" captures the intent: Soji is scouting the environment while you work elsewhere.

---

## How It Works

### Activation and Deactivation

Scout Mode is driven entirely by crossterm's focus change events. No polling, no timers, no manual toggle.

**Setup** (M1 implementation in `src/tui/mod.rs`):
```
crossterm::execute!(terminal.backend_mut(), EnableFocusChange)
```

This tells the terminal emulator to emit `FocusGained` and `FocusLost` events when the pane gains or loses focus. cmux, Zellij, and Ghostty all support this escape sequence.

**Transition to Scout** (`src/tui/events.rs`, `handle_focus`):
```
Event::FocusLost => {
    app.prev_mode = Some(app.mode.clone());
    app.mode = AppMode::Scout;
}
```

`prev_mode` stores whether the app was in Global or Workspace mode before the transition. This field is the only state needed to restore the correct layout on focus regain.

**Transition out of Scout** (`src/tui/events.rs`, `handle_focus`):
```
Event::FocusGained => {
    app.mode = app.prev_mode.take().unwrap_or(AppMode::Workspace);
}
```

`take()` consumes `prev_mode` (sets it back to `None`) and returns the stored value. If `prev_mode` was somehow empty (e.g., Soji launched directly into Scout), default to Workspace mode. The full interactive TUI redraws immediately on the next render tick.

The background 5-second refresh continues during Scout Mode. The compact layout always shows current data, not stale data from when focus was lost.

---

## Scout Layout (M1)

The Scout layout is optimized for a narrow pane (~40–60 columns wide). It is non-interactive: no cursor, no keybinding hints (except `q` to quit), no detail pane. The user cannot navigate or kill from Scout Mode — they must return focus to do that.

### Header Row

Single line at the top of the pane. Two variants:

**Workspace Mode was active**:
```
⬡ soji  ● 4p  :3000 :8080  main
```
- `⬡` — Soji hex glyph, workspace indicator
- `soji` — workspace name
- `● 4p` — green dot (healthy) or yellow dot (orphan present) + process count
- `:3000 :8080` — listening ports owned by this workspace (workspace-bound only)
- `main` — current git branch (from cmux sidebar-state or `git -C {cwd} branch --show-current`)

**Global Mode was active** (no workspace context):
```
⬡ global  3 workspaces  12p
```
- `global` — literal string indicating global context
- `3 workspaces` — count of known workspaces
- `12p` — total process count across all workspaces

**Health dot logic**:
- Green `●` — all processes workspace-bound or known shared infra; no orphans detected
- Yellow `●` — at least one process classified Unaffiliated or a detached Claude session present
- Red `●` — a process in kill_confirm state (unlikely in Scout, but possible if Scout activated mid-kill)

### Body Rows

Top N processes by memory descending, where N = available terminal height minus 2 (header + status bar). Each row:

```
  claude        ████████░░░░  487 MB
  rust-analyzer ████░░░░░░░░  212 MB
  node          ██░░░░░░░░░░  104 MB
  zed           █░░░░░░░░░░░   88 MB
```

Format:
- 2-space indent
- `{name:<14}` — process name, left-aligned, truncated at 14 chars
- Memory bar: 12 characters wide, filled proportionally to the highest-memory process in the list
- `{mem:>6} MB` — right-aligned memory in megabytes

The memory bar is the visual anchor — it lets the user see at a glance if something is consuming disproportionate resources without reading numbers. The highest-memory process in the current snapshot fills all 12 bar characters; all others scale proportionally.

**Bar character set**: `█` (full), `▓` (three-quarter), `▒` (half), `░` (empty). For M1, use `█` and `░` only (full and empty) for implementation simplicity.

**Process count cap**: Show at most 8 processes in Scout Mode. If there are more, a footer row:
```
  + 3 more
```

**No processes**: If the workspace has no tracked processes:
```
  (no active processes)
```

### Status Bar

Single line at the bottom of the pane:
```
[focus to interact]  q quit
```

This tells the user exactly how to re-enter interactive mode.

---

## What Changes Across Mode Transitions

When transitioning into Scout Mode, the following UI state is preserved in `App`:
- `cursor_index` — restored on focus regain; the user lands on the same row
- `selected` (HashSet) — preserved; multi-select state is not lost
- `kill_confirm` — preserved but visually inactive in Scout (no confirmation prompt shown); if the user returns focus with a kill_confirm pending, it resumes
- `active_tab` (Global Mode) — preserved; returning to Global Mode shows the same tab

When transitioning out of Scout Mode (FocusGained), no state changes occur — `app.mode` is simply restored, and the next render tick draws the full layout.

---

## What Is NOT in M1

The following Scout Mode capabilities are explicitly deferred to M1.5 or later:

**Surface-type-aware body**: Showing different data in the Scout body based on which surface type is focused in adjacent panes (e.g., "editor focused next pane — showing recent changes"). This requires cmux to emit focus events per-pane for other panes, not just the Soji pane. cmux automation capabilities are not confirmed for M1. Deferred to M1.5 once the cmux automation research spike (issue #2) is complete.

**Animated transitions**: Crossfade or slide animations between Scout and full TUI. Not worth the complexity for M1. The transition is instant.

**Inline Scout (prompt integration)**: A stripped-down version of Scout rendered as a prompt segment alongside Powerlevel10k. Different implementation, different use case. See `scout-vs-inline.md`.

**Per-surface-type detail**: Showing language server status when an editor is focused, or showing `cargo watch` output preview when a runner is focused. Deferred to M1.5.

---

## cmux Setup Guidance

Scout Mode works best when Soji has a **dedicated pane** in the cmux layout — a pane that is always visible but not always focused. The user works in other panes; Soji sits in the corner watching.

**Recommended configuration**:
- Rightmost column in a multi-column layout
- Width: 40–60 characters (sufficient for the Scout header and process bars)
- Command: `soji` (or `soji --workspace <name>` if you always want Workspace Mode)
- Do not set the pane to auto-close on exit — Soji should be persistent

**Why rightmost**: Most editor splits and terminal splits grow leftward. The rightmost column tends to be the most stable in terms of layout reflows, and it mirrors the convention of status displays (status bars, sidebars) appearing on the right.

**Width note**: At 40 columns, the Scout header fits comfortably. At fewer than 30 columns, the port list in the header may truncate. At 60+ columns, there is whitespace but nothing breaks.

**Tab groups**: In Column 3 of the reference layout (see `reference-layout.md`), Soji occupies a tab in a three-tab cmux window. Switching to the lazygit or runner tab causes `FocusLost` in the Soji pane, triggering Scout Mode. Switching back causes `FocusGained`. This is the natural transition — no manual configuration needed.

---

## Implementation Notes

### Files to Create/Modify

| File | Change |
|---|---|
| `src/tui/mod.rs` | Add `EnableFocusChange` to terminal setup sequence |
| `src/tui/events.rs` | Add `handle_focus()` for `FocusGained`/`FocusLost` events |
| `src/tui/app.rs` | Add `AppMode::Scout` variant, `prev_mode: Option<AppMode>` field |
| `src/tui/ui/scout.rs` | New file: `draw_scout(frame, area, app)` |
| `src/tui/ui/mod.rs` | Route `AppMode::Scout` to `draw_scout()` in dispatch |

### Terminal Emulator Compatibility

`EnableFocusChange` is supported by:
- Ghostty (confirmed — primary terminal for this project)
- Zellij panes (confirmed — used for cmux-adjacent workflows)
- iTerm2, Alacritty, WezTerm, Kitty (all support `\x1b[?1004h` focus tracking)
- macOS Terminal.app — not supported; FocusLost/FocusGained events will never fire. Soji gracefully degrades: without focus events, it stays in full interactive mode permanently. No crash, no visual corruption.

The crossterm `EnableFocusChange` call is safe to emit to any terminal — unsupporting terminals simply ignore the escape sequence.

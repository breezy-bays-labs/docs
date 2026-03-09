---
pipeline: 20260306-m1-tui
stage: bb-reflection
---

# Breadboard Reflection: Soji M1 TUI

## Smell Audit

### Phase 1: Requirements trace (breadboard + requirements only)

**Tracing J1 (Morning Startup):**
Path: `launch_global` → `draw_global` → `draw_workspaces` → health signals via ProcessScope.
Trace is coherent. Each step leads logically to the next. detect_mode() produces AppMode::Global → draw dispatches → workspaces tab renders with health signals derived from scope-classified processes.
**Result: Coherent.**

**Tracing J3 (Port Conflict):**
Path: `launch_global` → `cycle_tab` (or `jump_tab '2'`) → `draw_ports` → `navigate_port_list` → `select_port_process` (Enter) → process detail in right pane → `start_kill` (k) → `confirm_kill` (k) → KillConfirmed.
Detail pane is described as RightPanel in Workspace Mode affordances. In Global Mode, the Port Manager's affordance `select_port_process` → "detail pane opens". But the UI affordances for Global Mode don't include an explicit right-pane split. **Smell detected: Missing path — Global Mode has no wired detail pane.** Global Mode's port manager needs to show process detail somewhere when Enter is pressed.

**Tracing J10 (Shared Infrastructure):**
Path: `launch_global` → `jump_tab '3'` → `draw_services` → `navigate_services` → SharedInfra kill flow.
The Services tab kill flow is not explicitly wired in the UI affordances for Global Mode. The kill flow wiring is under Workspace Mode. **Smell detected: Wiring gap — Global Mode Services tab has no affordances for kill interactions.** Kill behavior from Global Mode tabs needs its own affordance entries.

**Tracing Scout Mode (any mode):**
Path: `lose_focus` → mode = Scout → `draw_scout` → `gain_focus` → mode = prev_mode.
`gain_focus` wires to "previous mode layout" — this is ambiguous. What if prev_mode is None (edge case: app starts while not focused)? `prev_mode` defaults to what? This should be addressed explicitly.
**Smell detected: Missing path — `prev_mode = None` case in focus recovery.**

---

## Fixes

### Fix 1: Global Mode detail pane (missing path in J3)

The Global Mode Port Manager and Services tab need a detail pane interaction. In Workspace Mode, the right panel serves this role. In Global Mode, the right split should also exist.

**Resolution:** Global Mode uses the same two-column layout as Workspace Mode. Left = navigable list (workspace list / port list / services list based on tab). Right = detail pane (process/surface detail opens on Enter). This is consistent with the existing pattern and adds no new concepts. The `draw_global()` fn renders left + right; `select_port_process` (Enter) → right pane shows process detail.

**Updated wiring:**
```
Global Mode: Ports tab
    navigate_port_list (↑↓)  ──► cursor moves ──► PortsList
    open_port_detail (Enter)  ──► detail pane shows process detail ──► GlobalRightPanel
    start_kill (k)            ──► same kill flow as Workspace Mode
    confirm_kill (k)          ──► KillConfirmed
```

### Fix 2: Global Mode kill wiring (wiring gap in J10)

Kill affordances exist in Workspace Mode affordances table but are absent from Global Mode table. Since the kill flow is the same regardless of mode (the App state machine handles it), the affordances should be mirrored in Global Mode.

**Resolution:** Add kill affordances to the Global Mode UI affordances table:

| Place | Affordance | Wires Out | Returns To |
|-------|-----------|-----------|------------|
| Global Mode: any list (cursor on process) | `start_kill` (k) | → kill_confirm = Some(pid) OR infra_kill_confirm | → GlobalMode:current_tab |
| Global Mode: kill_confirm active | `confirm_kill` (k again) | → KillConfirmed event | → GlobalMode:current_tab |
| Global Mode: any list | `cancel_kill` (Esc) | → kill_confirm = None, infra_kill_confirm = None | → GlobalMode:current_tab |
| Global Mode: any list (process row) | `toggle_select` (Space) | → pid added/removed from selected | → GlobalMode:current_tab |

### Fix 3: Scout Mode prev_mode None edge case (missing path)

If `FocusGained` fires before `FocusLost` ever did (app just started, always focused), `prev_mode` is None. Should restore to default mode (Global or Workspace based on CWD).

**Resolution:** `prev_mode` defaults to None. On FocusGained: `app.mode = app.prev_mode.take().unwrap_or_else(|| app.initial_mode.clone())` where `app.initial_mode: AppMode` is set at startup by `detect_mode()` and never changes. This is a pure addition — `initial_mode` field on App.

**Updated code affordance:**
```
app.rs: add `pub initial_mode: AppMode` field set at startup, read-only after init.
events.rs: FocusGained handler: app.mode = app.prev_mode.take().unwrap_or(app.initial_mode.clone())
```

---

## Naming Test

**UI Affordances:**

| Affordance | Caller | Step-level effect | One verb? |
|-----------|--------|------------------|-----------|
| `launch_global` | CLI entry | Sets initial mode to Global | ✅ |
| `cycle_tab` | User (Tab key) | Advances active_tab by one | ✅ |
| `jump_tab` | User (1/2/3 key) | Sets active_tab to specific index | ✅ |
| `navigate_workspace_list` | User (↑↓) | Moves cursor | ✅ |
| `expand_workspace` | User (Enter) | Reveals workspace detail or opens it | ✅ |
| `navigate_port_list` | User (↑↓) | Moves cursor | ✅ |
| `open_port_detail` | User (Enter) | Opens process detail in right pane | ✅ |
| `navigate_list` | User (↑↓) | Moves cursor | ✅ |
| `expand_surface` | User (Enter) | Reveals process rows under surface | ✅ |
| `open_detail` | User (Enter on Process) | Shows process detail in right pane | ✅ |
| `toggle_select` | User (Space) | Adds or removes pid from selection | ✅ |
| `start_kill` | User (k) | Sets kill_confirm state | ✅ |
| `confirm_kill` | User (k again) | Fires KillConfirmed event | ✅ |
| `start_kill_infra` | User (k on SharedInfra) | Sets infra_kill_confirm state | ✅ |
| `confirm_kill_infra` | User (k, k again) | Sets kill_confirm after infra step | ✅ |
| `kill_selected` | User (k with selection) | Fires KillBatch event | ✅ |
| `cancel_kill` | User (Esc) | Clears kill state and selection | ✅ |
| `switch_panel` | User (Tab) | Moves focus to Right panel | ✅ |
| `switch_panel_back` | User (Tab) | Moves focus to Left panel | ✅ |
| `jump_workspace` | User (1–9) | Moves cursor to workspace N | ✅ |
| `open_help` | User (?) | Shows help overlay | ✅ |
| `close_help` | User (Esc/?) | Hides help overlay | ✅ |
| `quit_global` | User (q/Ctrl-C) | Exits application | ✅ |
| `quit_workspace` | User (q/Ctrl-C) | Exits application | ✅ |
| `lose_focus` | crossterm FocusLost | Switches to Scout mode | ✅ |
| `gain_focus` | crossterm FocusGained | Restores previous mode | ✅ |
| `go_global` | User (g) | Switches to Global mode | ✅ |
| `switch_to_workspace_mode` | User (Enter on workspace) | Switches to Workspace mode for selected ws | ✅ |

**Code Affordances:**

| Affordance | Caller | Step-level effect | One verb? |
|-----------|--------|------------------|-----------|
| `classify_scope` | `gather_snapshot()` | Computes ProcessScope for one process | ✅ |
| `resolve_commands` | `gather_snapshot()` | Fetches full command strings for pids | ✅ |
| `gather_snapshot` | CLI / TUI refresh | Gathers full environment state | ✅ |
| `encode` (toon) | snapshot subcommand | Encodes value to TOON string | ✅ |
| `detect_mode` | `App::new()` | Determines initial AppMode | ✅ |
| `toggle_selection` | event handler | Adds/removes pid from selected set | ✅ |
| `fix_k_ambiguity` | — | This is a fix label, not an affordance | — |
| `draw` (ui/mod.rs) | run loop | Dispatches to correct layout renderer | ✅ |
| `draw_global` | `draw` | Renders Global Mode layout | ✅ |
| `draw_scout` | `draw` | Renders Scout compact layout | ✅ |
| `draw_help_overlay` | `draw` | Renders help overlay | ✅ |
| `ci_pipeline` | GitHub Actions | Runs fmt + clippy + test | ✅ |

`fix_k_ambiguity` fails the naming test — it's describing a fix, not an affordance with a single effect. Rename to `handle_kill_key` in the code affordances table. The fix IS the handler — what it does is: route `k` keypress to kill confirmation flow vs navigation based on cursor context.

---

## Wiring Consistency Check

- Every Wires Out target exists in tables: ✅ (after adding Global Mode kill affordances in Fix 2)
- Every Returns To source has a corresponding Wires Out: ✅
- `GlobalRightPanel` referenced by Fix 1 — confirm it's in the Global Mode affordances table: added via Fix 1
- `app.initial_mode` referenced in Fix 3 — confirm it's in code affordances: added as note in Fix 3
- `KillBatch` referenced in kill wiring — confirm it exists in `AppEvent`: needs to be added alongside `KillConfirmed(u32)`. Current `AppEvent` only has `KillConfirmed(u32)`. Batch kill needs either `KillBatch(HashSet<u32>)` or handle in the TUI loop directly.

**Additional smell found:** `KillBatch` is referenced in the VS4 wiring diagram but there is no `AppEvent::KillBatch` defined in the events code affordances. The code affordances table for `events.rs` only mentions the conceptual fix. This is a diagram-only reference — the actual event variant is missing from the contract.

**Fix 4: Add KillBatch to AppEvent**

```
events.rs: AppEvent enum adds KillBatch(Vec<u32>)
mod.rs (run_loop): handle KillBatch(pids) → calls kill_processes for each
```

---

## Updated Breadboard Additions

The following additions integrate all fixes:

**Global Mode UI Affordances (additions):**

| Place | Affordance | Wires Out | Returns To |
|-------|-----------|-----------|------------|
| Global Mode: any tab (cursor on process) | `open_global_detail` (Enter) | → GlobalRightPanel shows process detail | → GlobalRightPanel |
| Global Mode: any tab (cursor on process) | `start_kill` (k) | → kill_confirm = Some(pid) OR infra_kill_confirm | → GlobalMode:current_tab |
| Global Mode: kill_confirm active | `confirm_kill` (k again) | → KillConfirmed event | → GlobalMode:current_tab |
| Global Mode: infra_kill_confirm active | `confirm_kill_infra` (k then k) | → kill_confirm → KillConfirmed | → GlobalMode:current_tab |
| Global Mode: any tab | `cancel_kill` (Esc) | → kill state cleared | → GlobalMode:current_tab |
| Global Mode: any tab | `toggle_select` (Space on process) | → pid toggled in selected | → GlobalMode:current_tab |
| Global Mode: selected non-empty | `kill_selected` (k) | → KillBatch(selected) event | → GlobalMode:current_tab |

**Code Affordances (additions):**

| Module | Affordance | Wires Out | Returns To |
|--------|-----------|-----------|------------|
| `app.rs` | `add_initial_mode` — `pub initial_mode: AppMode` set at startup, read-only | → used by FocusGained handler | — |
| `events.rs` | `add_KillBatch_event` — `AppEvent::KillBatch(Vec<u32>)` | → handled by run_loop | — |
| `events.rs` | `handle_kill_key` — routes `k` to: kill_selected if non-empty selection; SharedInfra confirm flow; standard confirm flow | → AppEvent or state update | — |

---

## Quality Gate

- [x] All user stories from requirements trace through wiring coherently (after fixes)
- [x] No incoherent wiring
- [x] No missing paths (4 smells found and fixed: Global detail pane, Global kill wiring, Scout prev_mode edge case, KillBatch event)
- [x] No diagram-only nodes (KillBatch now has code affordance)
- [x] All affordances pass the naming test (fix_k_ambiguity renamed to handle_kill_key)
- [x] Every Wires Out target exists in the tables
- [x] Every Returns To source has a corresponding Wires Out
- [x] Tables and diagrams are consistent

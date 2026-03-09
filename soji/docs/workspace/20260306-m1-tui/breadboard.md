---
pipeline: 20260306-m1-tui
stage: breadboard
---

# Breadboard: Soji M1 TUI

## Vertical Slices

| Slice | Description | Shape Parts |
|-------|-------------|-------------|
| VS1 | Domain + classifier foundation | A1, A2 |
| VS2 | CLI wiring + agent output | A4, A9, A10 |
| VS3 | Global Mode TUI | A3, A5, A6 |
| VS4 | Multi-select kill + bug fixes | A7, A11, A12, A13 |
| VS5 | Scout Mode | A8 |
| VS6 | Tests + CI | A14, A15, A16, A17 |
| VS7 | M1 docs | A18 |

---

## UI Affordances

### Global Mode

| Place | Affordance | Wires Out | Returns To |
|-------|-----------|-----------|------------|
| App launch (outside workspace CWD) | `launch_global` | → GlobalMode:WorkspacesTab | — |
| Global Mode: Tab bar | `cycle_tab` (Tab key) | → active_tab updated | → GlobalMode:current_tab |
| Global Mode: Tab bar | `jump_tab` (1/2/3 keys) | → active_tab = Workspaces/Ports/Services | → GlobalMode:current_tab |
| Global Mode: Workspaces tab | `navigate_workspace_list` (↑↓) | → cursor moves | → WorkspacesList |
| Global Mode: Workspaces tab | `expand_workspace` (Enter) | → workspace drill-in or launch | → WorkspaceDetail |
| Global Mode: Ports tab | `navigate_port_list` (↑↓) | → cursor moves | → PortsList |
| Global Mode: Ports tab | `select_port_process` (Enter) | → detail pane opens | → ProcessDetail |
| Global Mode: Services tab | `navigate_services` (↑↓) | → cursor moves | → ServicesList |
| Global Mode: any tab | `quit_global` (q/Ctrl-C) | → terminal restored | — |
| Global Mode: any tab | `open_help` (?) | → help overlay shown | → HelpOverlay |
| Global Mode: any tab | `switch_to_workspace_mode` | → mode = Workspace for selected ws | → WorkspaceMode |

### Workspace Mode

| Place | Affordance | Wires Out | Returns To |
|-------|-----------|-----------|------------|
| App launch (inside workspace CWD) | `launch_workspace` | → WorkspaceMode | — |
| `--workspace <name>` flag | `launch_workspace_named` | → WorkspaceMode for named ws | — |
| Left panel | `navigate_list` (↑↓) | → cursor moves | → LeftPanel |
| Left panel: Surface row | `expand_surface` (Enter) | → process rows revealed under surface | → LeftPanel |
| Left panel: Process row | `open_detail` (Enter) | → RightPanel shows process detail | → RightPanel |
| Left panel: Process row | `toggle_select` (Space) | → pid added/removed from selected HashSet | → LeftPanel |
| Left panel: any row | `start_kill` (k, cursor on process) | → kill_confirm = Some(pid) | → LeftPanel |
| Left panel: kill_confirm active | `confirm_kill` (k again) | → KillConfirmed event → process killed | → LeftPanel |
| Left panel: SharedInfra process | `start_kill_infra` (k) | → infra_kill_confirm = Some(pid) | → LeftPanel |
| Left panel: infra_confirm active | `confirm_kill_infra` (k, k again) | → KillConfirmed event | → LeftPanel |
| Left panel: multi-select active | `kill_selected` (k) | → KillConfirmed batch | → LeftPanel |
| Left panel | `cancel_kill` (Esc) | → kill_confirm = None, selected cleared | → LeftPanel |
| Left panel | `switch_panel` (Tab) | → focused_panel = Right | → RightPanel |
| Left panel | `jump_workspace` (1–9) | → cursor jumps to workspace N | → LeftPanel |
| Right panel | `switch_panel_back` (Tab) | → focused_panel = Left | → LeftPanel |
| Any | `open_help` (?) | → help overlay shown | → HelpOverlay |
| Any | `close_help` (Esc / ?) | → help overlay hidden | → current panel |
| Any | `quit_workspace` (q / Ctrl-C) | → terminal restored | — |
| Header `← global` hint | `go_global` (g key) | → mode = Global | → GlobalMode |

### Scout Mode

| Place | Affordance | Wires Out | Returns To |
|-------|-----------|-----------|------------|
| App running (any mode) | `lose_focus` (FocusLost event) | → mode = Scout, draws compact layout | → ScoutLayout |
| Scout layout | `gain_focus` (FocusGained event) | → mode = previous (Global or Workspace) | → previous mode layout |
| Scout layout | `quit_scout` (q / Ctrl-C) | → terminal restored | — |

### Help Overlay

| Place | Affordance | Wires Out | Returns To |
|-------|-----------|-----------|------------|
| HelpOverlay | `close_help` (Esc / ?) | → show_help = false | → underlying layout |

---

## Code Affordances

### Domain Layer (`src/domain/`)

| Module | Affordance | Wires Out | Returns To |
|--------|-----------|-----------|------------|
| `process.rs` | `define_ProcessScope` — add enum `ProcessScope { WorkspaceBound { workspace_name }, SharedInfra { service_name }, Unaffiliated }` with Serialize/Deserialize | → used by Process struct | — |
| `process.rs` | `add_scope_field` — add `pub scope: ProcessScope` to `Process` struct | → scope available everywhere Process is used | — |

### Sources Layer (`src/sources/`)

| Module | Affordance | Wires Out | Returns To |
|--------|-----------|-----------|------------|
| `scope.rs` | `classify_scope(process, workspaces) -> ProcessScope` — pure fn: CWD prefix match → WorkspaceBound; port/name heuristic → SharedInfra; else → Unaffiliated | → called by `gather_snapshot()` for each process | → ProcessScope value |
| `command.rs` | `resolve_commands(pids) -> HashMap<u32, String>` — runs `ps -p <csv pids> -o pid=,args=` once, parses output | → called by `gather_snapshot()` | → full command strings by PID |
| `mod.rs` (gather_snapshot) | `gather_snapshot()` — existing fn; extended to call `resolve_commands()` then `classify_scope()` on each process | → EnvironmentSnapshot with scope + full commands | — |

### Output Layer (`src/output/`)

| Module | Affordance | Wires Out | Returns To |
|--------|-----------|-----------|------------|
| `toon.rs` | `encode(value: &impl Serialize) -> String` — TOON v3.0 encoder. Type-prefix scalars, compact arrays, key abbreviation map for Soji types | → called by snapshot subcommand | → TOON string |
| `toon.rs` | `SOJI_KEY_MAP` — static map of common Soji field names to single-char abbreviations (pid→p, memory_mb→m, workspace_name→w, scope→sc, command→c, status→st) | → used by encoder | — |

### TUI App State (`src/tui/app.rs`)

| Module | Affordance | Wires Out | Returns To |
|--------|-----------|-----------|------------|
| `app.rs` | `add_AppMode` — enum `AppMode { Global, Workspace, Scout }` | → App.mode field | — |
| `app.rs` | `add_GlobalTab` — enum `GlobalTab { Workspaces, Ports, Services }` | → App.active_tab field | — |
| `app.rs` | `add_selection` — `pub selected: HashSet<u32>` field on App; `toggle_selection(pid)` method | → used by kill flow and UI rendering | — |
| `app.rs` | `add_help_flag` — `pub show_help: bool` field on App | → used by help overlay renderer | — |
| `app.rs` | `add_infra_confirm` — `pub infra_kill_confirm: Option<u32>` distinct from `kill_confirm` | → SharedInfra extra confirmation | — |
| `app.rs` | `add_prev_mode` — `pub prev_mode: Option<AppMode>` — stores mode before Scout transition | → restore on FocusGained | — |
| `app.rs` | `detect_mode()` — extends `detect_workspace()`: checks `--global` flag, `--workspace` flag, CWD; returns `AppMode` | → sets `App.mode` at startup | → AppMode |

### TUI Events (`src/tui/events.rs`)

| Module | Affordance | Wires Out | Returns To |
|--------|-----------|-----------|------------|
| `events.rs` | `fix_k_ambiguity` — kill arm `Char('k')` as first match when cursor is on Process row. Navigation: `Up`/`Down` arrow keys + `Char('j')`/`Char('k')` only when NOT on Process row or when no kill pending | → clean separation of nav vs kill | — |
| `events.rs` | `handle_space` — `Char(' ')` toggles current pid into/out of `app.selected` | → updates App.selected | — |
| `events.rs` | `handle_question` — `Char('?')` sets `app.show_help = !app.show_help` | → triggers help overlay | — |
| `events.rs` | `handle_1_to_9` — `Char('1')..=Char('9')` in Global Mode switches tab or jumps workspace; in Workspace Mode jumps workspace | → updates cursor or active_tab | — |
| `events.rs` | `handle_focus` — `Event::FocusGained` / `Event::FocusLost` transitions `app.mode` | → mode = Scout on lost, prev_mode on gained | — |
| `events.rs` | `handle_g` — `Char('g')` in Workspace Mode transitions to Global | → app.mode = Global | — |

### TUI UI Layer (`src/tui/ui/`)

| Module | Affordance | Wires Out | Returns To |
|--------|-----------|-----------|------------|
| `ui/mod.rs` | `draw(frame, app)` — dispatches to correct layout renderer based on `app.mode` | → calls global/workspace/scout draw fn | — |
| `ui/global.rs` | `draw_global(frame, area, app)` — renders Global Mode (tab bar + active tab content) | → calls draw_workspaces / draw_ports / draw_services | — |
| `ui/global.rs` | `draw_workspaces(frame, area, app)` — workspace list with health signals | → renders list widget | — |
| `ui/global.rs` | `draw_ports(frame, area, app)` — port manager, all ports by scope tier | → renders port table | — |
| `ui/global.rs` | `draw_services(frame, area, app)` — SharedInfra processes | → renders services list | — |
| `ui/scout.rs` | `draw_scout(frame, area, app)` — compact Scout layout: workspace header + fallback body | → renders compact layout | — |
| `ui/help.rs` | `draw_help_overlay(frame, app)` — full-screen or centered overlay with all keybindings | → renders over current layout | — |
| `ui/left.rs` | enhance `draw_left`: add selection indicator (checkbox) when `app.selected` contains pid | → renders selected state | — |

### CLI (`src/main.rs`)

| Module | Affordance | Wires Out | Returns To |
|--------|-----------|-----------|------------|
| `main.rs` | `add_global_flag` — `--global` bool flag on `Cli` | → passed to `detect_mode()` | — |
| `main.rs` | `add_workspace_flag` — `--workspace <name>` Option<String> on `Cli` | → passed to `detect_mode()` | — |
| `main.rs` | `add_snapshot_subcommand` — `Commands::Snapshot { agent: bool, format: OutputFormat }` where `OutputFormat = Json | Toon` | → calls `gather_snapshot()` then `output::encode()` | — |

### Tests (`src/tests/`)

| Module | Affordance | Wires Out | Returns To |
|--------|-----------|-----------|------------|
| `scope_tests.rs` | unit tests for `classify_scope()` | → verifies WorkspaceBound/SharedInfra/Unaffiliated classification | — |
| `toon_tests.rs` | unit tests for `encode()` — round-trip, empty, unicode | → verifies TOON encoder correctness | — |
| `tui_behavior.rs` | behavioral tests for keybinding contracts via `handle_key()` | → verifies Enter is never KillConfirmed, Space toggles selection, etc. | — |

### CI (`.github/workflows/`)

| Module | Affordance | Wires Out | Returns To |
|--------|-----------|-----------|------------|
| `ci.yml` | `ci_pipeline` — fmt check + clippy + test on push/PR | → blocks merge on failure | — |

---

## Wiring Diagrams

### Slice VS1: Domain + Classifier

```
gather_snapshot()
    ──► resolve_commands(pids)         ─► HashMap<u32, String>
    ──► [build process list]
    ──► for each process:
          classify_scope(process, workspaces)
              ──► [CWD prefix match?] ─► WorkspaceBound { workspace_name }
              ──► [port/name match?]  ─► SharedInfra { service_name }
              ──► [else]              ─► Unaffiliated
    ──► EnvironmentSnapshot (all processes have scope + full command)
```

### Slice VS2: Agent Output

```
soji snapshot --agent
    ──► gather_snapshot()              ─► EnvironmentSnapshot
    ──► output::toon::encode(snapshot) ─► TOON string
    ──► print to stdout

soji snapshot --format json
    ──► gather_snapshot()              ─► EnvironmentSnapshot
    ──► serde_json::to_string_pretty() ─► JSON string
    ──► print to stdout
```

### Slice VS3: Global Mode TUI

```
App::new()
    ──► detect_mode()
          ──► [--global flag?]         ─► AppMode::Global
          ──► [--workspace flag?]      ─► AppMode::Workspace(name)
          ──► [CWD in workspace?]      ─► AppMode::Workspace(detected)
          ──► [else]                   ─► AppMode::Global

draw(frame, app)
    ──► [app.mode == Global]           ─► draw_global(frame, area, app)
          ──► draw_tab_bar()
          ──► [active_tab == Workspaces] ─► draw_workspaces()
          ──► [active_tab == Ports]      ─► draw_ports()
          ──► [active_tab == Services]   ─► draw_services()
```

### Slice VS4: Multi-Select Kill + Bug Fixes

```
handle_key(app, KeyCode::Char(' '))
    ──► app.toggle_selection(current_pid)
    ──► [pid was in selected] ─► remove → deselected
    ──► [pid not in selected] ─► insert → selected

handle_key(app, KeyCode::Char('k'))
    ──► [selected.len() > 1]           ─► KillBatch(selected.clone())
    ──► [selected.len() == 1]          ─► kill single selected
    ──► [cursor on Process, selected empty]
          ──► [scope == SharedInfra]
                ──► [infra_confirm == None]  ─► infra_confirm = Some(pid)
                ──► [infra_confirm == Some] ─► [kill_confirm == None]  ─► kill_confirm = Some(pid)
                                             ─► [kill_confirm == Some] ─► KillConfirmed(pid)
          ──► [scope != SharedInfra]
                ──► [kill_confirm == None]   ─► kill_confirm = Some(pid)
                ──► [kill_confirm == Some]   ─► KillConfirmed(pid)
```

### Slice VS5: Scout Mode

```
poll_events(app, rx)
    ──► Event::FocusLost
          ──► app.prev_mode = Some(app.mode)
          ──► app.mode = AppMode::Scout
    ──► Event::FocusGained
          ──► app.mode = app.prev_mode.unwrap_or(AppMode::Workspace)
          ──► app.prev_mode = None

draw(frame, app)
    ──► [app.mode == Scout]            ─► draw_scout(frame, area, app)
          ──► draw_scout_header()       (workspace name, health, count, ports, git)
          ──► draw_scout_body()         (top processes, memory bars)
```

---

## User Story Traces

### Journey 1: Morning Startup — "What state am I in?"

1. Developer runs `soji` from `~/Github` (outside any workspace CWD)
2. `detect_mode()` → CWD not in any workspace → `AppMode::Global`
3. `draw_global()` renders Workspaces tab (default)
4. Workspace list shows each workspace with health signals (active dot, orphan warning, process count)
5. Developer sees red `⚠ orphan` on `kata` workspace — drills in with Enter
6. **Covered by**: VS3 (Global Mode), A3, A5, A6, A2 (scope for health signals)

### Journey 3: Port Conflict — "Something is using my port"

1. Developer runs `soji` → Global Mode
2. Presses `2` → jumps to Ports tab
3. Port list shows all ports by scope tier. Finds `:3000` — WorkspaceBound to `kata`, node process, 18h old
4. Developer presses Enter → detail pane shows full command (VS1, A10 fix truncation)
5. Developer presses `k` → kill_confirm. Presses `k` again → KillConfirmed → process killed
6. **Covered by**: VS1 (scope on process), VS2 (no — this is TUI not CLI), VS3 (Port Manager), VS4 (kill flow), A10 (command resolution)

### Journey 4: End of Day Cleanup — "Tidying up"

1. Developer in Global Mode, Workspaces tab
2. Navigates workspace list, sees multiple workspaces have orphans
3. Presses Tab → Ports tab, sees all cleanable processes
4. Presses Space on each orphan → builds multi-select
5. Presses `k` → KillBatch → all selected killed
6. **Covered by**: VS3, VS4 (multi-select kill), A7, A11

### Journey 10: Shared Infrastructure — "Is OrbStack running?"

1. Developer in Global Mode, presses `3` → Services tab
2. Services tab shows all SharedInfra processes: OrbStack ✓, Ollama :11434 ✓, Docker daemon ✓
3. Developer accidentally tries to kill OrbStack: `k` → infra_confirm prompt. `k` again → kill_confirm. `k` third time → KillConfirmed (or Esc to cancel)
4. **Covered by**: VS1 (SharedInfra classification), VS3 (Services tab), VS4 (extra confirmation flow)

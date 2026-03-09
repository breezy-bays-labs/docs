---
pipeline: 20260306-m1-tui
stage: impl-plan
---

# Implementation Plan: Soji M1 TUI

**Goal:** Ship a complete, shippable M1 with dual-mode launch (Global + Workspace), Port Manager, ProcessScope model, multi-select kill, Scout Mode foundation, agent output (TOON), CI, and tests.
**Architecture:** Layered build on existing ratatui/crossterm/tokio scaffold. Additive changes only — no rewrites. ProcessScope computed at gather time, flows through all layers.
**Tech Stack:** Rust, ratatui 0.29, crossterm 0.28, tokio, serde/serde_json, clap, GitHub Actions
**Appetite:** Large (full milestone)

## Execution Model

**Single worktree, single PR.** All work happens in the current worktree (`parallel-greeting-diffie` branch). No branch switching. One PR to main when all waves are done.

**Parallel work via subagents.** Sessions within a wave run in parallel using the `Agent` tool (subagents). The orchestrating session launches subagents for parallel tasks, waits for completion, then starts the next wave. Subagents operate on the same working directory — file ownership rules (see Merge Strategy) prevent conflicts.

---

## Wave Structure

```
Wave 0 (serial): Domain + Classifier + Command Resolution
    └── 0.1: domain-and-scope   (Process scope field + classifier + command resolver)

Wave 1 (parallel): Core TUI Infrastructure
    ├── 1.1: app-state-modes    (AppMode, GlobalTab, selection, help flag, initial_mode)
    ├── 1.2: cli-and-snapshot   (CLI flags, snapshot subcommand, TOON encoder)
    └── 1.3: event-fixes        (k ambiguity fix, Space, ?, 1-9, focus events, KillBatch)

Wave 2 (parallel): UI Rendering
    ├── 2.1: global-mode-ui     (Global Mode layout, tabs, workspace list, port manager, services)
    ├── 2.2: scout-mode-ui      (Scout compact layout)
    └── 2.3: help-overlay-ui    (? help overlay, enhance left panel for selection indicators)

Wave 3 (parallel): Tests + CI + Docs
    ├── 3.1: tests              (scope_tests, toon_tests, tui_behavior tests)
    ├── 3.2: ci                 (GitHub Actions workflow)
    └── 3.3: m1-docs            (docs/design/ directory with all 7 design docs)
```

---

## Wave 0: Foundation

### Task 0.1: Domain and Scope Foundation

**Topic:** `domain-and-scope`
**Depends on:** None
**Complexity:** Medium
**Files:**
- `src/domain/process.rs` — add `ProcessScope` enum + `scope` field on `Process`
- `src/sources/scope.rs` — new file: `classify_scope()` pure function
- `src/sources/command.rs` — new file: `resolve_commands()` batch ps runner
- `src/sources/mod.rs` — extend `gather_snapshot()` to call resolve_commands + classify_scope

**Steps:**

1. In `src/domain/process.rs`, add `ProcessScope` enum:
   ```rust
   #[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
   pub enum ProcessScope {
       WorkspaceBound { workspace_name: String },
       SharedInfra { service_name: String },
       Unaffiliated,
   }
   ```
   Add `pub scope: ProcessScope` field to `Process` struct (default to `Unaffiliated` in existing code that constructs Process).

2. Create `src/sources/scope.rs`:
   - `pub fn classify_scope(process: &Process, workspaces: &[Workspace]) -> ProcessScope`
   - WorkspaceBound: iterate workspaces, check if `process.cwd` starts with `workspace.cwd` (at any depth, including worktree paths). Return first match.
   - SharedInfra: static lookup table of (port, name) pairs — OrbStack (:5556), Ollama (:11434), Docker daemon, Postgres (:5432), Redis (:6379), MySQL (:3306). Match by port in `process.metadata.ports` OR by process name substring match (case-insensitive: "orbstack", "ollama", "postgres", "redis", "mysqld", "docker").
   - Unaffiliated: default case.
   - Add `SHARED_INFRA_TABLE: &[(&str, &[u16])]` static for the name+port lookup.

3. Create `src/sources/command.rs`:
   - `pub async fn resolve_commands(pids: &[u32]) -> HashMap<u32, String>`
   - Build CSV of pids: `pids.iter().map(|p| p.to_string()).collect::<Vec<_>>().join(",")`
   - Run: `ps -p <csv> -o pid=,args=`
   - Parse output: each line is `"<pid> <full command>"`, split on first space
   - Return HashMap. On any error, return empty HashMap (graceful degradation).

4. In `src/sources/mod.rs`, extend `gather_snapshot()`:
   - After building the process list, collect all pids
   - Call `resolve_commands(&pids).await`
   - Update each `process.command` with the resolved full command (fall back to existing if not found)
   - Then call `classify_scope(&process, &workspaces)` for each process, set `process.scope`

5. Update all Process construction sites (in `sources/claude.rs`, `sources/system.rs`, etc.) to include `scope: ProcessScope::Unaffiliated` as placeholder (will be overwritten by step 4).

**Acceptance:**
- `cargo build` passes
- `cargo test` (any existing tests) passes
- `Process` struct has `scope: ProcessScope` field
- `gather_snapshot()` returns processes with non-Unaffiliated scope where applicable (verify manually with `cargo run -- status`)

**Session Prompt:**
```
You are implementing the domain and scope foundation for Soji M1 (Wave 0, Task 0.1).

Context files to read first:
- CLAUDE.md — project architecture and commands
- src/domain/process.rs — current Process struct
- src/domain/workspace.rs — Workspace and Surface types
- src/sources/mod.rs — gather_snapshot() function
- src/sources/claude.rs — example of how processes are constructed
- docs/workspace/20260306-m1-tui/breadboard.md — VS1 wiring
- docs/workspace/20260306-m1-tui/shaping.md — A1, A2, A10 parts

What to build:
1. Add ProcessScope enum to src/domain/process.rs + scope field on Process
2. Create src/sources/scope.rs with classify_scope() pure function
3. Create src/sources/command.rs with resolve_commands() batch ps runner
4. Extend gather_snapshot() in src/sources/mod.rs to call both

Constraints:
- Do NOT modify any TUI files (src/tui/)
- Do NOT modify main.rs
- classify_scope() must be a pure function (no I/O, no async)
- resolve_commands() must be async (shells out to ps)
- Graceful degradation: if ps fails, return empty HashMap (don't panic)
- All Process construction sites need scope: ProcessScope::Unaffiliated placeholder
- Run `cargo build` to verify before finishing
```

---

## Wave 1: Core TUI Infrastructure

### Task 1.1: App State Modes

**Topic:** `app-state-modes`
**Depends on:** `0.1-domain-and-scope` complete
**Complexity:** Medium
**Files:**
- `src/tui/app.rs` — add AppMode, GlobalTab, selected, show_help, initial_mode, infra_kill_confirm, prev_mode, add_initial_mode

**Steps:**

1. Add enums to `src/tui/app.rs`:
   ```rust
   #[derive(Debug, Clone, PartialEq, Eq)]
   pub enum AppMode { Global, Workspace, Scout }

   #[derive(Debug, Clone, PartialEq, Eq)]
   pub enum GlobalTab { Workspaces, Ports, Services }
   ```

2. Add fields to `App` struct:
   ```rust
   pub mode: AppMode,
   pub initial_mode: AppMode,
   pub active_tab: GlobalTab,
   pub selected: HashSet<u32>,
   pub show_help: bool,
   pub infra_kill_confirm: Option<u32>,
   pub prev_mode: Option<AppMode>,
   ```

3. Extend `App::new()` to accept `mode: AppMode` parameter. Set `initial_mode = mode.clone()`, `mode = mode`, all others to defaults.

4. Add methods:
   - `pub fn toggle_selection(&mut self, pid: u32)` — insert if absent, remove if present
   - `pub fn cycle_global_tab(&mut self)` — Workspaces → Ports → Services → Workspaces
   - `pub fn jump_global_tab(&mut self, n: usize)` — 1=Workspaces, 2=Ports, 3=Services
   - `pub fn restore_mode(&mut self)` — `self.mode = self.prev_mode.take().unwrap_or(self.initial_mode.clone())`

5. Update `detect_workspace()` → rename and extend to `pub fn detect_mode(global_flag: bool, workspace_name: Option<&str>, snapshot: &EnvironmentSnapshot) -> AppMode`. Logic: global_flag → Global; workspace_name → Workspace(find by name); CWD match → Workspace(detected); else → Global.

**Acceptance:**
- `cargo build` passes
- `AppMode`, `GlobalTab` enums exist and are pub
- `App::new(snapshot, mode)` accepts mode parameter
- `toggle_selection`, `cycle_global_tab`, `restore_mode` methods exist

**Session Prompt:**
```
You are implementing App state mode additions for Soji M1 (Wave 1, Task 1.1).

Context files to read first:
- src/tui/app.rs — current App struct (ADD to this file)
- src/domain/process.rs — ProcessScope enum (already in domain from Wave 0)
- docs/workspace/20260306-m1-tui/breadboard.md — Code affordances: app.rs section
- docs/workspace/20260306-m1-tui/shaping.md — A3, A5, A7 parts

What to build:
1. Add AppMode and GlobalTab enums
2. Add new fields to App struct (mode, initial_mode, active_tab, selected, show_help, infra_kill_confirm, prev_mode)
3. Extend App::new() to accept AppMode parameter
4. Add toggle_selection, cycle_global_tab, jump_global_tab, restore_mode methods
5. Rename/extend detect_workspace() to detect_mode(global_flag, workspace_name, snapshot)

Constraints:
- Do NOT modify any UI render files (src/tui/ui/*)
- Do NOT modify events.rs yet (Wave 1.3 handles events)
- Do NOT modify main.rs yet (Wave 1.2 handles CLI)
- App::new() signature changes will break mod.rs call site — add a TODO comment at that call site
- Run `cargo build` to verify; fix any compile errors from signature change
```

### Task 1.2: CLI and TOON Output

**Topic:** `cli-and-snapshot`
**Depends on:** `0.1-domain-and-scope` complete
**Complexity:** Medium
**Files:**
- `src/main.rs` — add --global, --workspace flags, Snapshot subcommand
- `src/output/mod.rs` — new module
- `src/output/toon.rs` — new file: TOON encoder
- `src/lib.rs` or `src/main.rs` — add `mod output;`

**Steps:**

1. Add `src/output/` directory with `mod.rs` (re-exports) and `toon.rs`.

2. Implement TOON encoder in `src/output/toon.rs`:
   - `pub fn encode(value: &serde_json::Value) -> String` — takes a JSON value (serialize with serde_json first, then encode)
   - Alternatively: `pub fn encode_snapshot(snapshot: &EnvironmentSnapshot) -> String`
   - TOON v3.0 rules: type-prefix scalars (`s:string`, `n:42`, `b:true`, `null`), arrays as `[a,b,c]`, objects as `{key:value,key2:value2}`
   - Key abbreviation map for Soji types: `pid→p`, `memory_mb→m`, `workspace_name→w`, `scope→sc`, `command→cmd`, `status→st`, `cwd→d`, `kind→k`, `metadata→md`, `processes→ps`, `workspaces→ws`, `surfaces→sf`
   - Implementation strategy: serialize to `serde_json::Value` first, then walk the Value tree applying TOON encoding rules + key abbreviation

3. In `src/main.rs`:
   - Add to `Cli`: `#[arg(long)] global: bool` and `#[arg(long)] workspace: Option<String>`
   - Add `Commands::Snapshot { #[arg(long)] agent: bool, #[arg(long, value_enum)] format: Option<OutputFormat> }` where `OutputFormat` is an enum `Json | Toon`
   - Handle `Commands::Snapshot`: gather_snapshot, then if `agent` or format==Toon → toon::encode; if format==Json → serde_json; else → toon (agent default)
   - Pass `global` and `workspace` flags to `tui::run()` (update its signature) or store in a config struct

**Acceptance:**
- `cargo build` passes
- `soji snapshot --agent` prints something TOON-encoded (not a panic)
- `soji snapshot --format json` prints valid JSON
- `--global` and `--workspace` flags are accepted by clap

**Session Prompt:**
```
You are implementing CLI flags and TOON output for Soji M1 (Wave 1, Task 1.2).

Context files to read first:
- src/main.rs — current CLI structure
- src/domain/process.rs, src/domain/workspace.rs — types to serialize
- src/sources/mod.rs — gather_snapshot() return type
- docs/workspace/20260306-m1-tui/breadboard.md — VS2 wiring, output layer affordances
- docs/workspace/20260306-m1-tui/shaping.md — A4, A9 parts
- MEMORY.md — Agent Output Format Contract section for TOON spec

What to build:
1. Create src/output/mod.rs and src/output/toon.rs
2. TOON encoder: encode serde_json::Value to TOON string with Soji key abbreviation map
3. Add --global, --workspace flags to Cli
4. Add snapshot subcommand with --agent and --format flags
5. Wire snapshot subcommand: gather_snapshot → encode → print

Constraints:
- TOON encoder works on serde_json::Value (serialize domain types first with serde_json::to_value)
- Do NOT modify src/tui/ files (TUI gets --global/--workspace in a later integration step)
- Do NOT add TOON decoding (encoding only for M1)
- Graceful: if serde fails, emit an error message to stderr and exit 1
- Run `cargo build` after each step
```

### Task 1.3: Event Fixes and Focus Handling

**Topic:** `event-fixes`
**Depends on:** `1.1-app-state-modes` complete
**Complexity:** Medium
**Files:**
- `src/tui/events.rs` — fix k ambiguity, add Space/? /1-9/focus/g/KillBatch
- `src/tui/mod.rs` — add EnableFocusChange to terminal setup

**Steps:**

1. In `src/tui/mod.rs`, add `EnableFocusChange` to the terminal setup `execute!` call. Add `DisableFocusChange` to cleanup.

2. Rewrite `handle_key()` in `src/tui/events.rs`:
   - Add `AppEvent::KillBatch(Vec<u32>)` to the `AppEvent` enum
   - Fix k ambiguity: use explicit cursor context check. `Char('k')` arm: if cursor is on a `NavItem::Process(pid)`, route to kill flow. Navigation: `Up`/`Down` arrows always navigate. `Char('j')` always navigates down. Navigation via `k` is removed to eliminate ambiguity — use arrow keys only. (User confirmed arrow keys are primary navigation.)
   - Add `Char(' ')` arm: if cursor is on Process row, call `app.toggle_selection(pid)`
   - Add `Char('?')` arm: `app.show_help = !app.show_help`
   - Add `Char('g')` arm: `app.mode = AppMode::Global` (in Workspace mode)
   - Add `Char('1')..=Char('9')` arm: in Global mode, 1/2/3 → cycle tab via `jump_global_tab`; in Workspace mode, jump workspace by index (cycle `snapshot.workspaces`)
   - Kill flow for k: if `app.selected.len() > 1` → return `Some(AppEvent::KillBatch(app.selected.drain().collect()))`; if SharedInfra → infra_confirm flow (two k presses); else standard kill_confirm flow

3. Add focus event handling:
   - In `poll_events()`, also match `Event::FocusGained` and `Event::FocusLost`
   - FocusLost: `app.prev_mode = Some(app.mode.clone()); app.mode = AppMode::Scout`
   - FocusGained: `app.restore_mode()`

4. In `src/tui/mod.rs` run_loop, handle `AppEvent::KillBatch(pids)`:
   - Kill each pid: `actions::kill_processes(&proc_refs).await`
   - Refresh snapshot after batch kill

**Acceptance:**
- `cargo build` passes
- k key navigates up when on a non-process row (or only arrows navigate)
- k key triggers kill flow when on a process row
- Space toggles selection
- ? toggles help flag
- FocusLost transitions to Scout mode

**Session Prompt:**
```
You are fixing events and adding focus handling for Soji M1 (Wave 1, Task 1.3).

Context files to read first:
- src/tui/events.rs — current event handler (this is the primary file to modify)
- src/tui/mod.rs — terminal setup and run_loop
- src/tui/app.rs — App struct with new fields from Wave 1.1 (already merged)
- docs/workspace/20260306-m1-tui/breadboard.md — VS4/VS5 wiring, events code affordances
- docs/workspace/20260306-m1-tui/bb-reflection.md — Fix 3 (prev_mode edge case), Fix 4 (KillBatch)
- docs/workspace/20260306-m1-tui/shaping.md — A11, A12, A13 parts

What to build:
1. Add EnableFocusChange/DisableFocusChange to terminal setup in mod.rs
2. Add AppEvent::KillBatch(Vec<u32>) to AppEvent enum
3. Fix k key ambiguity — k navigates UP only when NOT on a Process row. k kills when ON a Process row.
4. Wire Space, ?, g, 1-9 keys
5. Add FocusGained/FocusLost handlers to poll_events
6. Handle AppEvent::KillBatch in run_loop (mod.rs)

Important design constraints:
- Enter MUST NEVER return AppEvent::KillConfirmed — it only expands/collapses
- k on a Surface row does nothing (no kill, no navigation)
- SharedInfra kill requires 3 k presses total (infra_confirm → kill_confirm → KillConfirmed)
- Navigation: arrow keys UP/DOWN are primary. Char('j') = down. Char('k') is ONLY kill, not navigation.
- Run `cargo build` to verify
```

---

## Wave 2: UI Rendering

### Task 2.1: Global Mode UI

**Topic:** `global-mode-ui`
**Depends on:** `1.1-app-state-modes` complete, `1.3-event-fixes` complete
**Complexity:** High
**Files:**
- `src/tui/ui/global.rs` — new file: entire Global Mode renderer
- `src/tui/ui/mod.rs` — dispatch to draw_global when mode == Global
- `src/tui/mod.rs` — pass AppMode to run() from main.rs (if not already done)

**Steps:**

1. Create `src/tui/ui/global.rs`:
   - `pub fn draw_global(f: &mut Frame, area: Rect, app: &App)`
   - Split area: tab bar (top 1 line) + content area (rest)
   - `draw_tab_bar()`: render "[ Workspaces ]  [ Ports ]  [ Services ]" with active tab highlighted
   - Split content area: left (list) + right (detail pane, same pattern as existing right.rs)
   - `draw_workspaces(f, area, app)`: list of workspaces with health signals
     - Each row: `● name  ⚠ 2 orphans  14 procs  234 MB` (or `○ inactive`)
     - Health signals derived from `snapshot.workspaces` + scope-classified processes
   - `draw_ports(f, area, app)`: port table grouped by scope tier
     - Headers: "WORKSPACE-BOUND", "SHARED INFRASTRUCTURE", "UNAFFILIATED"
     - Each port row: `:3000  kata (node)  18h  234 MB`
     - Source: iterate `snapshot.processes`, filter those with `metadata.ports` non-empty, group by scope
   - `draw_services(f, area, app)`: SharedInfra processes
     - Each row: `OrbStack  running  :5556  45 MB`
     - Source: `snapshot.processes.iter().filter(|p| matches!(p.scope, ProcessScope::SharedInfra {..}))`
   - Global right pane: same process/surface detail as existing `right.rs` — reuse `process_detail()` logic

2. In `src/tui/ui/mod.rs`, update `draw()`:
   ```rust
   pub fn draw(f: &mut Frame, app: &App) {
       match app.mode {
           AppMode::Global => ui::global::draw_global(f, f.area(), app),
           AppMode::Workspace => { /* existing layout */ },
           AppMode::Scout => ui::scout::draw_scout(f, f.area(), app),
       }
       if app.show_help {
           ui::help::draw_help_overlay(f, app);
       }
   }
   ```

**Acceptance:**
- `cargo build` passes
- Running `soji --global` renders Global Mode with tabs (even if data is mock/sparse)
- Tab bar shows correct active tab highlight
- Port Manager tab shows ports grouped by scope tier
- Services tab shows SharedInfra processes

**Session Prompt:**
```
You are implementing the Global Mode TUI renderer for Soji M1 (Wave 2, Task 2.1).

Context files to read first:
- src/tui/ui/mod.rs — current draw dispatch (add AppMode dispatch here)
- src/tui/ui/left.rs — existing left panel pattern (reference for list rendering)
- src/tui/ui/right.rs — existing right panel pattern (reuse process_detail logic)
- src/tui/app.rs — App struct with AppMode, GlobalTab, fields (already from Wave 1.1)
- src/domain/process.rs — ProcessScope enum (from Wave 0)
- docs/workspace/20260306-m1-tui/breadboard.md — Global Mode UI affordances, draw_global wiring
- docs/workspace/20260306-m1-tui/bb-reflection.md — Fix 1 (Global Mode detail pane required)

What to build:
1. Create src/tui/ui/global.rs with draw_global, draw_tab_bar, draw_workspaces, draw_ports, draw_services
2. Update src/tui/ui/mod.rs to dispatch on app.mode

Color palette reference (match existing style):
- BG: Rgb(26, 27, 38) — deep background
- PRI: Rgb(192, 202, 245) — primary text
- DIM: Rgb(86, 95, 137) — muted
- MUT: Rgb(59, 66, 97) — very muted
- BLU: Rgb(122, 162, 247) — blue accent
- GRN: Rgb(158, 206, 106) — green (healthy)
- YEL: Rgb(224, 175, 104) — yellow (warning)
- RED: Rgb(255, 100, 100) — red (error/orphan)

Constraints:
- Do NOT modify existing Workspace Mode rendering (left.rs, right.rs, header.rs, status.rs)
- Global Mode and Workspace Mode share the same color palette
- Process detail in Global right pane: reuse process_detail() from right.rs (make it pub or move to shared)
- Run `cargo build` to verify
```

### Task 2.2: Scout Mode UI

**Topic:** `scout-mode-ui`
**Depends on:** `1.1-app-state-modes` complete, `1.3-event-fixes` complete
**Complexity:** Low
**Files:**
- `src/tui/ui/scout.rs` — new file: Scout compact layout

**Steps:**

1. Create `src/tui/ui/scout.rs`:
   - `pub fn draw_scout(f: &mut Frame, area: Rect, app: &App)`
   - Two rows (or flexible height):
     - **Header row**: workspace name + health dot + process count + active ports + git branch
       - Example: `⬡ soji  ● 4 procs  :3000 :8080  main ✓`
     - **Body rows**: top N processes sorted by memory_mb descending, showing name + memory bar + MB
       - Compact format: `  claude-code  ████░░░░  312 MB`
   - Width-aware: Scout is designed for a narrow pane (~40–60 cols). Keep everything on one line.
   - If workspace is None (Global Mode was active before Scout), show system-level summary.

2. Color treatment in Scout: use same palette, but dimmer — Scout is ambient, not interactive. No selection highlighting, no kill confirmation UI.

**Acceptance:**
- `cargo build` passes
- Triggering FocusLost (can simulate with a test or manual cmux defocus) renders Scout layout
- Scout shows workspace name + process count in the header

**Session Prompt:**
```
You are implementing the Scout Mode compact layout for Soji M1 (Wave 2, Task 2.2).

Context files to read first:
- src/tui/ui/mod.rs — draw dispatch (already updated in Task 2.1)
- src/tui/app.rs — AppMode::Scout, App.workspace field
- src/domain/process.rs — Process, ProcessScope
- docs/workspace/20260306-m1-tui/breadboard.md — Scout Mode UI affordances, VS5 wiring
- docs/workspace/20260306-m1-tui/shaping.md — A8 part (Scout Mode description)
- MEMORY.md — Scout Mode design decisions (workspace header + fallback body)

What to build:
1. Create src/tui/ui/scout.rs with draw_scout()
2. Header: workspace name, health dot, process count, ports list, git branch
3. Body: top N (fit available height) processes by memory, compact format with memory bar

Constraints:
- Scout is display-only: no interactive elements, no cursor, no selection state rendered
- Designed for narrow panes (~40-60 char width): everything must fit on one line per row
- If app.workspace is None, show: "⬡ global  N workspaces  M processes"
- Memory bar: 8 chars wide using block chars ████░░░░ scaled to max_mem_mb
- Run `cargo build` to verify
```

### Task 2.3: Help Overlay and Selection Indicators

**Topic:** `help-overlay-ui`
**Depends on:** `1.1-app-state-modes` complete, `2.1-global-mode-ui` complete
**Complexity:** Low
**Files:**
- `src/tui/ui/help.rs` — new file: help overlay
- `src/tui/ui/left.rs` — add selection checkbox indicator to process rows

**Steps:**

1. Create `src/tui/ui/help.rs`:
   - `pub fn draw_help_overlay(f: &mut Frame, app: &App)` — renders centered box over current layout
   - Content: two-column keybinding table:
     ```
     Navigation          Actions
     ↑↓  navigate        k    kill (confirm)
     Tab switch panel    Space select
     1-9 workspace/tab   Esc  cancel
     ?   help            q    quit
     Enter expand        g    global mode
     ```
   - Use `ratatui::widgets::Clear` to clear the background area
   - Box: 60×20 centered, bordered, background fills with BG color

2. In `src/tui/ui/left.rs`, enhance process row rendering:
   - When `app.selected.contains(&pid)`, prepend a `[✓]` or filled square indicator to the row
   - When `app.kill_confirm == Some(pid)`, show highlight in yellow

**Acceptance:**
- `cargo build` passes
- `app.show_help = true` renders the overlay centered over whatever layout is underneath
- Selected processes in left panel show visual indicator

**Session Prompt:**
```
You are implementing the ? help overlay and selection indicators for Soji M1 (Wave 2, Task 2.3).

Context files to read first:
- src/tui/ui/mod.rs — draw dispatch (help overlay is rendered last, over everything)
- src/tui/ui/left.rs — existing process row rendering (add selection indicator here)
- src/tui/app.rs — app.show_help, app.selected fields
- docs/workspace/20260306-m1-tui/breadboard.md — HelpOverlay UI affordances, help.rs code affordances

What to build:
1. Create src/tui/ui/help.rs with draw_help_overlay()
   - Centered box, 60-wide or terminal_width/2, bordered
   - Use ratatui::widgets::Clear before rendering the box
   - Two-column keybinding table with all M1 keybindings
2. Enhance src/tui/ui/left.rs process rows:
   - Show [✓] or filled indicator when pid is in app.selected
   - Show yellow highlight on kill_confirm row

Constraints:
- Help overlay renders ON TOP of everything — call draw_help_overlay after all other draw calls in mod.rs
- Overlay must not crash if terminal is very small — handle min size gracefully
- Do NOT modify right.rs, header.rs, status.rs, global.rs, scout.rs
- Run `cargo build` to verify
```

---

## Wave 3: Quality and Docs

### Task 3.1: Tests

**Topic:** `tests`
**Depends on:** `0.1-domain-and-scope` complete, `1.3-event-fixes` complete
**Complexity:** Medium
**Files:**
- `src/sources/scope_tests.rs` (or `src/sources/scope.rs` inline `#[cfg(test)]`)
- `src/output/toon_tests.rs` (or inline in `src/output/toon.rs`)
- `src/tui/tests/tui_behavior.rs` (new `src/tui/tests/` module)

**Steps:**

1. Scope classifier tests (in `src/sources/scope.rs` `#[cfg(test)]` block):
   - Test WorkspaceBound: process CWD exactly matches workspace root → WorkspaceBound
   - Test WorkspaceBound: process CWD is nested under workspace root → WorkspaceBound
   - Test WorkspaceBound: worktree CWD (`workspace_root/.claude/worktrees/x/src/`) → WorkspaceBound for that workspace
   - Test SharedInfra by port: process with port 11434 → SharedInfra { service_name: "Ollama" }
   - Test SharedInfra by name: process with command "ollama serve" → SharedInfra
   - Test Unaffiliated: process with random CWD and no known port → Unaffiliated
   - Test multiple workspaces: process CWD matches second workspace, not first → correct WorkspaceBound

2. TOON encoder tests (in `src/output/toon.rs` `#[cfg(test)]` block):
   - Test encode empty object: `{}` → `{}`
   - Test encode simple process: `{pid: 123, command: "node"}` → `{p:n:123,cmd:s:node}`
   - Test encode array: `[1, 2, 3]` → `[n:1,n:2,n:3]`
   - Test encode bool and null: `{active: true, tty: null}` → `{active:b:true,tty:null}`
   - Test key abbreviation: `pid` → `p`, `memory_mb` → `m`, `workspace_name` → `w`
   - Test encode full EnvironmentSnapshot (round-trip: serde_json deserializable from serde_json::from_str of the encoded value — note: TOON is not JSON, so round-trip means TOON→JSON→domain type is stable, not that TOON decodes back. Verify the JSON intermediate is stable.)

3. TUI behavioral tests (in new test module, testing `handle_key()` directly):
   - Test: Enter on Surface row → toggle_expand() called, no KillConfirmed returned
   - Test: Enter on Process row → no KillConfirmed returned (open_detail only)
   - Test: k on Surface row → no kill state set
   - Test: k on Process row (non-SharedInfra) → kill_confirm set, no KillConfirmed
   - Test: k then k on Process row → KillConfirmed returned
   - Test: Space on Process row → pid added to selected
   - Test: Space again → pid removed from selected
   - Test: ? → show_help toggled true then false

**Acceptance:**
- `cargo test` passes with all tests green
- At minimum 8 scope tests, 6 TOON tests, 8 behavioral tests

**Session Prompt:**
```
You are writing tests for Soji M1 (Wave 3, Task 3.1).

Context files to read first:
- src/sources/scope.rs — classify_scope() function (Wave 0, already merged)
- src/output/toon.rs — encode() function (Wave 1.2, already merged)
- src/tui/events.rs — handle_key() function (Wave 1.3, already merged)
- src/tui/app.rs — App struct
- docs/workspace/20260306-m1-tui/shaping.md — A14, A15, A16 parts

What to write:
1. #[cfg(test)] block in src/sources/scope.rs: 8+ unit tests for classify_scope()
2. #[cfg(test)] block in src/output/toon.rs: 6+ tests for encode()
3. New test module at src/tui/behavior_tests.rs (or inline in events.rs): 8+ behavioral tests

Test construction tips:
- For classify_scope tests: construct minimal Process and Workspace values with just the fields needed
- For handle_key tests: construct App with App::new(snapshot, AppMode::Workspace), call handle_key(app, KeyEvent{...}), assert app state
- For KillConfirmed test: call handle_key twice with k on a Process row, assert second call returns Some(AppEvent::KillConfirmed(pid))

Constraints:
- Tests must be deterministic — no I/O, no async in scope tests
- TOON encoder tests: compare output strings exactly where the encoding is deterministic, or parse the result as JSON where ordering might vary
- Run `cargo test` to verify all pass
```

### Task 3.2: CI/CD

**Topic:** `ci-pipeline`
**Depends on:** `3.1-tests` complete
**Complexity:** Low
**Files:**
- `.github/workflows/ci.yml` — new file

**Steps:**

1. Create `.github/workflows/ci.yml`:
   ```yaml
   name: CI
   on:
     push:
       branches: [main]
     pull_request:
       branches: [main]
   jobs:
     check:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - uses: dtolnay/rust-toolchain@stable
         - uses: Swatinem/rust-cache@v2
         - name: fmt
           run: cargo fmt --check
         - name: clippy
           run: cargo clippy -- -D warnings
         - name: test
           run: cargo test
   ```

2. Verify: `cargo fmt --check` passes locally (fix any fmt issues in current code).
3. Verify: `cargo clippy -- -D warnings` passes locally (fix any warnings).

**Acceptance:**
- CI yml file exists at correct path
- `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo test` all pass locally

**Session Prompt:**
```
You are setting up CI/CD for Soji M1 (Wave 3, Task 3.2).

Context files to read first:
- CLAUDE.md — project commands
- .github/ directory (check if it exists, create if not)

What to build:
1. Create .github/workflows/ci.yml with fmt + clippy + test jobs
2. Run `cargo fmt --check` locally — if it fails, run `cargo fmt` then note which files changed
3. Run `cargo clippy -- -D warnings` locally — fix any warnings in your changes (do NOT touch files owned by other sessions)
4. Confirm `cargo test` passes

Constraints:
- Use ubuntu-latest, Rust stable, dtolnay/rust-toolchain@stable, Swatinem/rust-cache@v2
- Do NOT add coverage threshold yet (cargo-tarpaulin setup is complex; defer to a follow-up)
- Only fix clippy warnings in files you created in this session; flag other warnings to the user
```

### Task 3.3: M1 Design Docs

**Topic:** `m1-docs`
**Depends on:** None (can run in parallel from Wave 1 start)
**Complexity:** Low
**Files:**
- `docs/design/surface-types.md`
- `docs/design/scout-mode.md`
- `docs/design/scout-vs-inline.md`
- `docs/design/reference-layout.md`
- `docs/design/context-signals.md`
- `docs/design/display-modes.md`
- `docs/design/conceptual-model.md`

**Steps:**

1. Create `docs/design/` directory.

2. `surface-types.md` — Define the 7 surface types with description and Soji context signals:
   - Agent/LLM session (Claude Code, etc.)
   - Runner/log tail (CI output, npm run dev)
   - Passive monitor (btop, bottom)
   - Directory navigator (yazi)
   - Active work (editors, IDEs)
   - Quick terminal (ad-hoc commands)
   - Browser tab (Chrome, Arc)

3. `scout-mode.md` — Scout Mode design spec:
   - What Scout Mode is (ambient pane that watches while you work elsewhere)
   - FocusGained/FocusLost transition mechanics
   - Two-layer layout: workspace header + fallback body
   - What's in M1 vs M1.5 (surface-type-aware body deferred)
   - cmux setup guidance (one dedicated Soji pane in layout)

4. `scout-vs-inline.md` — Framework for choosing Scout vs Inline:
   - Scout: workspace-level, always-visible dedicated pane, interactive on focus
   - Inline: surface-specific, complements P10k status line, never full-screen
   - When to use each, what data belongs where

5. `reference-layout.md` — Reference cmux layout spec:
   - Three-column reference layout with tab groups
   - Column 1: Agent session + Quick terminal below
   - Column 2: Directory navigator (yazi), tall skinny
   - Column 3: Soji Scout pane + lazygit tab + runner/logs tab
   - cmux layout file pseudocode

6. `context-signals.md` — Inventory of signals Soji reads per surface type:
   - What each surface type exposes: git branch, CWD, ports, process memory, model, session name
   - Which signals are in M1 vs later

7. `display-modes.md` — Mode definitions:
   - Global Mode: what it shows, when it's active, navigation
   - Workspace Mode: what it shows, when it's active, navigation
   - Scout Mode: what it shows, transitions

8. `conceptual-model.md` — Hierarchy: Org → Project → Workspace → Layout → Surface → Process. How these relate to cmux, Soji's domain model, and soji.toml (future).

**Acceptance:**
- All 7 files exist in `docs/design/`
- Each is at least 200 words with concrete specifics (not just outlines)

**Session Prompt:**
```
You are writing the M1 design documentation for Soji (Wave 3, Task 3.3).

Context files to read first:
- docs/vision.md — Soji's identity and three pillars
- docs/user-stories.md — the journeys driving design decisions
- MEMORY.md — all locked design decisions for M1 (Scout Mode, dual-mode launch, keybindings, etc.)
- docs/workspace/20260306-m1-tui/shaping.md — A18 part (full list of docs to create)
- docs/workspace/20260306-m1-tui/frame.md — problem and outcome context

What to write:
1. docs/design/surface-types.md — 7 surface type taxonomy
2. docs/design/scout-mode.md — Scout Mode design spec
3. docs/design/scout-vs-inline.md — decision framework
4. docs/design/reference-layout.md — cmux layout spec with tabs
5. docs/design/context-signals.md — signals inventory per surface type
6. docs/design/display-modes.md — Global/Workspace/Scout mode definitions
7. docs/design/conceptual-model.md — project/workspace/layout/surface/process hierarchy

Constraints:
- Write docs for future builders, not for the current session
- Be specific: include concrete examples, keybindings, data fields, not just intent
- Scout Mode is "Scout Mode" (codename) — not "Focus Mode"
- Inline mode is deferred to M1.5 — mention it but mark as future
- Do NOT modify any .rs files
```

---

## Session Sizing

| Wave | Session | Complexity | Files | Can parallelize with |
|------|---------|------------|-------|---------------------|
| W0 | 0.1 domain-and-scope | Medium | 4 | — |
| W1 | 1.1 app-state-modes | Medium | 1 | 1.2 |
| W1 | 1.2 cli-and-snapshot | Medium | 3 | 1.1 |
| W1 | 1.3 event-fixes | Medium | 2 | 1.2 (not 1.1) |
| W2 | 2.1 global-mode-ui | High | 2 | 2.2, 2.3 |
| W2 | 2.2 scout-mode-ui | Low | 1 | 2.1, 2.3 |
| W2 | 2.3 help-overlay-ui | Low | 2 | 2.1, 2.2 |
| W3 | 3.1 tests | Medium | 3 | 3.2, 3.3 |
| W3 | 3.2 ci-pipeline | Low | 1 | 3.3 (not 3.1) |
| W3 | 3.3 m1-docs | Low | 7 | 3.1, 3.2 |

---

## Dependency DAG

```
0.1 domain-and-scope
    ├──► 1.1 app-state-modes
    │        └──► 1.3 event-fixes
    │                  ├──► 2.1 global-mode-ui
    │                  ├──► 2.2 scout-mode-ui
    │                  └──► (2.3 help-overlay-ui waits for 2.1)
    └──► 1.2 cli-and-snapshot
              └──► (no wave 2 dependency)

Wave 3:
    3.1 tests ──► [depends on 0.1, 1.2, 1.3]
    3.2 ci ──► [depends on 3.1]
    3.3 m1-docs ──► [no code dependencies, can start at Wave 1]
```

---

## Critical Path

```
W0:0.1 → W1:1.1 → W1:1.3 → W2:2.1 → W2:2.3 → W3:3.1 → W3:3.2
```
7 serial steps. Parallel sessions (1.2, 2.2, 3.3) are off the critical path.

---

## File Ownership

All tasks write to the same working directory. No two parallel tasks touch the same file. File ownership is the coordination mechanism — no merging needed within the branch.

**File ownership (no two parallel tasks write the same file):**

| File | Owner |
|------|-------|
| `src/domain/process.rs` | W0:0.1 only |
| `src/sources/scope.rs` | W0:0.1 only |
| `src/sources/command.rs` | W0:0.1 only |
| `src/sources/mod.rs` | W0:0.1 only |
| `src/tui/app.rs` | W1:1.1 only |
| `src/main.rs` | W1:1.2 only |
| `src/output/toon.rs` | W1:1.2 only |
| `src/tui/events.rs` | W1:1.3 only |
| `src/tui/mod.rs` | W1:1.3 (EnableFocusChange) + W1:1.2 (run() signature) — potential conflict |
| `src/tui/ui/mod.rs` | W2:2.1 only |
| `src/tui/ui/global.rs` | W2:2.1 only |
| `src/tui/ui/scout.rs` | W2:2.2 only |
| `src/tui/ui/help.rs` | W2:2.3 only |
| `src/tui/ui/left.rs` | W2:2.3 only |
| Test files | W3:3.1 only |
| `.github/workflows/ci.yml` | W3:3.2 only |
| `docs/design/*.md` | W3:3.3 only |

**Conflict point:** `src/tui/mod.rs` is touched by both 1.2 (run() signature needs global/workspace params) and 1.3 (EnableFocusChange). These tasks run as parallel subagents. Resolution: serialize them — run 1.2 first (signature change), then 1.3 adds EnableFocusChange/DisableFocusChange. The orchestrator must wait for 1.2 to finish before spawning 1.3 for mod.rs changes only.

**Conflict point:** `src/tui/ui/left.rs` is touched by 2.3 (selection indicators). It is NOT touched by 2.1 or 2.2 — no conflict.

---

## Notes

- **Wave 0 is the critical foundation** — nothing else can start until `scope` field exists on `Process`. Don't rush it; make sure `classify_scope()` has good defaults and graceful error handling.
- **1.1 and 1.2 run in parallel** but both may need to update `mod.rs` to wire `run(global, workspace)`. Coordinate the signature change: 1.2 defines the signature, 1.3 uses it. 1.1 doesn't touch mod.rs.
- **3.3 m1-docs is independent** — can be started as soon as Wave 0 is merged. Good task to run in parallel while wave 1/2 code sessions are in progress.
- **Global Mode detail pane**: Wave 2.1 should reuse `process_detail()` from `right.rs` — make that function `pub` in right.rs (small change in right.rs owned by 2.1 if needed, or expose via re-export).
- **TOON encoder** is a new Rust module with no external dependencies — straightforward to implement and test independently.
- **k navigation removal**: The fix in 1.3 removes `k` as a navigation key entirely (only arrow keys + j navigate). This is a UX change the user accepted — document in Scout Mode spec.

---
shaping: true
pipeline: 20260306-m1-tui
stage: shaping
shape_selected: A
---

# Shaping: Soji M1 TUI

## Requirements

| Req | Requirement | Status |
|-----|-------------|--------|
| R0 | Dual-mode launch — Soji detects CWD context and opens Global Mode (outside workspace) or Workspace Mode (inside workspace). `--global` and `--workspace <name>` override. | Must-have |
| R1 | Global Mode: three tabs (Workspaces / Ports / Services), Tab-key navigation, workspace summary list with health signals. | Must-have |
| R2 | Port Manager (Global Mode Ports tab): shows all listening ports grouped by ProcessScope tier (WorkspaceBound / SharedInfra / Unaffiliated). | Must-have |
| R3 | ProcessScope model: every `Process` struct carries `scope: ProcessScope` (WorkspaceBound { workspace_name } / SharedInfra { service_name } / Unaffiliated). Computed once at gather time by pure classifier. | Must-have |
| R4 | Multi-select kill: `Space` toggles process selection, `k` kills all selected, SharedInfra processes require extra confirmation. | Must-have |
| R5 | Scout Mode foundation: FocusGained/FocusLost events via crossterm `EnableFocusChange`. Compact ambient layout on FocusLost. Full interactive TUI on FocusGained. | Must-have |
| R6 | Agent output: `soji snapshot --agent` emits TOON. `--format json` and `--format toon` explicit overrides. | Must-have |
| R7 | TOON encoder: scoped Rust implementation in `src/output/toon.rs`. Not JSON. Lossless. Covers EnvironmentSnapshot type. | Must-have |
| R8 | Full command resolution: batch `ps -p <pids> -o pid=,args=` fixes truncation for all tracked processes. | Must-have |
| R9 | Bug fixes: resolve k key ambiguity (events.rs:41/67), wire `Space` multi-select, wire `?` help overlay, wire `1–9` workspace switching. | Must-have |
| R10 | Tests: unit tests for ProcessScope classifier and TOON encoder. Behavioral tests for TUI keybinding contracts and mode switching. | Must-have |
| R11 | CI/CD: GitHub Actions on push/PR — `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo test`. Coverage threshold (>70%). | Must-have |
| R12 | M1 docs deliverable: surface type taxonomy, Scout Mode spec, Scout vs Inline framework, reference cmux layout spec, context signals inventory, display mode definitions, conceptual model. | Must-have |
| R13 | Workspace Mode Ports section shows workspace-scoped ports only (not all tiers). | Must-have |
| R14 | `Enter` never triggers destructive action. Kill confirmation requires k+k. SharedInfra requires extra k. | Must-have |
| R15 | Graceful degradation: Soji works when cmux / Docker / ks are not installed. Missing sources emit empty data, not panics. | Must-have |

---

## Shapes

### Shape A: Layered build on existing TUI scaffold *(selected)*

Build M1 as a layered series of changes on the existing ratatui/crossterm/tokio scaffold. No architectural rewrites. Each layer adds to what exists rather than replacing it.

**Why selected:** The existing TUI scaffold is architecturally sound. `App`, `events`, `ui/*`, `sources`, `domain`, `actions` separation is clean. Dual-mode launch is an extension of `detect_workspace()`, not a rewrite. Global Mode adds new layouts alongside existing Workspace Mode layout. ProcessScope adds a field + classifier. Scout Mode adds a focus event handler + new layout branch. TOON is a new output module. All changes are additive.

**Discarded alternatives:**

- **Shape B: Rewrite around a state machine** — App state as an explicit state machine enum with Global/Workspace/Scout variants as states. Cleaner in theory but risky (rewrites event handling, layout dispatch, test surface grows). Deferred to M2 if complexity warrants.

- **Shape C: Daemon + TUI client** — Background daemon gathers data; TUI is a thin client. Much richer history support but deployment complexity is incompatible with M1 appetite. Deferred to M5.

---

## Shape A: Parts

| Part | Mechanism | Flag |
|------|-----------|:----:|
| **A1** | Add `scope: ProcessScope` to `Process` struct in `src/domain/process.rs`. Add `ProcessScope` enum: `WorkspaceBound { workspace_name: String }`, `SharedInfra { service_name: String }`, `Unaffiliated`. Derive `Serialize, Deserialize, Clone, Debug`. | |
| **A2** | Add `classify_scope(process: &Process, workspaces: &[Workspace]) -> ProcessScope` pure function in `src/sources/scope.rs`. Logic: CWD prefix match → WorkspaceBound; port heuristics + name table → SharedInfra; else → Unaffiliated. Call at end of `gather_snapshot()` to set `scope` on each process. Shared infra name table is a static list (OrbStack, Ollama, Docker daemon, local DBs). | |
| **A3** | Add `AppMode` enum (`Global`, `Workspace`, `Scout`) to `src/tui/app.rs`. `App::new()` calls `detect_mode()` which checks CWD (existing `detect_workspace()` logic extended). Global and Scout are new modes. `run()` in `src/tui/mod.rs` selects initial layout based on mode. | |
| **A4** | Add `--global` and `--workspace <name>` flags to `Cli` in `src/main.rs`. Add `Snapshot` subcommand with `--agent` flag and `--format` (json/toon) option. Wire flag overrides into `detect_mode()`. | |
| **A5** | Add `GlobalTab` enum (`Workspaces`, `Ports`, `Services`) to `src/tui/app.rs`. Add `active_tab: GlobalTab` field to `App`. Wire `Tab` key to cycle tabs. Add `tab_index: usize` for `1–9` switching (1=workspace, 2=ports, 3=services within Global Mode). | |
| **A6** | Add `src/tui/ui/global.rs`: Global Mode layout renderer. Three tab panels: workspace list (health signals — active dot, orphan warning, process count, memory), port manager (all ports, scoped by ProcessScope tier), services (SharedInfra processes). | |
| **A7** | Add `selected: HashSet<u32>` to `App` struct. Wire `Space` key: toggle current process into/out of `selected`. Update kill flow: if `selected` is non-empty, `k` kills all selected; if empty, kills cursor process. SharedInfra confirmation: two additional `k` presses (or explicit prompt). Visual indicator in left panel rows (checkbox or highlight). | |
| **A8** | Add `EnableFocusChange` to terminal setup in `src/tui/mod.rs`. Handle `Event::FocusGained` and `Event::FocusLost` in `src/tui/events.rs` — transition `app.mode` between `Scout` and `Workspace`/`Global`. Add Scout layout in `src/tui/ui/scout.rs`: workspace header (name, health, count, ports, git branch) + fallback body (top processes + memory bars). | |
| **A9** | Add `src/output/toon.rs`: Rust TOON encoder. Public API: `encode(value: &impl Serialize) -> String`. Implements TOON spec v3.0 encoding: type-prefix scalars, compact arrays, key abbreviation map for known Soji types (pid→p, memory_mb→m, workspace_name→w, etc.). Round-trip test: `decode(encode(v)) == v`. | |
| **A10** | Add `src/sources/command.rs`: batch command resolver. `resolve_commands(pids: &[u32]) -> HashMap<u32, String>` runs `ps -p <pids joined by comma> -o pid=,args=` once, parses output, returns full command strings by PID. Call in `gather_snapshot()` after building process list. | |
| **A11** | Fix `src/tui/events.rs`: resolve k key ambiguity by separating navigation from kill. `KeyCode::Up` and `KeyCode::Char('j')`/`KeyCode::Char('k')` for navigation only when no kill_confirm active OR explicitly without kill intent. Kill: `KeyCode::Char('k')` as its own arm, checked first, fires only when cursor is on a Process row. Separate navigation key `k` behind vim mode or alias with arrow keys only. | |
| **A12** | Add `?` key handler: set `app.show_help = true`. Add `src/tui/ui/help.rs`: overlay rendered on top of current layout showing all keybindings. `Esc` or `?` again closes. | |
| **A13** | Wire `1–9` keys: in Workspace Mode, jump to workspace by index (cycles through `snapshot.workspaces`). In Global Mode `1`=Workspaces tab, `2`=Ports tab, `3`=Services tab; `4–9` jump to workspace N in the workspaces list. | |
| **A14** | Add `src/tests/scope_tests.rs`: unit tests for `classify_scope()` — WorkspaceBound by CWD prefix, SharedInfra by port + name heuristic, Unaffiliated for everything else, worktree CWD resolves to parent workspace. | |
| **A15** | Add `src/tests/toon_tests.rs`: unit tests for TOON encoder — round-trip for EnvironmentSnapshot, empty snapshot, single process, unicode strings, special characters. | |
| **A16** | Add `src/tests/tui_behavior.rs`: behavioral tests for keybinding contracts — `Enter` never returns `KillConfirmed`, `k` on Surface row does not kill, `Space` toggles selection, `?` opens help, `Esc` clears state. | |
| **A17** | Add `.github/workflows/ci.yml`: GitHub Actions workflow — triggers on push + PR to main. Jobs: `fmt` (`cargo fmt --check`), `lint` (`cargo clippy -- -D warnings`), `test` (`cargo test`). Optional: coverage via `cargo-tarpaulin` with 70% threshold. | |
| **A18** | Create `docs/design/` directory with: `surface-types.md` (taxonomy of 7 surface types), `scout-mode.md` (Scout Mode design spec), `scout-vs-inline.md` (decision framework), `reference-layout.md` (cmux layout spec with tabs), `context-signals.md` (inventory of signals Soji reads per surface type), `display-modes.md` (Global/Workspace/Scout mode definitions), `conceptual-model.md` (project/workspace/layout/surface/process hierarchy). | |

---

## Fit Check

| Req | Requirement | Status | A |
|-----|-------------|--------|---|
| R0 | Dual-mode launch (CWD detection, `--global`, `--workspace`) | Must-have | ✅ |
| R1 | Global Mode: three tabs, workspace health list | Must-have | ✅ |
| R2 | Port Manager: all ports by ProcessScope tier | Must-have | ✅ |
| R3 | ProcessScope field on Process, computed at gather time | Must-have | ✅ |
| R4 | Multi-select kill with SharedInfra extra confirm | Must-have | ✅ |
| R5 | Scout Mode: focus events, compact layout, mode switch | Must-have | ✅ |
| R6 | Agent output: `soji snapshot --agent` emits TOON | Must-have | ✅ |
| R7 | TOON encoder: Rust implementation, lossless | Must-have | ✅ |
| R8 | Full command resolution via batch ps | Must-have | ✅ |
| R9 | Bug fixes: k ambiguity, Space, ?, 1–9 | Must-have | ✅ |
| R10 | Tests: unit + behavioral | Must-have | ✅ |
| R11 | CI/CD: GitHub Actions | Must-have | ✅ |
| R12 | M1 docs deliverable | Must-have | ✅ |
| R13 | Workspace Mode Ports: workspace-scoped only | Must-have | ✅ |
| R14 | Kill safety: Enter never destructive, k+k confirm, SharedInfra extra | Must-have | ✅ |
| R15 | Graceful degradation when tools missing | Must-have | ✅ |

Shape A passes all requirements. No flagged unknowns (⚠️). All mechanisms are understood.

---

## Decision Log

| Decision | Rationale |
|----------|-----------|
| Shape A (layered build, no rewrite) | Existing scaffold is sound. Additive changes have lower regression risk. |
| ProcessScope as field on Process (Option A) | Scope must be canonical for agent snapshots and CLI kill safety gate. Can't be TUI-only. |
| TOON as agent default, always | Benchmarks show TOON is always ≤ pretty JSON token count. No threshold needed. |
| Scout Mode codename (not "Focus Mode") | "Focus Mode" conflicts with Ghostty pane zoom feature. Scout is unambiguous. |
| Scout Mode in M1 foundation only | Surface-type awareness (dynamic surface body) requires cmux focus hooks — deferred to M1.5. M1 ships fallback body. |
| Inline TUI mode → M1.5 | Inline mode targets different workflow than Scout (P10k complement vs dedicated pane). Lower priority for M1. |
| cmux automation research → M1.5 | cmux automation capabilities not yet confirmed. Scout Mode works without it for M1. |
| soji doctor → M1.5 | Not blocking M1 journeys. Lower priority. |
| Worktree CWD matching | Any CWD descending from workspace root counts as workspace-bound. git worktree list handles parallel worktrees. |
| SharedInfra extra confirmation | OrbStack, Ollama, Docker daemon are shared infrastructure — accidental kill is high impact. Extra `k` required. |

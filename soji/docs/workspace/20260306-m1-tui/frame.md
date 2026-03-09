---
pipeline: 20260306-m1-tui
stage: frame
---

# Frame: Soji M1 TUI

## Source Material

An extensive discovery and alignment session (2026-03-06) produced locked design decisions covering all major M1 features. Decisions were made collaboratively with the project owner. All open questions were resolved. Source documents:

- `docs/user-stories.md` — 10 journeys + smaller stories. Key M1 journeys: J1 (morning startup), J3 (port conflict), J4 (end of day cleanup), J5 (memory), J8 (unknown process), J10 (shared infrastructure).
- `docs/open-questions.md` — All critical/high-priority questions resolved: dual-mode launch (Q1), workspace health signals (Q5), Port Manager relationship (Q6), unaffiliated process visibility (Q7).
- `docs/vision.md` — Three pillars: Observability, Maintenance, Organization. Soji as orchestration wrapper.
- `~/.claude/projects/-Users-cmbays-Github-soji/memory/MEMORY.md` — Locked design decisions, TUI keybinding contracts, ProcessScope model, agent output format contract, Scout Mode spec, dual-mode launch spec.

### What is already built (pre-M1)

The TUI shell exists but is incomplete. Current state:

- **Workspace Mode only** — the TUI hardcodes detection to a single workspace. No Global Mode.
- **Left panel** — navigable list with surfaces, process rows, section headers (Env/Secrets/Git). Working.
- **Right panel** — detail pane for surfaces, processes, sections. Working.
- **Header** — shows workspace name, `← global ? q` hints. Global Mode not implemented.
- **Status bar** — shows keybinding hints. Advertises `Space select` which is NOT implemented.
- **Kill flow** — single process kill with k+k confirm. Working. But has a k key ambiguity bug (events.rs:41 — `Up | k` arm consumes `k` before the kill arm at line 67).
- **5-second background refresh** — working.
- **RAII terminal cleanup** — working.
- **Missing**: Global Mode, Port Manager tab, Services tab, multi-select (Space), Scout Mode, agent output (TOON), ProcessScope field, `?` help overlay, `1–9` workspace switching, CI/CD, tests, full command resolution.

---

## Problem Statement

Soji M1 is partially built. The core TUI scaffold exists (Workspace Mode, ratatui, crossterm), but the milestone is not shippable. Missing pieces span every layer:

1. **Product completeness** — Global Mode doesn't exist. The Port Manager tab doesn't exist. A developer opening `soji` from outside a workspace gets partial/wrong behavior. Journeys 1, 3, and 10 are entirely unserved.

2. **Domain model gap** — `Process` has no `scope` field. Every process is display-only; there is no machine-readable answer to "is this workspace-bound or shared infrastructure?" This blocks kill safety (can't require extra confirm for SharedInfra), agent snapshots (can't tell agents which processes belong to which workspace), and Port Manager scoped filtering.

3. **Interaction bugs** — The `k` key is ambiguously handled — the `Up | k` arm on line 41 consumes `k` before the kill arm on line 67 can fire. `Space` is advertised but does nothing. `?` help does nothing. `1–9` workspace switching does nothing.

4. **Agent output missing** — `soji snapshot --agent` doesn't exist. There is no TOON encoder. Agents cannot query Soji for a token-efficient workspace snapshot.

5. **Scout Mode missing** — No focus event handling. No compact ambient surface mode. Soji cannot live as a dedicated cmux pane that transitions between ambient and interactive.

6. **No quality gate** — No CI/CD, no tests. Regressions have no safety net.

---

## Outcome Definition

A shippable M1 that:

1. **Dual-mode launch** — `soji` from inside a workspace CWD opens Workspace Mode. From outside opens Global Mode. `--global` and `--workspace <name>` overrides work.

2. **Global Mode** — Three tabs (Workspaces / Ports / Services) with Tab key switching. Workspace list shows name, health signals (active/inactive, orphan indicator, process count). Port Manager shows all ports across all scope tiers. Services tab shows shared infrastructure.

3. **Workspace Mode** — Left-panel collapsible sections (Processes, Ports, Env/Secrets/Git). Right detail pane. Already partially built — needs to receive ProcessScope data and be enhanced with multi-select.

4. **ProcessScope model** — Every `Process` has a `scope: ProcessScope` field (WorkspaceBound / SharedInfra / Unaffiliated). Computed once at gather time by a pure `classify_scope()` function in sources. SharedInfra processes show extra confirmation in kill flow.

5. **Multi-select kill** — `Space` toggles process selection into `HashSet<u32>` in App state. `k` on a multi-select kills all selected. Visual selection indicators in left panel.

6. **Scout Mode foundation** — crossterm `EnableFocusChange` enabled. `FocusGained`/`FocusLost` events handled. App enters Scout (compact) mode on FocusLost, Interactive mode on FocusGained. Scout layout shows workspace header (name, health, process count, ports, git branch) + fallback body (workspace processes + ports + memory bars).

7. **Agent output** — `soji snapshot --agent` subcommand emits TOON-encoded workspace snapshot to stdout. `--format json` emits JSON. `--format toon` emits TOON explicitly. A scoped Rust TOON encoder (`toon.rs`) handles encoding (not JSON pretending to be TOON).

8. **Command truncation fix** — Batch `ps -p <pids> -o pid=,args=` resolves full commands for all tracked processes. No more truncated display names.

9. **Tests + CI** — Unit tests for ProcessScope classifier and TOON encoder. Behavioral tests for TUI keybinding contracts and mode switching. GitHub Actions CI: `cargo fmt`, `cargo clippy -D warnings`, `cargo test`, coverage threshold.

10. **Bug fixes** — k key ambiguity resolved. `?` help overlay implemented. `1–9` workspace switching implemented. `Space` multi-select wired up.

11. **M1 docs deliverable** — `docs/design/` contains: surface type taxonomy, Scout Mode design spec, Scout vs Inline decision framework, reference cmux layout spec, context signals inventory, display mode definitions, conceptual model (project/workspace/layout/surface).

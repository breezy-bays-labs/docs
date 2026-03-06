---
title: Vision
description: 'The identity, principles, and desired end state for Soji — why it exists and where it is going.'
category: canonical
status: draft
last_updated: 2026-03-04
depends_on:
  - introduction
---

# Vision

> Living document. Updated as Soji evolves and our understanding of workspace management deepens.

## Identity

Soji is **meta-tooling for the agentic development workflow**. It manages the development environment itself — the workspaces, processes, services, and resources that accumulate as you build.

It is an **orchestration wrapper** — not a replacement for any tool. Soji provides a workspace-centric lens over existing dev tooling (cmux, git, docker, Claude, MCP servers, cloud services) and gives you observability, cleanup, and lifecycle management through that lens.

### Connection to Org Philosophy

Soji is a direct expression of several [core beliefs](/breezy-bays-labs/identity):

- **Structure IS Memory** — Agents have amnesia, so workspace definitions become the structural memory for environment state. If the workspace isn't defined, it can't be reproduced.
- **Automate Then Elevate** — Soji automates the repeatable parts of workspace management (cleanup, lifecycle, observation) so human attention can focus on higher-leverage decisions.
- **Elegance Through Recursion** — Soji observes itself. As a tool that monitors dev tooling, it should be aware of its own resource usage and workspace footprint.

## Core Abstraction

**Workspace = repo + everything that serves its development.**

A workspace encompasses:

| Layer | Examples |
|-------|----------|
| **Code** | Git repo, branches, worktrees |
| **Terminal** | cmux workspace, surfaces (terminal + browser tabs) |
| **Sessions** | Claude sessions, agents, MCP servers |
| **Services** | Dev servers, Docker containers, databases |
| **Environment** | Env vars, secrets (via direnv/ks), config files |
| **Cloud** | Vercel deployments, Supabase projects, Upstash instances |
| **Tools** | LazyGit, btop, other TUIs active in the workspace |

Everything in Soji's model maps to a workspace. Processes that don't map to any workspace are candidates for cleanup.

## Three Pillars

### Pillar 1: Observability

**See your workspace state clearly.**

- What's running across all workspaces? Within a single workspace?
- How much memory and CPU are your dev tools consuming?
- Which Claude sessions are active, what are they working on, and where?
- What services are listening on which ports?
- What env vars are loaded per workspace?
- What cloud assets (deployments, databases) are associated with each workspace?

Observability is the foundation — you can't maintain or organize what you can't see.

#### Key Capabilities
- Live-refreshing TUI dashboard grouped by workspace
- Workspace-scoped view (filter to a single workspace)
- Process-to-workspace mapping (match PIDs to cmux surfaces)
- Resource usage tracking (memory, ports, containers)
- Environment snapshot diffing ("what changed since I last checked")

#### Port Manager

A dedicated port management panel — the workspace-aware answer to tools like Port Assassin. Shows all listening ports across the system, not just known dev servers.

Ports are classified into three tiers:

| Scope | What it is | Detection |
|-------|-----------|-----------|
| **Workspace-bound** | Any process whose CWD is at or under a workspace root (including worktrees at any depth) | CWD prefix match against known workspace roots |
| **Shared infrastructure** | Services that intentionally serve multiple workspaces — OrbStack, Docker daemon, local DBs, Ollama, etc. | Port heuristics + process name; expandable via `~/.config/soji/services.toml` |
| **Unaffiliated** | Everything else — listening but not attributable | Catch-all; visible in Port Manager, hidden from workspace views |

Every port entry shows the full spawning command (resolved via `ps -p PID -o command=`), not just the truncated process name that `lsof` provides. This is the key enrichment over raw `lsof` output.

The Workspace View shows only workspace-bound ports, inline under their owning workspace. The Port Manager panel shows everything, grouped by tier. "Dev server" is not a separate category — if a `node`/`bun`/`vite` process's CWD maps to a workspace, it's workspace-bound; otherwise it's unaffiliated.

#### Workspace Observability

A workspace in Soji exposes its full environment, not just its running processes:

| Layer | What Soji shows |
|-------|----------------|
| **Processes** | PIDs, ports, memory, full command, status |
| **Environment vars** | Variable names and source (direnv, `.env`, `ks`, shell) — values never stored or displayed |
| **Secrets** | `ks`-managed secret names + provider + loaded status; macOS Keychain entries visible by name |
| **Config files** | Presence of `docker-compose.yml`, `.envrc`, `soji.toml`, `.env.*` — not contents |
| **Git state** | Current branch, worktrees, dirty status |
| **cmux state** | Active surfaces, pane layout |

Secrets are shown by name only. Variable names matching common secret patterns (`TOKEN`, `KEY`, `SECRET`, `PASSWORD`) are flagged as sensitive regardless of source. The goal: you can see that `GITHUB_TOKEN` is loaded from direnv and confirmed by `ks`, without ever seeing its value.

### Pillar 2: Maintenance

**Keep the environment clean.**

- Kill orphaned Claude sessions and agents
- Prune stale git branches and orphaned worktrees
- Clean up stopped Docker containers, dangling images, unused volumes
- Reclaim memory from detached and zombie processes
- Identify and flag resource creep over time

Maintenance is reactive — it addresses entropy that has already accumulated. The goal is to make cleanup fast, safe, and satisfying.

#### Key Capabilities
- Interactive TUI cleanup with multi-select process table and preview
- Individual row kill (`k`) or batch kill across selections — never a single destructive action on `Enter`
- Smart orphan detection (agent parent liveness, CWD matching, staleness)
- Git branch and worktree pruning with safety checks
- Docker pruning with size estimates
- Cleanup summaries ("freed 2.4 GB, killed 7 processes")

### Pillar 3: Organization

**Define, automate, and enforce workspace structure.**

- Declarative workspace definitions — what a workspace *should* look like
- Lifecycle automation — `soji up` / `soji down` to spin workspaces up and tear them down
- Templates — create new workspaces from proven patterns
- Drift detection — flag when running state diverges from declared state
- Cross-repo organization — bird's-eye view across all workspaces

Organization is proactive — it prevents entropy rather than reacting to it. This is the most ambitious pillar and the one most likely to evolve as our understanding deepens.

#### Key Capabilities
- Canonical workspace model (TOML definitions)
- Workspace lifecycle commands (`soji up <workspace>`, `soji down <workspace>`)
- Workspace templates for common project types
- Drift detection (declared state vs running state)
- Cross-repo dashboard with health indicators

## Design Principles

### Orchestrate, Don't Rebuild
Soji calls existing tools — it never reimplements them. Git stays git. Docker stays docker. cmux stays cmux. Soji's value is the workspace-centric perspective that unifies them, not replacing their functionality.

### Workspace-Centric Lens
Everything is viewed through the workspace abstraction. A process isn't just PID 12345 using 200MB — it's "the Claude session in the kata workspace working on the TUI feature." Context transforms raw data into actionable information.

### Graceful Degradation
If cmux isn't installed, Soji still works — it just doesn't have workspace grouping. If Docker isn't running, Docker sections are absent. Every data source is optional. The tool remains useful in reduced environments.

### Progressive Disclosure
The default view shows what matters most. Details are available on demand. `soji` gives you the dashboard. `--full` gives you everything. The TUI lets you drill in. You're never overwhelmed.

In the TUI, progressive disclosure is a split-pane pattern: the left pane is always a navigable table; `Enter` on any row opens a rich detail pane on the right without leaving the current context. For a port, the detail pane shows full command, CWD, memory, uptime, loaded env vars, and (where available) a log stream. For a workspace, it shows all of the above plus secrets, config file inventory, and git state. The detail pane closes with `Esc` — `Enter` always means "tell me more," never "take an action."

### Safety Over Speed
Cleanup operations always confirm before acting (unless `--force`). Branch deletion is conservative. The cost of accidentally killing the wrong process far exceeds the cost of an extra confirmation prompt.

## Desired End State

When Soji reaches maturity, the daily workflow looks like:

1. **Start of day**: `soji up kata` — workspace spins up with all the right tabs, services, and environment
2. **During work**: Soji TUI open in a workspace tab, showing live status. Quick glance confirms everything is healthy.
3. **Context switch**: `soji up spp` — second workspace comes alive. First stays running but monitored.
4. **End of day**: `soji down --all` — everything tears down cleanly. Memory reclaimed. No orphans.
5. **Weekly review**: `soji diff --weekly` — see how the dev environment evolved. What grew, what was cleaned up, what drifted.

The human never has to manually hunt for zombie processes, wonder what's consuming memory, or set up a development environment from scratch. The structure is defined, the lifecycle is automated, the state is observable.

## Growth Areas

> Honest about what's not yet figured out.

- **Where does Soji run?** — CLI, TUI, daemon, or hybrid? The runtime model has architectural implications we haven't fully resolved. See [issue #4](https://github.com/cmbays/soji/issues/4).
- **cmux automation surface** — We don't yet know what cmux can do programmatically for workspace creation. This is a critical dependency for the organization pillar. See [issue #2](https://github.com/cmbays/soji/issues/2).
- **Cross-repo orchestration boundaries** — Where does Soji end and infrastructure-as-code tooling begin? We're intentionally keeping cloud management as metadata/observability rather than lifecycle management, but the line may shift.
- **Secret management integration** — We observe secret variable names (never values) from two sources: `ks` (the project's CLI secret manager) and macOS Keychain (`security` command). Both need to be modelled as data sources. There's an open question about whether `ks` should become the single canonical interface and `security` be an implementation detail behind it, or whether Soji surfaces both. The long-term answer is probably a `soji.toml` declaration that says which secret provider a workspace uses.
- **LLM-assisted organization** — Could Soji suggest workspace templates or detect patterns? Deferred until the deterministic foundation is solid.
- **Process manager territory** — Soji kills processes but doesn't manage long-running services. The boundary with systemd/launchd (OS-level process supervisors) needs clarification.

## TUI Launch Modes

Soji detects its context from the working directory and launches the appropriate mode automatically.

### Global Mode

Launched from outside any workspace (or `soji --global`). System-level perspective. Tab navigation across three top-level views:

- **Workspaces** (home): All known workspaces as a summary list — active status, process count, port, memory, health. `Enter` to drill into a workspace. `u`/`d` to spin up or down.
- **Ports**: Full system Port Manager — all listening ports, three-tier grouped (workspace-bound → shared infrastructure → unaffiliated).
- **Services**: Shared infrastructure health — OrbStack, Docker, Ollama, local databases, etc.

This is the view you'd use from a standalone Soji window or for system-wide review.

### Workspace Mode

Launched from inside a workspace's directory tree (or `soji --workspace <name>`). Designed to live in a dedicated pane within your cmux workspace layout — open while you work, persistently showing that workspace's state.

Split-pane layout:
- **Left panel**: Collapsible sections — Processes, Ports (workspace-scoped), Env, Secrets, Git. Navigate sections with `↑`/`↓`; `Enter` on a section header to expand/collapse.
- **Right panel**: Detail pane for the currently selected item. Opens on `Enter`, closes on `Esc`.
- **Header**: Shows current workspace name + `← global` link to jump to Global Mode.

Port data in Workspace Mode is filtered to workspace-bound processes only — no shared infrastructure noise, no unaffiliated processes.

### Context Detection

```
soji          # detects CWD → Workspace Mode if inside a workspace, else Global Mode
soji --global # always Global Mode
soji --workspace kata  # Workspace Mode for 'kata', regardless of CWD
```

CWD matching checks against all known workspace roots, including worktrees at any depth under a workspace root.

## TUI Interaction Model

### Navigation

| Key | Action |
|-----|--------|
| `↑` / `↓` (also `k` / `j`) | Navigate rows in current panel |
| `Tab` / `Shift-Tab` | Switch between panels/sections |
| `1`–`9` | Jump to workspace by index |
| `Enter` | Open detail pane for current item |
| `Space` | Toggle selection (multi-select) |
| `k` | Kill selected processes (or current row if no selection) |
| `Esc` | Close detail pane / clear selection |
| `?` | Keybindings help overlay |
| Mouse click | Move cursor to row; double-click to open detail |

`Enter` always means "show me more." It never triggers a destructive action. Killing requires explicit `k` after selection.

Mouse support is built in from the start via crossterm's `EnableMouseCapture` — click to navigate, don't require keyboard-only use.

### Layout

Two primary views, switchable via `Tab`:

**Workspace View** — workspace-bound processes, ports, and environment inline under each workspace. Clean. No shared infrastructure noise. No unaffiliated entries.

**Port Manager** — all listening ports across the system, grouped: workspace-bound → shared infrastructure → unaffiliated. Full command, PID, port, scope attribution. Interactive multi-select kill.

## Roadmap

| Milestone | Focus | Status |
|-----------|-------|--------|
| **M0** | Printf dashboard + process cleanup | Done |
| **M0.5** | Foundation — vision, research spikes, user stories | Active |
| **M1** | ratatui TUI shell: Port Manager panel, split-pane detail view, interactive kill, ProcessScope model, full command resolution | Planned |
| **M1.5** | Workspace observability: env vars (direnv), secrets (`ks` + macOS Keychain), config file inventory, expanded domain model | Planned |
| **M2** | Canonical workspace definition (`soji.toml`), lifecycle commands (`soji up`/`down`), orchestration of cmux + ks + services | Planned |
| **M3** | Extended sources, smarter orphan detection, cloud observability (Vercel, Supabase), workspace health indicators | Planned |
| **M4** | Cross-repo org view, workspace templates, drift detection | Planned |
| **M5** | Snapshot history, weekly diff, LLM-assisted organization suggestions | Planned |

See [GitHub Milestones](https://github.com/cmbays/soji/milestones) for detailed issue breakdown.

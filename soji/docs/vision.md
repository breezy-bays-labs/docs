---
title: Vision
description: 'The identity, principles, and desired end state for Soji — why it exists and where it is going.'
category: canonical
status: draft
last_updated: 2026-03-03
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

### Pillar 2: Maintenance

**Keep the environment clean.**

- Kill orphaned Claude sessions and agents
- Prune stale git branches and orphaned worktrees
- Clean up stopped Docker containers, dangling images, unused volumes
- Reclaim memory from detached and zombie processes
- Identify and flag resource creep over time

Maintenance is reactive — it addresses entropy that has already accumulated. The goal is to make cleanup fast, safe, and satisfying.

#### Key Capabilities
- Interactive TUI cleanup with process selection and preview
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
- **Secret management integration** — We observe env vars (names only, never values) but the integration with ks and direnv needs design work.
- **LLM-assisted organization** — Could Soji suggest workspace templates or detect patterns? Deferred until the deterministic foundation is solid.
- **Process manager territory** — Soji kills processes but doesn't manage long-running services. The boundary with systemd/launchd (OS-level process supervisors) needs clarification.

## Roadmap

| Milestone | Focus | Status |
|-----------|-------|--------|
| **M0** | Printf dashboard + process cleanup | Done |
| **M0.5** | Foundation — vision, research spikes, user stories | Active |
| **M1** | ratatui TUI with live refresh, workspace-scoped views | Planned |
| **M2** | Canonical workspace model + lifecycle commands | Planned |
| **M3** | Extended sources, interactive cleanup, observability polish | Planned |
| **M4** | Cross-repo org view, workspace templates | Planned |
| **M5** | Snapshot history, drift detection | Planned |

See [GitHub Milestones](https://github.com/cmbays/soji/milestones) for detailed issue breakdown.

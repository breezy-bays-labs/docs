---
title: Conceptual Model
description: The Soji domain hierarchy — Org, Project, Workspace, Layout, Surface, Process — with definitions, relationships, and how soji.toml formalizes this structure at M2.
category: design
milestone: M1
status: final
last_updated: 2026-03-06
---

# Conceptual Model

Soji organizes everything it observes into a five-level hierarchy. Understanding this hierarchy is prerequisite to understanding how data flows through Soji, how scope classification works, and why the TUI is organized the way it is.

```
Org
└── Project (a git repository)
    └── Workspace (an active development environment for a project)
        ├── Layout (a cmux window with surfaces)
        │   └── Surface (a cmux tab/pane: terminal, browser, etc.)
        │       └── Process (any running OS process: Claude, dev server, MCP server, etc.)
        └── Environment (env vars, secrets, config files, git state)
```

Each level is described below.

---

## Org

**What it is**: The top-level organizational unit. In practice, this is a GitHub organization or a personal GitHub account (`cmbays`). An Org contains multiple Projects.

**Soji's relationship to Org**: Org-level views are a **M4 feature** — cross-repo dashboards, workspace templates that apply across all of an org's repos, org-level health indicators. In M1, Soji does not reason at the Org level. The concept is introduced here for architectural clarity: the hierarchy has a top, and that top is the Org.

**Why it matters now**: Data models that will eventually support org-level aggregation should not hardcode single-user assumptions. `Workspace.owner` and `Project.github_repo` fields should be designed to support an org prefix even if they are not yet used in multi-org contexts.

---

## Project

**What it is**: A git repository. The root of the filesystem hierarchy for a unit of code. Identified by its git root — the directory containing the `.git` directory (or for worktrees, the `.git` file pointing to the worktree parent).

**How Soji identifies a Project**: By its git root. `git rev-parse --show-toplevel` (or equivalent) from any directory inside the repo returns the project root. Soji builds its list of known projects by:
1. Reading known workspace roots (from cmux workspace list)
2. Running `git rev-parse --show-toplevel` for each workspace root
3. Deduplicating — two workspaces serving the same repo map to one Project

**Worktrees and Projects**: A git worktree at `/Users/cmbays/Github/kata/.claude/worktrees/parallel-task-abc/` belongs to the same Project as the main worktree at `/Users/cmbays/Github/kata/`. The git root for both is `/Users/cmbays/Github/kata`. This is the foundation for worktree CWD matching — any CWD that descends from a known Project root is attributed to that Project.

**Project vs Workspace**: A Project is the static artifact (the code repository). A Workspace is the live instance (the active development environment around that code). One Project can have multiple active Workspaces — for example, `kata` project might have:
- A `kata-main` workspace for ongoing development
- A `kata-feature-tui` workspace on a feature worktree
- A `kata-review` workspace for reviewing a PR

**Soji stores for each Project**:
- `root: PathBuf` — absolute filesystem path to git root
- `github_repo: Option<String>` — `owner/repo` string from `git remote get-url origin`
- `name: String` — derived from directory name (`kata` from `/Users/cmbays/Github/kata`)

**Milestone**: Project is implicit in M1 (used for CWD matching). Explicit Project entity in the domain model at M2 when `soji.toml` formalizes project identity.

---

## Workspace

**What it is**: An active development environment for a Project. A Workspace = the Project plus everything currently serving its development: cmux surfaces, running processes, loaded environment, cloud assets.

**The central abstraction in Soji.** Everything Soji monitors is attributed to a Workspace (or flagged as belonging to no workspace). The three ProcessScope tiers (WorkspaceBound, SharedInfra, Unaffiliated) are defined relative to the Workspace concept.

**How Soji identifies a Workspace**: Through cmux. `cmux list-workspaces` returns a list of active cmux workspaces with their names and root directories. Each cmux workspace that maps to a git repository root becomes a Soji Workspace. The cmux workspace name becomes the Soji workspace name.

Without cmux: Soji attempts to identify workspaces from the set of processes whose CWDs can be grouped by git root. This is a weaker signal — it finds projects that have running processes, but misses projects that exist but are currently idle.

**Multiple Workspaces per Project**: A single Project can have multiple Workspaces at the same time. This happens with git worktrees: a developer working on two parallel features may have:
- cmux workspace `kata-main` → CWD `/Users/cmbays/Github/kata`
- cmux workspace `kata-tui` → CWD `/Users/cmbays/Github/kata/.claude/worktrees/tui-feature`

Both workspaces share the same Project root (same git repo), but are distinct Workspaces with their own processes, ports, and cmux layouts.

**What Soji stores for a Workspace** (M1 scope):
```rust
pub struct Workspace {
    pub name: String,                     // cmux workspace name
    pub root: PathBuf,                    // CWD of the workspace root
    pub surfaces: Vec<Surface>,           // cmux panes/tabs
    pub processes: Vec<Process>,          // all WorkspaceBound processes
    pub git_branch: Option<String>,       // current branch
    pub git_dirty: bool,                  // uncommitted changes present
}
```

**Planned expansion at M1.5**:
```rust
pub struct Workspace {
    // ... M1 fields above, plus:
    pub env_vars: Vec<EnvVar>,            // names + source, never values
    pub secrets: Vec<SecretRef>,          // names + provider + loaded bool
    pub config_files: Vec<ConfigFile>,    // .envrc, soji.toml, docker-compose.yml presence
}
```

---

## Layout

**What it is**: A cmux window within a Workspace. A Layout contains Surfaces (panes and tabs). Most Workspaces have exactly one Layout (one cmux window). Complex setups may have multiple windows within a single cmux workspace.

**How Soji identifies a Layout**: From cmux's workspace structure. `cmux list-workspaces` returns workspaces; each workspace has one or more windows; each window is a Layout.

**Soji's use of Layout**: In M1, Layout is not explicitly represented as a separate domain type — it is folded into the Workspace (a Workspace contains Surfaces directly). The Layout level is made explicit in M2 when `soji.toml` defines which Layouts a Workspace should have on startup.

**Why it matters**: Layout is the structural container that makes the reference layout spec (see `reference-layout.md`) work. A three-column Layout has three top-level pane slots. Tab groups within a pane slot are sub-Layouts. Understanding that Soji occupies one Surface within one Layout within one Workspace clarifies the scope of what Scout Mode can see: it sees the Workspace (all processes, all ports), but it lives in one Surface of one Layout.

---

## Surface

**What it is**: A cmux tab or pane — a single terminal-or-browser unit within a Layout. The fundamental unit Soji monitors. Every process that Soji tracks is associated with a Surface (or, for daemon processes, is marked as having no surface).

**Seven surface types**: See `surface-types.md` for the complete taxonomy. The types are:
1. Agent / LLM session
2. Runner / log tail
3. Passive monitor
4. Directory navigator
5. Active work surface (editor)
6. Quick terminal
7. Browser tab

**Surface identity**: cmux assigns each pane a numeric ID and, optionally, a name. Soji uses the cmux pane ID as the Surface identifier. The surface name (if set) is displayed in the left panel.

**Process-to-Surface attribution**: Soji maps OS processes to Surfaces by:
1. Reading the cmux surface list (pane ID → PID of the foreground process)
2. Walking the process tree: all children of the foreground process are attributed to the same Surface
3. This gives a Surface → [PIDs] mapping. The left panel's process rows are organized under their parent Surface.

**What Soji stores for a Surface** (M1):
```rust
pub struct Surface {
    pub pane_id: u32,                     // cmux pane ID
    pub name: Option<String>,             // cmux pane name if set
    pub surface_type: SurfaceType,        // from cmux list-pane-surfaces
    pub cwd: Option<PathBuf>,             // from cmux sidebar-state
    pub git_branch: Option<String>,       // from cmux sidebar-state
    pub processes: Vec<Process>,          // processes attributed to this surface
}
```

**Browser Surface**: Browser tabs are a special case. The OS process (`arc`, `Google Chrome`) is a single process covering all tabs. Individual tabs are identified via the cmux sidebar integration, not via OS process signals. Browser Surfaces have `processes = []` (no directly attributable processes) and are identified solely by their URL and title from the sidebar.

---

## Process

**What it is**: Any running OS process associated with a Workspace (or not associated with any workspace). Identified by PID. Carries the full context Soji can resolve: full command, CWD, memory, ports, and ProcessScope.

**ProcessScope**: The classification system for processes. Every `Process` struct carries a `scope: ProcessScope` field computed at gather time by the `classify_scope()` pure function.

```rust
pub enum ProcessScope {
    WorkspaceBound { workspace_name: String },   // CWD at or under a workspace root
    SharedInfra { service_name: String },         // well-known service: OrbStack, Ollama, etc.
    Unaffiliated,                                  // everything else
}
```

**Scope classification logic** (in `src/sources/scope.rs`):
1. **WorkspaceBound check**: Does `process.cwd` start with any known workspace root? If yes → `WorkspaceBound`. The workspace with the longest matching prefix wins (most specific workspace).
2. **SharedInfra check**: Does `process.name` match the static SharedInfra name table, OR does `process.listening_port` match the SharedInfra port table? If yes → `SharedInfra`.
3. **Default**: `Unaffiliated`.

**SharedInfra name table** (M1 static list, expandable via `~/.config/soji/services.toml` in M3):
- OrbStack (`orbstack`, `com.docker.backend`)
- Docker daemon (`dockerd`, `containerd`, `docker-proxy`)
- Ollama (`ollama`)
- Local databases: Postgres (`postgres`, `pg_ctl`), MySQL (`mysqld`), Redis (`redis-server`), SQLite (`sqlite3`)

**SharedInfra port table**:
- :2375 / :2376 — Docker daemon
- :11434 — Ollama
- :5432 — PostgreSQL
- :3306 — MySQL
- :6379 — Redis

**Worktree handling for WorkspaceBound**: A process with CWD `/Users/cmbays/Github/kata/.claude/worktrees/parallel-task-abc/src` is WorkspaceBound to `kata` because `/Users/cmbays/Github/kata/.claude/worktrees/parallel-task-abc/src` starts with the kata workspace root `/Users/cmbays/Github/kata`. Worktrees at any depth under the repo root are correctly attributed.

**Process vs Container**: Docker containers are not OS processes in the traditional sense — they are managed by the Docker daemon and surfaced via `docker ps`. Soji models containers separately (as `Container` domain types) rather than forcing them into the `Process` model. Both appear in the Workspace Model left panel but in different sections (Processes vs Docker).

**What Soji stores for a Process** (M1):
```rust
pub struct Process {
    pub pid: u32,
    pub name: String,                         // process name from ps
    pub full_command: String,                 // resolved from ps -o args= (no truncation)
    pub cwd: Option<PathBuf>,                 // from lsof -d cwd
    pub memory_mb: u32,                       // RSS from ps aux, converted from KB
    pub listening_ports: Vec<u16>,            // from lsof -iTCP -sTCP:LISTEN
    pub scope: ProcessScope,                  // computed by classify_scope()
    pub status: ProcessStatus,                // Active / Orphaned / Detached / etc.
    pub surface_id: Option<u32>,              // cmux pane ID if attributed to a surface
    // Claude-specific (Option because most processes are not Claude):
    pub session_id: Option<String>,           // --session-id flag value
    pub model: Option<String>,                // --model flag value
    pub agent_id: Option<String>,             // --agent-id flag value
}
```

---

## Environment

**What it is**: The non-process state associated with a Workspace — environment variables, secrets, config files, and git state. Orthogonal to the Surface/Process hierarchy: a Workspace has both a Layout (surfaces and processes) and an Environment.

**M1 scope**: Git state only (branch, dirty status). The rest is M1.5.

**M1.5 scope**:
- `env_vars: Vec<EnvVar>` — names + source (Direnv, DotEnv, Ks, Shell). Never values.
- `secrets: Vec<SecretRef>` — names + provider + loaded bool. Never values.
- `config_files: Vec<ConfigFile>` — presence of `.envrc`, `soji.toml`, `docker-compose.yml`, `.env*` files

```rust
pub struct EnvVar {
    pub name: String,
    pub source: EnvSource,                    // Direnv | DotEnv | Ks | Shell
    pub is_secret: bool,                      // heuristic: name contains TOKEN/KEY/SECRET/etc.
}

pub struct SecretRef {
    pub name: String,
    pub provider: SecretProvider,             // Ks | MacosKeychain
    pub loaded: bool,                         // is it accessible in the current environment?
}
```

---

## How soji.toml Formalizes This (M2)

In M1 and M1.5, the Workspace model is **discovered** — Soji infers it from running processes, cmux state, and git roots. This works for observability but not for lifecycle management. You cannot `soji up` a workspace that has never been defined.

At M2, `soji.toml` makes the Workspace model **declarative**. A `soji.toml` at the Project root defines:

```toml
[workspace]
name = "soji"
root = "."
cmux_workspace = "soji"

[[workspace.services]]
name = "dev server"
command = "bun run dev"
port = 3000

[[workspace.secrets]]
name = "GITHUB_TOKEN"
provider = "ks"
required = true

[[workspace.layout.panes]]
name = "agent"
command = "claude"
size = "70%"

[[workspace.layout.panes]]
name = "soji"
command = "soji"
size = "30%"
```

With `soji.toml` present, Soji shifts from inference to verification: it knows what a healthy workspace looks like (defined state) and can compare it to what is actually running (observed state). Drift between declared and observed state is surfaced as a health warning.

`soji up` reads `soji.toml`, creates the cmux workspace and layout, starts services in the defined panes, loads env vars via direnv, and verifies secrets are loaded. `soji down` tears it all down cleanly.

The conceptual model (Project → Workspace → Layout → Surface → Process + Environment) is the same whether Soji is in discovery mode (M1) or declarative mode (M2+). The hierarchy is stable; the data source changes from inference to definition.

---
shaping: true
pipeline: m1.5-workspace-observability
stage: shaping
shape_selected: A
---

# Shaping: Soji M1.5 — Workspace Observability

## Requirements

| Req | Requirement | Status |
|-----|-------------|--------|
| R0 | ENV section: parse `.envrc` for declared vars (plain `export` and `$(secret path)` patterns); cross-reference against shell environment (`std::env::var`) for loaded/missing gap; annotate each var with source and `is_sensitive` heuristic. | Must-have |
| R1 | SECRETS section: parse `.envrc` for `$(secret <path>)` calls to build the workspace's secret manifest; cross-reference with `ks ls` for exists-in-keychain; cross-reference with shell env for loaded — three-state per secret. Values never shown. | Must-have |
| R2 | Secrets scope: `shared/` secrets visually distinguished from workspace-scoped (`<workspace>/`) secrets throughout the SECRETS section. | Must-have |
| R3 | GIT section: local git state from `git -C <cwd>` — current branch, dirty/clean, open worktrees (path + branch + dirty), stash count, ahead/behind best-effort. | Must-have |
| R4 | GITHUB section: open milestones with open/closed issue counts; expand milestone → open issues with title and labels; labels view showing only labels that appear on currently-open issues — all via `gh` CLI. | Must-have |
| R5 | CONFIG section: presence indicators for `.envrc`, `docker-compose.yml`, `docker-compose.yaml`, `soji.toml`, `.env`, `.env.local`, `.env.development`, `Makefile`; Enter on `.envrc` opens inline content viewer in right panel. | Must-have |
| R6 | Section expansion in TUI: ENV/SECRETS/GIT/GITHUB/CONFIG headers in the left panel become expandable/collapsible via Enter; child `NavItem` variants appear inline on expansion; existing surface expand/collapse pattern is extended (not replaced). | Must-have |
| R7 | Workspace-level separation: ENV/SECRETS/GIT/GITHUB/CONFIG sections live at the workspace level and are not repeated per surface; surface rows remain process/port focused. | Must-have |
| R8 | Graceful degradation: missing `.envrc` → ENV/SECRETS show informative empty state; `gh` not authed → GITHUB shows "gh auth required"; `git` not found → GIT shows unavailable; `ks` not installed → SECRETS shows unavailable. No panics. | Must-have |

**Out of scope for M1.5:**
- `soji.toml` label configuration (which labels to surface) → M2
- Re-provisioning secrets from within the TUI → M2
- Deep-link to lazygit from GIT section → M2
- Cloud observability (Vercel, Supabase) → M3
- Git branch pruning → M2
- `direnv export json` subprocess (not needed — Soji inherits shell env, `std::env::var` is sufficient for Workspace Mode)

---

## Shapes

### Shape A: Layered extension on M1 scaffold *(selected)*

Extend M1 Shape A with new domain types, new source modules, and TUI wiring. All additive — no architectural changes to the existing `App`, `events`, `ui/*`, `sources`, `domain` separation.

**Why selected:** The existing scaffold was designed for exactly this extension. `Workspace` struct gains optional fields. New source modules plug into the existing `tokio::join!` in `gather_snapshot()`. The section header nav items already exist — they just need expansion logic and real data. Shape A preserves the proven architecture while filling in the M1 stubs cleanly.

**Discarded alternatives:**

- **Shape B: Separate workspace-observability snapshot pass** — Gather env/secrets/git/github as a second async pass after the main snapshot. Cleaner isolation in theory, but two-phase gather adds latency and complexity. The domain naturally extends `Workspace` — no reason to separate.

- **Shape C: Per-section lazy loading** — Load data on demand when user navigates to a section. Faster startup, but introduces async triggers from TUI event loop, makes first section access feel laggy, complicates App state machine. Not worth it for M1.5 data volume.

---

## Shape A: Parts

| Part | Mechanism | Flag |
|------|-----------|:----:|
| **A1** | Add to `src/domain/workspace.rs`: `WorkspaceEnv { vars: Vec<EnvVar>, envrc_present: bool }`, `EnvVar { name: String, source: EnvVarSource, ks_path: Option<String>, loaded: bool, is_sensitive: bool }`, `EnvVarSource` enum (Direnv / DotEnv / Shell), `WorkspaceSecrets { refs: Vec<SecretRef> }`, `SecretRef { name: String, ks_path: String, scope: SecretScope, category: String, exists_in_keychain: bool, loaded: bool }`, `SecretScope` enum (Shared / Workspace), `WorkspaceGit { branch: Option<String>, dirty: bool, ahead: u32, behind: u32, stash_count: usize, worktrees: Vec<WorktreeInfo> }`, `WorktreeInfo { path: String, branch: Option<String>, dirty: bool }`, `WorkspaceGithub { milestones: Vec<Milestone>, labels_on_open_issues: Vec<String>, auth_available: bool }`, `Milestone { title: String, number: u64, open_issues: u32, closed_issues: u32, issues: Vec<Issue> }`, `Issue { number: u64, title: String, labels: Vec<String> }`, `ConfigFiles { has_envrc: bool, has_docker_compose: bool, has_soji_toml: bool, has_makefile: bool, dot_env_files: Vec<String>, has_cargo_toml: bool, has_package_json: bool }`. Add `env: Option<WorkspaceEnv>`, `secrets: Option<WorkspaceSecrets>`, `git: Option<WorkspaceGit>`, `github: Option<WorkspaceGithub>`, `config_files: Option<ConfigFiles>` to `Workspace` struct. | |
| **A2** | Add `src/sources/env.rs`: `EnvSource::gather(cwd: &str) -> WorkspaceEnv`. (1) Read `.envrc` at cwd; regex-parse lines: `export VAR=$(secret path)` → EnvVar with ks_path; `export VAR=.*` → plain EnvVar. (2) For each declared var: `loaded = std::env::var(&name).is_ok()`. (3) `is_sensitive` heuristic: name contains TOKEN, KEY, SECRET, PASSWORD, CREDENTIAL, CERT (case-insensitive). Returns `WorkspaceEnv { vars, envrc_present }`. No subprocess — inherits shell env. | |
| **A3** | Add `src/sources/secrets.rs`: `SecretsSource::gather(cwd: &str, env_vars: &[EnvVar]) -> WorkspaceSecrets`. (1) Extract `SecretRef` list from `env_vars` where `ks_path.is_some()`. (2) Run `ks ls` once; parse output into a `HashSet<String>`. (3) For each ref: `exists_in_keychain = ks_list.contains(&ks_path)`. (4) `loaded = std::env::var(&name).is_ok()`. (5) Parse scope from ks_path prefix: `shared/` → `SecretScope::Shared`; else → `SecretScope::Workspace`. Parse category as second path segment. Degrade gracefully if `ks` not installed (`which ks` check). | |
| **A4** | Add `src/sources/git.rs`: `GitSource::gather(cwd: &str) -> Option<WorkspaceGit>`. Shell out: `git -C <cwd> rev-parse --abbrev-ref HEAD` (branch), `git -C <cwd> status --porcelain` (dirty = non-empty), `git -C <cwd> worktree list --porcelain` (parse into `WorktreeInfo` list), `git -C <cwd> stash list` (count lines), `git -C <cwd> rev-list --count @{u}..HEAD 2>/dev/null` and `HEAD..@{u} 2>/dev/null` (ahead/behind, best-effort — default 0 on failure). Returns `None` if git not available or cwd has no git repo. | |
| **A5** | Add `src/sources/github.rs`: `GitHubSource::gather(cwd: &str) -> Option<WorkspaceGithub>`. (1) `git -C <cwd> remote get-url origin` → parse GitHub `org/repo` slug. (2) Check auth: `gh auth status` exit code (0 = authed). (3) `gh api repos/{owner}/{repo}/milestones` → JSON array with fields `title`, `number`, `open_issues`, `closed_issues` (snake_case). (4) `gh issue list --repo org/repo --state open --json number,title,labels` (all open issues); labels are objects `{ name, ... }` — extract `.name`. (5) Aggregate labels: union of all label names on open issues. (6) Eager loading at snapshot time — total latency ~700ms, well under threshold. Returns `WorkspaceGithub { milestones: [], labels_on_open_issues: [], auth_available: false }` if gh not installed or not authed. | |
| **A6** | Add `src/sources/config.rs`: `ConfigSource::scan(cwd: &str) -> ConfigFiles`. Check `std::fs::metadata` for each: `.envrc`, `docker-compose.yml`, `docker-compose.yaml`, `soji.toml`, `Makefile`, `Cargo.toml`, `package.json`. Glob for `.env*` files (`.env`, `.env.local`, `.env.development`, `.env.production`, `.env.test`). No file parsing. Fast. | |
| **A7** | Wire all new sources into `src/sources/mod.rs` `gather_snapshot()`. Add to `tokio::join!` in parallel. Share env output between A2 and A3 (secrets source consumes the env var list to avoid re-parsing `.envrc`). Attach results to each `workspace` in `workspaces` vec. One gather call per workspace — if multiple workspaces, gather all concurrently. | |
| **A8** | Extend `NavItem` in `src/tui/app.rs` with new variants: `EnvVarRow(usize)` (index into workspace env vars), `SecretRow(usize)` (index into workspace secrets), `WorktreeRow(usize)`, `MilestoneRow(usize)`, `IssueRow(usize, usize)` (milestone idx, issue idx), `ConfigFileRow(String)`. Extend `update_nav_workspace()` to emit child rows when a `SectionHeader` is in `expanded` set. Extend `toggle_expand()` to handle `SectionHeader` items (keyed by `SectionKind` string). | |
| **A9** | Update `src/tui/ui/left.rs`: render new `NavItem` variants. `EnvVarRow`: `[✓/✗] VAR_NAME  source  [secret]`. `SecretRow`: `[✓/✗/⚠] path  [shared/ws]  [exists/missing]`. `WorktreeRow`: `● path  branch  [dirty]`. `MilestoneRow`: `title  N open / M closed`. `IssueRow`: `#N title  [label]`. `ConfigFileRow`: `filename  [present/absent]`. Section header rows gain `▸` (collapsed) / `▾` (expanded) toggle indicator and a live summary (e.g. `ENV  12 vars  2 missing`). | |
| **A10** | Update `src/tui/ui/right.rs`: real detail views for section types. `SectionKind::Env` → table of all declared vars with name, source, ks_path, loaded status, sensitive flag. `SectionKind::Secrets` → manifest grouped by scope (shared first, then workspace), three-state per row. `SectionKind::Git` → branch + ahead/behind + dirty + stash count + worktrees table. `SectionKind::Github` → milestone rows with progress indicators; milestone expand shows issue list; labels panel. `SectionKind::Config` → file list; if `.envrc` selected in left panel, show file contents in right panel (read-only, no values redacted since .envrc has no values). Add `SectionKind::Github` and `SectionKind::Config` variants. | |
| **A11** | Add `SectionKind::Github` and `SectionKind::Config` variants to `SectionKind` enum in `src/tui/app.rs`. Update `update_nav_workspace()` to emit these section headers after the existing Env/Secrets/Git headers. Update all match exhaustiveness in events.rs and ui modules. | |

---

## Fit Check

| Req | Requirement | Status | A |
|-----|-------------|--------|---|
| R0 | ENV section: `.envrc` parsing, shell env cross-reference, loaded/missing gap, source annotation | Must-have | ✅ |
| R1 | SECRETS section: `.envrc` → ks path manifest, `ks ls` cross-reference, three-state (declared/exists/loaded), values never shown | Must-have | ✅ |
| R2 | Secrets scope: `shared/` visually distinct from workspace-scoped | Must-have | ✅ |
| R3 | GIT section: branch, dirty, worktrees, stash count, ahead/behind | Must-have | ✅ |
| R4 | GITHUB section: milestones + issue counts, issue list with labels, labels on open issues, via `gh` CLI | Must-have | ✅ |
| R5 | CONFIG section: presence indicators, `.envrc` inline viewer | Must-have | ✅ |
| R6 | Section expansion: expandable headers, child NavItem rows, extend existing pattern | Must-have | ✅ |
| R7 | Workspace-level separation: sections at workspace level, not per-surface | Must-have | ✅ |
| R8 | Graceful degradation: informative empty states for all missing tools/auth | Must-have | ✅ |

Shape A passes all requirements. No flagged unknowns. All mechanisms are understood.

**Spike resolved (spike-github.md):** `gh milestone` subcommand does not exist — use `gh api repos/{owner}/{repo}/milestones`. Fields are snake_case (`open_issues`, `closed_issues`). Labels are objects, extract `.name`. Auth via `gh auth status` exit code confirmed reliable. Eager loading at ~700ms total latency confirmed acceptable.

---

## Decision Log

| Decision | Rationale |
|----------|-----------|
| Shape A (layered extension, no rewrite) | M1 scaffold is proven and sound. All M1.5 changes are additive. |
| `std::env::var` for loaded check (not `direnv export json`) | Soji inherits the shell environment in Workspace Mode. Subprocess not needed, avoids direnv trust prompts, simpler. |
| `.envrc` as declared layer (not soji.toml) | `.envrc` is committed to git, present in all worktrees, already self-documenting. `soji.toml` declarations deferred to M2. |
| `ks` for secrets source (not `security` directly) | After reviewing the architecture: `ks ls` is a thin wrapper but gives the clean `scope/category/key` naming we need. Direct `security dump-keychain` gives raw keychain entries without the path structure. `ks ls` is the right level of abstraction here. Note: earlier memory said "use security directly" — this decision supersedes that for the listing use case. `security` is still used for the actual show/add/remove operations that `ks` wraps. |
| `gh api` for milestones (not `gh milestone list`) | `gh milestone` subcommand does not exist. `gh api repos/{owner}/{repo}/milestones` is the correct path. Confirmed by spike. |
| `gh` CLI for GitHub (not direct API via reqwest/octocrab) | gh is installed, authed, and abstracts token management. Shell-out architecture principle. Direct API would require token threading. |
| Eager GitHub data gather (not lazy-load per section) | Consistency with other sections. GitHub data is gathered at snapshot time alongside env/git. Lazy-load adds async complexity to event loop. Acceptable latency at M1.5 scale. |
| Github and Config as new SectionKind variants | GitHub and Config are distinct enough from Env/Secrets/Git to warrant their own section kinds and right-panel detail views. |
| Section child rows as NavItem variants (not overlay) | Consistent with existing surface-expand pattern (surfaces expand to show process child rows). Overlay would break progressive disclosure model. |
| `shared/` secrets shown in workspace SECRETS section | Shared secrets are loaded in the workspace env — they're relevant to workspace readiness. Scope distinction (visual badge) is sufficient to avoid confusion. |

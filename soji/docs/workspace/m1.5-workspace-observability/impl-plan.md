---
pipeline: m1.5-workspace-observability
stage: impl-planning
---

# Implementation Plan: Soji M1.5 — Workspace Observability

**Goal:** Extend the M1 ratatui TUI to surface real env/secrets/git/GitHub/config data in Workspace Mode's section stubs.

**Architecture:** Additive extension of the M1 scaffold. Five new source modules gather data into five new domain types on `Workspace`. New `NavItem` variants enable section expansion. Left and right panels render the new data. No existing behavior is changed — only extended.

**Tech Stack:** Rust 2024 edition, ratatui 0.29, crossterm 0.28, tokio async, serde_json (already in Cargo.toml), anyhow, tracing. Shell-out architecture: `std::env::var`, `ks`, `git`, `gh` CLI.

**Appetite:** Medium — 7 sessions across 4 waves. No new crates needed (serde_json already present; no regex needed — .envrc parsing uses string matching).

---

## Wave Structure

```
W0: Domain Types (serial — everything depends on this)
    └─ 0.1: workspace-domain-types         [workspace.rs]

W1: Sources (parallel — all new files, no conflicts)
    ├─ 1.1: env-secrets-sources            [env.rs, secrets.rs]
    └─ 1.2: git-github-sources             [git.rs, github.rs]

W2: Wiring + App Extensions (parallel — different files)
    ├─ 2.1: config-source-and-wiring       [config.rs, sources/mod.rs]
    └─ 2.2: app-and-events-extensions      [tui/app.rs, tui/events.rs]

W3: Rendering (parallel — different files)
    ├─ 3.1: left-panel-rendering           [tui/ui/left.rs]
    └─ 3.2: right-panel-rendering          [tui/ui/right.rs]
```

**Critical path:** W0 → W1 → W2 → W3 (4 sequential gates, W1/W2/W3 have parallel sessions within each wave)

---

## Wave 0: Domain Types

### Task 0.1: Workspace Domain Types

**Topic:** `workspace-domain-types`
**Worktree:** `git worktree add ../.claude/worktrees/workspace-domain-types -b workspace-domain-types`
**Depends on:** None
**Complexity:** Medium
**Files:**
- `src/domain/workspace.rs` — extend with all new M1.5 types and `Workspace` fields

**Steps:**
1. Add `WorkspaceEnv { vars: Vec<EnvVar>, envrc_present: bool }` and `EnvVar { name: String, source: EnvVarSource, ks_path: Option<String>, loaded: bool, is_sensitive: bool }` and `EnvVarSource { Direnv, DotEnv, Shell }` enum.
2. Add `WorkspaceSecrets { refs: Vec<SecretRef>, unavailable_reason: Option<String> }` and `SecretRef { name: String, ks_path: String, scope: SecretScope, category: String, exists_in_keychain: bool, loaded: bool }` and `SecretScope { Shared, Workspace }` enum.
3. Add `WorkspaceGit { branch: Option<String>, dirty: bool, ahead: u32, behind: u32, stash_count: usize, worktrees: Vec<WorktreeInfo> }` and `WorktreeInfo { path: String, branch: Option<String>, dirty: bool }`.
4. Add `WorkspaceGithub { milestones: Vec<Milestone>, labels_on_open_issues: Vec<String>, auth_available: bool }`, `Milestone { title: String, number: u64, open_issues: u32, closed_issues: u32, issues: Vec<Issue> }`, `Issue { number: u64, title: String, labels: Vec<String> }`.
5. Add `ConfigFiles { has_envrc: bool, has_docker_compose: bool, has_soji_toml: bool, has_makefile: bool, has_cargo_toml: bool, has_package_json: bool, dot_env_files: Vec<String> }`.
6. Add optional fields to `Workspace` struct: `pub env: Option<WorkspaceEnv>`, `pub secrets: Option<WorkspaceSecrets>`, `pub git: Option<WorkspaceGit>`, `pub github: Option<WorkspaceGithub>`, `pub config_files: Option<ConfigFiles>`.
7. Derive `Debug, Clone, Serialize, Deserialize` on all new types. All enums need `PartialEq, Eq`.
8. Run `cargo build` — must compile with zero errors.

**Session Prompt:**
```
You are implementing Soji M1.5 domain types. Soji is a Rust TUI (ratatui 0.29, tokio async, Rust 2024 edition).

Read these files first:
- docs/workspace/m1.5-workspace-observability/shaping.md — Part A1 describes all types to add
- src/domain/workspace.rs — current state; you are adding to this file only

Task: Extend src/domain/workspace.rs with all new M1.5 types listed in shaping.md Part A1.

Key requirements:
- ALL new types use `#[derive(Debug, Clone, Serialize, Deserialize)]`; enums also `PartialEq, Eq`
- Add 5 optional fields to the existing `Workspace` struct: env, secrets, git, github, config_files
- Default values for the new Workspace fields must not break existing code — all are Option<T>, so existing construction still compiles
- Do NOT modify any other file
- Run `cargo build` to verify; fix any compilation errors

Output: modified src/domain/workspace.rs only.
```

**Acceptance:**
- `cargo build` passes
- All 5 optional `Workspace` fields present
- All supporting types present with correct derives
- Existing `Workspace` construction in other files still compiles (all new fields are `Option`, default to absent)

---

## Wave 1: Sources

### Task 1.1: Env + Secrets Sources

**Topic:** `env-secrets-sources`
**Worktree:** `git worktree add ../.claude/worktrees/env-secrets-sources -b env-secrets-sources`
**Depends on:** `0.1-workspace-domain-types` merged
**Complexity:** Medium
**Files:**
- `src/sources/env.rs` — new file
- `src/sources/secrets.rs` — new file

**Steps:**
1. Create `src/sources/env.rs` with `pub struct EnvSource;` and `impl EnvSource { pub async fn gather(cwd: &str) -> WorkspaceEnv }`.
   - Read `.envrc` with `tokio::fs::read_to_string`. On error: return `WorkspaceEnv { vars: vec![], envrc_present: false }`.
   - Parse lines: skip comments (`#`), skip blank lines.
   - Match `export VAR=$(secret path)` pattern: `contains("$(secret ")` — extract VAR name (before `=`) and ks_path (between `$(secret ` and `)`).
   - Match `export VAR=...` (plain assignment): extract VAR name.
   - For each declared var: `loaded = std::env::var(&name).is_ok()`.
   - `is_sensitive` heuristic: `name.to_uppercase().contains(any of ["TOKEN", "KEY", "SECRET", "PASSWORD", "CREDENTIAL", "CERT"])`.
   - Return `WorkspaceEnv { vars, envrc_present: true }`.

2. Create `src/sources/secrets.rs` with `pub struct SecretsSource;` and `impl SecretsSource { pub async fn gather(cwd: &str, env_vars: &[EnvVar]) -> WorkspaceSecrets }`.
   - Build `Vec<SecretRef>` from env_vars where `ks_path.is_some()`.
   - Check if `ks` is available: `tokio::process::Command::new("which").arg("ks").output()`. If unavailable: return `WorkspaceSecrets { refs: vec![], unavailable_reason: Some("ks not installed".to_string()) }`.
   - Run `ks ls` once: `tokio::process::Command::new("ks").arg("ls").output()`. Parse stdout lines into `HashSet<String>` of known paths.
   - For each `SecretRef`: `exists_in_keychain = ks_set.contains(&ref.ks_path)`, `loaded = std::env::var(&ref.name).is_ok()`.
   - Parse scope: `ks_path.starts_with("shared/")` → `SecretScope::Shared`; else → `SecretScope::Workspace`.
   - Parse category: second path segment (e.g., `mokumo/supabase/url` → category `"supabase"`).
   - Return `WorkspaceSecrets { refs, unavailable_reason: None }`.

3. Graceful degradation: all errors return empty/unavailable struct rather than propagating. Use `tracing::warn!` for degraded paths.

4. Add `mod env; mod secrets;` and `pub use env::EnvSource; pub use secrets::SecretsSource;` to `src/sources/mod.rs`. Do NOT yet wire into `gather_snapshot()` — that's Task 2.1.

5. Run `cargo build` — must compile.

**Session Prompt:**
```
You are implementing two new Soji M1.5 data source modules. Soji is a Rust TUI (ratatui 0.29, tokio async, Rust 2024 edition).

Read these files first:
- docs/workspace/m1.5-workspace-observability/shaping.md — Parts A2 and A3
- src/domain/workspace.rs — WorkspaceEnv, EnvVar, EnvVarSource, WorkspaceSecrets, SecretRef, SecretScope types
- src/sources/mod.rs — existing pattern to follow for module structure

Task: Create src/sources/env.rs and src/sources/secrets.rs.

EnvSource::gather(cwd: &str) -> WorkspaceEnv:
- Read .envrc at cwd using tokio::fs::read_to_string
- Parse lines for `export VAR=$(secret path)` (secret-backed) and `export VAR=...` (plain)
- For each var: loaded = std::env::var(&name).is_ok()
- is_sensitive: name contains TOKEN, KEY, SECRET, PASSWORD, CREDENTIAL, or CERT (case-insensitive check)
- Return WorkspaceEnv { vars, envrc_present: bool }; empty vars on missing .envrc

SecretsSource::gather(cwd: &str, env_vars: &[EnvVar]) -> WorkspaceSecrets:
- Filter env_vars to those with ks_path.is_some() to build the SecretRef list
- Run `ks ls` once; parse output lines into HashSet<String>
- For each ref: exists_in_keychain = set contains ks_path; loaded = std::env::var(&name).is_ok()
- Scope: "shared/" prefix → SecretScope::Shared; else → SecretScope::Workspace
- Category: second path segment of ks_path (e.g. "mokumo/supabase/url" → "supabase")
- Return WorkspaceSecrets { refs, unavailable_reason: None }; set unavailable_reason if `ks` not installed

Also add `mod env; mod secrets;` and pub use to src/sources/mod.rs.
Do NOT modify gather_snapshot() yet.

All errors must degrade gracefully (log with tracing::warn!, return empty struct). No panics.
Run cargo build to verify.
```

**Acceptance:**
- `cargo build` passes
- `EnvSource::gather` parses `export VAR=$(secret path)` and `export VAR=value` correctly
- `SecretsSource::gather` handles `ks not installed` without panic
- Module declarations added to `src/sources/mod.rs`

---

### Task 1.2: Git + GitHub Sources

**Topic:** `git-github-sources`
**Worktree:** `git worktree add ../.claude/worktrees/git-github-sources -b git-github-sources`
**Depends on:** `0.1-workspace-domain-types` merged
**Complexity:** Medium
**Files:**
- `src/sources/git.rs` — new file
- `src/sources/github.rs` — new file

**Steps:**
1. Create `src/sources/git.rs` with `pub struct GitSource;` and `impl GitSource { pub async fn gather(cwd: &str) -> Option<WorkspaceGit> }`.
   - Return `None` if git not available (check via `which git`) or no git repo at cwd.
   - Shell out with `tokio::process::Command` and `-C cwd` flag:
     - `git -C cwd rev-parse --abbrev-ref HEAD` → branch (returns "HEAD" for detached)
     - `git -C cwd status --porcelain` → dirty = stdout is non-empty
     - `git -C cwd worktree list --porcelain` → parse into `Vec<WorktreeInfo>` (each block: `worktree <path>`, `branch refs/heads/<branch>` or `detached`, `HEAD <hash>` + blank line separator; dirty check: run `git -C <path> status --porcelain` per worktree)
     - `git -C cwd stash list` → stash_count = line count
     - `git -C cwd rev-list --count @{u}..HEAD 2>/dev/null` → ahead (parse u64, default 0 on error)
     - `git -C cwd rev-list --count HEAD..@{u} 2>/dev/null` → behind (parse u64, default 0 on error)
   - All commands use `output()` with error handling — any command failure degrades gracefully (returns None or 0 for counts).

2. Create `src/sources/github.rs` with `pub struct GitHubSource;` and `impl GitHubSource { pub async fn gather(cwd: &str) -> Option<WorkspaceGithub> }`.
   - Return `None` if `gh` not installed (check via `which gh`).
   - Get repo slug: `git -C cwd remote get-url origin` → parse `github.com/owner/repo` or `git@github.com:owner/repo.git` → `"owner/repo"` string. Return None if not a GitHub remote.
   - Check auth: `gh auth status` exit code 0 = authed. If not authed: return `WorkspaceGithub { milestones: vec![], labels_on_open_issues: vec![], auth_available: false }`.
   - Fetch milestones: `gh api repos/{owner}/{repo}/milestones` → parse JSON array. Fields: `title` (String), `number` (u64), `open_issues` (u32), `closed_issues` (u32).
   - Fetch all open issues: `gh issue list --repo {owner}/{repo} --state open --json number,title,labels` → parse JSON array. `labels` is array of `{ "name": "..." }` objects.
   - Build `labels_on_open_issues`: collect all unique label names across all open issues.
   - For each milestone: filter the open issues to those belonging to that milestone by re-querying with `--milestone "{title}"` flag. This avoids loading all issue milestone metadata in the initial fetch.
   - Alternatively (simpler): fetch issues per milestone with `gh issue list --repo {owner}/{repo} --milestone "{title}" --state open --json number,title,labels`. Total calls: 1 (milestones) + N (one per milestone) + 1 (all open issues for labels). Acceptable at ~700ms total per spike findings.
   - Return `WorkspaceGithub { milestones, labels_on_open_issues, auth_available: true }`.

3. JSON parsing: use `serde_json::from_str` on command stdout (serde_json is already in Cargo.toml). Define local `#[derive(Deserialize)]` structs for the raw API shapes, then map to domain types.

4. Add `mod git; mod github;` and `pub use git::GitSource; pub use github::GitHubSource;` to `src/sources/mod.rs`. Do NOT yet wire into `gather_snapshot()`.

5. Run `cargo build`.

**Session Prompt:**
```
You are implementing two new Soji M1.5 data source modules. Soji is a Rust TUI (ratatui 0.29, tokio async, Rust 2024 edition). serde_json is already in Cargo.toml.

Read these files first:
- docs/workspace/m1.5-workspace-observability/shaping.md — Parts A4 and A5
- docs/workspace/m1.5-workspace-observability/spike-github.md — confirmed API shapes and latency
- src/domain/workspace.rs — WorkspaceGit, WorktreeInfo, WorkspaceGithub, Milestone, Issue types
- src/sources/mod.rs — existing pattern to follow

Task: Create src/sources/git.rs and src/sources/github.rs.

GitSource::gather(cwd: &str) -> Option<WorkspaceGit>:
- Shell out using tokio::process::Command with -C cwd flag
- Commands: rev-parse --abbrev-ref HEAD (branch), status --porcelain (dirty), worktree list --porcelain (parse worktrees), stash list (count lines), rev-list --count @{u}..HEAD and HEAD..@{u} (ahead/behind, best-effort default 0)
- Parse worktree list --porcelain: each block separated by blank line; extract path, branch (refs/heads/... or detached), then check dirty per worktree path
- Return None if git unavailable or cwd has no repo

GitHubSource::gather(cwd: &str) -> Option<WorkspaceGithub>:
- Get org/repo slug from `git remote get-url origin` — handle both https and ssh remote URL formats
- Check `gh auth status` exit code (0 = authed). If not authed, return WorkspaceGithub { auth_available: false, milestones: vec![], labels_on_open_issues: vec![] }
- Fetch milestones via `gh api repos/{owner}/{repo}/milestones` — JSON fields: title, number, open_issues, closed_issues (snake_case confirmed in spike)
- Fetch all open issues via `gh issue list --repo {owner}/{repo} --state open --json number,title,labels` — labels are objects { name: String, ... }
- For each milestone: fetch its issues via `gh issue list --repo {owner}/{repo} --milestone "{title}" --state open --json number,title,labels`
- Aggregate labels_on_open_issues: unique label names across all open issues
- Use serde_json + local #[derive(Deserialize)] structs for raw API shapes; map to domain types

All errors degrade gracefully (tracing::warn!, return None or empty structs). No panics.
Add mod declarations to src/sources/mod.rs but do NOT modify gather_snapshot() yet.
Run cargo build to verify.
```

**Acceptance:**
- `cargo build` passes
- `GitSource::gather` handles missing git, no repo, detached HEAD gracefully
- `GitHubSource::gather` handles missing `gh`, unauthenticated state, no GitHub remote gracefully
- JSON parsing uses serde_json with local deserialization structs

---

## Wave 2: Wiring + App Extensions

### Task 2.1: Config Source + gather_snapshot Wiring

**Topic:** `config-source-and-wiring`
**Worktree:** `git worktree add ../.claude/worktrees/config-source-and-wiring -b config-source-and-wiring`
**Depends on:** `1.1-env-secrets-sources` and `1.2-git-github-sources` merged
**Complexity:** Low
**Files:**
- `src/sources/config.rs` — new file
- `src/sources/mod.rs` — extend `gather_snapshot()`

**Steps:**
1. Create `src/sources/config.rs` with `pub struct ConfigSource;` and `impl ConfigSource { pub async fn scan(cwd: &str) -> ConfigFiles }`.
   - Use `std::fs::metadata` to check presence of: `.envrc`, `docker-compose.yml`, `docker-compose.yaml`, `soji.toml`, `Makefile`, `Cargo.toml`, `package.json`.
   - Glob `.env*` variants: check `.env`, `.env.local`, `.env.development`, `.env.production`, `.env.test` — collect into `dot_env_files: Vec<String>`.
   - Return `ConfigFiles` struct with all presence flags.
   - No file parsing — only presence checks.

2. Extend `gather_snapshot()` in `src/sources/mod.rs`:
   - Import the new source types: `EnvSource, SecretsSource, GitSource, GitHubSource, ConfigSource`.
   - After the existing `tokio::join!` block, add per-workspace gathering. For each workspace in `workspaces`:
     - Get the workspace `cwd` string (or skip if None).
     - Run N1 (EnvSource) first: `let env_data = EnvSource::gather(cwd).await;`
     - Then run N2–N5 concurrently: `let (secrets_data, git_data, github_data, config_data) = tokio::join!(SecretsSource::gather(cwd, &env_data.vars), GitSource::gather(cwd), GitHubSource::gather(cwd), ConfigSource::scan(cwd)).await;`
     - Assign: `workspace.env = Some(env_data); workspace.secrets = Some(secrets_data); workspace.git = git_data; workspace.github = github_data; workspace.config_files = Some(config_data);`
   - Wrap the per-workspace loop in a way that runs all workspaces concurrently (use `futures::future::join_all` or iterate sequentially — sequential is fine given typical 1-2 workspaces at M1.5 scale).
   - Note: `futures` crate is not in Cargo.toml. Use a `Vec<tokio::task::JoinHandle>` approach or just sequential iteration per workspace (simple, correct at M1.5 scale).

3. Update `tracing::info!` at end of `gather_snapshot()` to include any relevant observability counts.

4. Run `cargo build`.

**Session Prompt:**
```
You are implementing the config source and wiring all M1.5 sources into gather_snapshot(). Soji is a Rust TUI (ratatui 0.29, tokio async, Rust 2024 edition).

Read these files first:
- docs/workspace/m1.5-workspace-observability/shaping.md — Parts A6 and A7
- docs/workspace/m1.5-workspace-observability/breadboard.md — Flow 1 wiring diagram
- src/domain/workspace.rs — ConfigFiles type; Workspace struct (all 5 new optional fields)
- src/sources/mod.rs — current gather_snapshot() to extend
- src/sources/env.rs, src/sources/secrets.rs, src/sources/git.rs, src/sources/github.rs — source modules to call

Task: Create src/sources/config.rs and extend gather_snapshot() in src/sources/mod.rs.

ConfigSource::scan(cwd: &str) -> ConfigFiles:
- std::fs::metadata checks for: .envrc, docker-compose.yml, docker-compose.yaml, soji.toml, Makefile, Cargo.toml, package.json
- Check .env variants: .env, .env.local, .env.development, .env.production, .env.test; collect present ones into dot_env_files: Vec<String>

gather_snapshot() extension:
- After the existing tokio::join! block, iterate over workspaces
- For each workspace with a cwd: first run EnvSource::gather(cwd).await (sequential — its output feeds SecretsSource)
- Then run SecretsSource, GitSource, GitHubSource, ConfigSource concurrently via tokio::join!
- Assign results to workspace.env, workspace.secrets, workspace.git, workspace.github, workspace.config_files
- Keep iteration sequential over workspaces (no futures::join_all needed for M1.5 scale)

The `futures` crate is NOT available — use sequential workspace iteration.
All existing behavior is preserved; only additions are made.
Run cargo build to verify.
```

**Acceptance:**
- `cargo build` passes
- `ConfigFiles` has all presence flags + `dot_env_files`
- `gather_snapshot()` populates all 5 workspace fields
- N1 (EnvSource) runs before N2 (SecretsSource) — sequential, not concurrent
- N2–N5 run concurrently via `tokio::join!`
- Existing snapshot behavior for processes/docker/cmux unchanged

---

### Task 2.2: App + Events Extensions

**Topic:** `app-and-events-extensions`
**Worktree:** `git worktree add ../.claude/worktrees/app-and-events-extensions -b app-and-events-extensions`
**Depends on:** `0.1-workspace-domain-types` merged
**Complexity:** High
**Files:**
- `src/tui/app.rs` — extend `NavItem`, `SectionKind`, `toggle_expand()`, `update_nav_workspace()`
- `src/tui/events.rs` — extend Enter key dispatch

**Steps:**
1. Extend `SectionKind` enum in `app.rs`:
   - Add `Github` and `Config` variants.
   - Add `fn key(&self) -> &str` method: `Env → "section:env"`, `Secrets → "section:secrets"`, `Git → "section:git"`, `Github → "section:github"`, `Config → "section:config"`.

2. Extend `NavItem` enum in `app.rs`:
   - Add: `EnvVarRow(usize)`, `SecretRow(usize)`, `WorktreeRow(usize)`, `MilestoneRow(usize)`, `IssueRow(usize, usize)` (milestone_idx, issue_idx), `ConfigFileRow(String)`.

3. Extend `toggle_expand()` in `app.rs`:
   - Current: only handles `NavItem::Surface(ref_id)`.
   - Extend to also handle `NavItem::SectionHeader(kind)` — key = `kind.key()` (e.g., `"section:env"`).
   - Extend to also handle `NavItem::MilestoneRow(idx)` — key = `format!("milestone:{}", idx)`.
   - For all handled cases: toggle key in `self.expanded`, call `self.update_nav()`, clamp cursor.
   - Pattern: check current cursor item and match on multiple variants.

4. Extend `update_nav_workspace()` in `app.rs`:
   - After adding the three existing `SectionHeader` items, add `SectionHeader(SectionKind::Github)` and `SectionHeader(SectionKind::Config)`.
   - After each `SectionHeader`, check `self.expanded.contains(kind.key())`. If expanded, emit child rows:
     - `SectionKind::Env` → for each var in `workspace.env.as_ref().map(|e| &e.vars).unwrap_or(&[])`: push `NavItem::EnvVarRow(i)`.
     - `SectionKind::Secrets` → for each ref in `workspace.secrets.as_ref().map(|s| &s.refs).unwrap_or(&[])`: push `NavItem::SecretRow(i)`.
     - `SectionKind::Git` → for each worktree in `workspace.git.as_ref().map(|g| &g.worktrees).unwrap_or(&[])`: push `NavItem::WorktreeRow(i)`.
     - `SectionKind::Github` → for each (i, milestone) in milestones: push `NavItem::MilestoneRow(i)`; if `self.expanded.contains(&format!("milestone:{}", i))`: for each (j, _issue) in milestone.issues: push `NavItem::IssueRow(i, j)`.
     - `SectionKind::Config` → for each file that is present (iterate ConfigFiles fields): push `NavItem::ConfigFileRow(filename.to_string())`.

5. Update `events.rs` Enter key handling:
   - Current: calls `app.toggle_expand()` for all non-Global Enter presses.
   - The existing `toggle_expand()` now handles `SectionHeader` and `MilestoneRow` items, so the Enter key path in events.rs does NOT need to change — `app.toggle_expand()` already dispatches correctly via the current-cursor item.
   - Verify: the existing `KeyCode::Enter` branch already calls `app.toggle_expand()` for Workspace mode. This remains correct — toggle_expand now handles all expandable NavItems.
   - Add `SectionKind::Github` and `SectionKind::Config` to any `match` arms on `SectionKind` in events.rs that need exhaustiveness (add `_ => {}` or explicit arms as needed).

6. Fix match exhaustiveness in `right.rs` and `left.rs` for the new `NavItem` and `SectionKind` variants — the compiler will flag these. Add `_ => {}` or placeholder arms as needed. These will be properly filled in by Task 3.1 and 3.2.

7. Run `cargo build` — must compile. Expect some `#[allow(unused_variables)]` may be needed temporarily; use `_ =>` arms in match exhaustiveness rather than suppressing warnings.

**Session Prompt:**
```
You are implementing the M1.5 app state and event extensions for Soji. Soji is a Rust TUI (ratatui 0.29, tokio async, Rust 2024 edition).

Read these files first:
- docs/workspace/m1.5-workspace-observability/shaping.md — Parts A8, A9, A11
- docs/workspace/m1.5-workspace-observability/breadboard.md — Flows 2 and 4, and the NavItem/SectionKind affordance tables
- src/tui/app.rs — FULL file; understand existing toggle_expand(), update_nav_workspace(), NavItem, SectionKind
- src/tui/events.rs — FULL file; understand existing Enter key dispatch
- src/tui/ui/left.rs — note the build_list_item match; you must add arms for new NavItem variants (can be placeholder ListItem::new(Line::default()) for now)
- src/tui/ui/right.rs — note the draw_right match; you must add arms for new NavItem variants (can be empty vec![] for now)

Task: Extend src/tui/app.rs and src/tui/events.rs.

Changes to src/tui/app.rs:
1. Add to SectionKind enum: Github, Config variants. Add key(&self) -> &str method.
2. Add to NavItem enum: EnvVarRow(usize), SecretRow(usize), WorktreeRow(usize), MilestoneRow(usize), IssueRow(usize, usize), ConfigFileRow(String).
3. Extend toggle_expand(): handle SectionHeader(kind) → key = kind.key(); handle MilestoneRow(idx) → key = "milestone:{idx}". Keep existing Surface handling.
4. Extend update_nav_workspace(): add Github and Config section headers; emit child rows when each section is expanded. Read from workspace.env, workspace.secrets, workspace.git, workspace.github, workspace.config_files (all Option<T>; use .as_ref().map(...).unwrap_or_default() or similar).

Changes to src/tui/ui/left.rs and src/tui/ui/right.rs:
- Add match arms for new NavItem variants to satisfy Rust exhaustiveness. Use placeholder renderings (empty Line or empty vec![]) — these will be properly implemented in separate sessions.

Changes to src/tui/events.rs:
- The existing Enter handler calls app.toggle_expand() for Workspace mode — this remains correct because toggle_expand() now handles SectionHeader and MilestoneRow. No logic change needed.
- Fix any SectionKind match exhaustiveness (Github, Config arms).

Run cargo build to verify. All new variants must compile; placeholder renderings are fine.
```

**Acceptance:**
- `cargo build` passes
- `SectionKind` has 5 variants with `key()` method
- `NavItem` has 6 new variants
- `toggle_expand()` handles `SectionHeader`, `MilestoneRow`, and `Surface` cases
- `update_nav_workspace()` emits child rows when sections are expanded
- All match exhaustiveness resolved (no compiler errors)

---

## Wave 3: Rendering

### Task 3.1: Left Panel Rendering

**Topic:** `left-panel-rendering`
**Worktree:** `git worktree add ../.claude/worktrees/left-panel-rendering -b left-panel-rendering`
**Depends on:** `2.2-app-and-events-extensions` merged
**Complexity:** Medium
**Files:**
- `src/tui/ui/left.rs` — implement all new NavItem variant renderers + real section header badges

**Steps:**
1. Replace `section_header_item()` with a version that takes `&App` context to read real data:
   ```rust
   fn section_header_item(kind: &SectionKind, app: &App, selected: bool) -> ListItem<'static>
   ```
   - Update `build_list_item()` to pass `app` to `section_header_item`.
   - Badge text per section kind:
     - `Env`: read `app.workspace.as_ref().and_then(|w| w.env.as_ref())` → "N vars · M missing" (count `!var.loaded`). If `envrc_present = false`: "no .envrc". If `None`: "—".
     - `Secrets`: "N entries · M missing" (count `!loaded`). If unavailable_reason is Some: "unavailable". If `None`: "—".
     - `Git`: show `branch · [dirty]` or `—`. If `None` → "unavailable".
     - `Github`: "N milestones" or "auth required" (check `auth_available`). If `None`: "—".
     - `Config`: "N files present" (count present fields). If `None`: "—".
   - Show `∨` when `app.expanded.contains(kind.key())`, else `›`.
   - Section header has `BOLD` label, dim badge text, muted toggle indicator.

2. Implement `env_var_row(idx: usize, app: &App, selected: bool) -> ListItem<'static>`:
   - Read `app.workspace.as_ref().and_then(|w| w.env.as_ref()).and_then(|e| e.vars.get(idx))`.
   - If None: return empty ListItem.
   - Format: `[✓/✗]` (green if loaded, red if not), `VAR_NAME` (bold if selected), source abbreviation (`direnv`/`dotenv`/`shell`), `🔑` icon if `ks_path.is_some()`.
   - Status icon: `✓` in green for loaded, `✗` in red for missing.

3. Implement `secret_row(idx: usize, app: &App, selected: bool) -> ListItem<'static>`:
   - Read `workspace.secrets.refs[idx]`.
   - Three-state: `✓` (loaded, green), `✗` (missing from keychain, red), `⚠` (in keychain but not loaded, yellow).
   - Show: status icon, `ks_path`, scope badge (`[shared]` in blue or `[ws]` in muted).

4. Implement `worktree_row(idx: usize, app: &App, selected: bool) -> ListItem<'static>`:
   - Read `workspace.git.worktrees[idx]`.
   - Format: `●` bullet (red if dirty, green if clean), path (shortened), branch or "detached".

5. Implement `milestone_row(idx: usize, app: &App, selected: bool) -> ListItem<'static>`:
   - Read `workspace.github.milestones[idx]`.
   - Format: milestone title (bold), `N open / M closed`, toggle `∨`/`›` (check `expanded.contains("milestone:N")`).

6. Implement `issue_row(m_idx: usize, i_idx: usize, app: &App, selected: bool) -> ListItem<'static>`:
   - Read `workspace.github.milestones[m_idx].issues[i_idx]`.
   - Format: `  #N title  [label]` (indented under milestone). Show first label chip if any.

7. Implement `config_file_row(filename: &str, app: &App, selected: bool) -> ListItem<'static>`:
   - Format: `✓ filename` (green) if present, or `— filename` (dim) — all listed files shown.

8. Update `build_list_item()` match arms to call the new functions.

9. Run `cargo build`.

**Session Prompt:**
```
You are implementing M1.5 left panel rendering for Soji. Soji is a Rust TUI (ratatui 0.29, crossterm, tokio, Rust 2024 edition).

Read these files first:
- docs/workspace/m1.5-workspace-observability/shaping.md — Part A9 (left panel rendering spec)
- docs/workspace/m1.5-workspace-observability/breadboard.md — U1–U12 affordances
- src/tui/ui/left.rs — FULL file; you are replacing section_header_item and adding new functions
- src/tui/app.rs — NavItem, SectionKind, App struct; new variants from Task 2.2
- src/domain/workspace.rs — all M1.5 domain types (WorkspaceEnv, WorkspaceSecrets, etc.)

Task: Implement all new NavItem variant renderers in src/tui/ui/left.rs.

Color palette (already defined in the file):
- BG = Rgb(26,27,38), SEL_BG = Rgb(47,51,77), PRI = Rgb(192,202,245)
- DIM = Rgb(86,95,137), MUT = Rgb(59,66,97), BLU = Rgb(122,162,247), GRN = Rgb(158,206,106)
- Add: RED = Rgb(255,100,100), YEL = Rgb(224,175,104) if not present

1. Replace section_header_item() with fn section_header_item(kind: &SectionKind, app: &App, selected: bool) -> ListItem<'static>:
   - Update build_list_item() to pass app and selected to section_header_item
   - Real badge text per section (read from app.workspace.*): see shaping Part A9 for spec
   - ∨ when expanded (check app.expanded.contains(kind.key())), › when collapsed
   - BOLD label, dim badge, muted toggle

2. Implement rendering functions for each new NavItem variant:
   - EnvVarRow(idx): [✓/✗] in green/red + VAR_NAME + source + 🔑 if secret-backed
   - SecretRow(idx): three-state icon (✓ loaded/✗ not in keychain/⚠ in keychain not loaded) + ks_path + [shared]/[ws] badge
   - WorktreeRow(idx): ● (red=dirty, green=clean) + shortened path + branch
   - MilestoneRow(idx): title + "N open / M closed" + ∨/› toggle indicator
   - IssueRow(m_idx, i_idx): indented "  #N title  [label]"
   - ConfigFileRow(filename): ✓ filename (green) or — filename (dim)

3. Update build_list_item() match arms to call new functions.

All new functions must handle None/out-of-bounds gracefully (return empty ListItem if data not found).
Run cargo build to verify.
```

**Acceptance:**
- `cargo build` passes
- Section headers show real badge text (not hardcoded "12 vars")
- All 6 new `NavItem` variants have rendering functions
- Status icons use correct colors (green=ok, red=missing, yellow=warning)
- All out-of-bounds/None cases return gracefully

---

### Task 3.2: Right Panel Rendering

**Topic:** `right-panel-rendering`
**Worktree:** `git worktree add ../.claude/worktrees/right-panel-rendering -b right-panel-rendering`
**Depends on:** `2.2-app-and-events-extensions` merged
**Complexity:** High
**Files:**
- `src/tui/ui/right.rs` — implement all new section detail views

**Steps:**
1. Replace `section_detail()` with a full dispatch based on cursor item:
   - Extend `draw_right()` match to handle all new `NavItem` variants.
   - `NavItem::EnvVarRow(i)` → `env_var_detail(i, app)`: single var detail (name, source, ks_path, loaded status, sensitive flag, "ks add <path>" hint if not in keychain).
   - `NavItem::SecretRow(i)` → `secret_detail(i, app)`: ks_path, scope, category, exists_in_keychain, loaded; "ks add <path>" hint if missing.
   - `NavItem::WorktreeRow(i)` → `worktree_detail(i, app)`: path, branch, dirty/clean.
   - `NavItem::MilestoneRow(i)` → `milestone_detail(i, app)`: milestone title + progress + issue list.
   - `NavItem::IssueRow(m, i)` → `issue_detail(m, i, app)`: full issue detail (#number, title, all labels).
   - `NavItem::ConfigFileRow(name)` → `config_file_detail(name, app)`: if `.envrc` → envrc_viewer, else just presence info.

2. Replace `section_detail(kind)` with real data views:
   - `SectionKind::Env` → `env_section_detail(app)`: table of all declared vars (name, source, ks_path if any, loaded ✓/✗, sensitive indicator). Header: "ENV · N vars · M missing". Group by loaded/missing (missing first or all together — keep simple).
   - `SectionKind::Secrets` → `secrets_section_detail(app)`: manifest grouped: shared first, then workspace. Three-state per row. Header: "SECRETS". If unavailable: show reason.
   - `SectionKind::Git` → `git_section_detail(app)`: branch + dirty status + ahead/behind + stash count + worktrees table. If None: "git unavailable".
   - `SectionKind::Github` → `github_section_detail(app)`: milestone summary table (title, open/closed counts) + labels panel below ("Labels in use: label1  label2  …"). If auth_available = false: "gh auth required — run `gh auth login`".
   - `SectionKind::Config` → `config_section_detail(app)`: file list with ✓/— per file.

3. `envrc_viewer(app)` → read `.envrc` contents from disk with `std::fs::read_to_string`. Display as syntax-highlighted lines (export lines in PRI, comment lines in DIM, blank lines as-is). No value redaction needed (`.envrc` has no secret values — only `$(secret path)` references).

4. `empty_state(reason: &str)` → helper for graceful degradation views ("no .envrc found", "gh auth required", "git not found", etc.).

5. Update `draw_right()` dispatch to handle new `NavItem` variants. Current match is:
   ```rust
   Some(NavItem::Surface(ref_id)) => surface_detail(ref_id, app),
   Some(NavItem::Process(pid)) => process_detail(*pid, app),
   Some(NavItem::BrowserSurface(ref_id)) => browser_detail(ref_id, app),
   Some(NavItem::SectionHeader(kind)) => section_detail(kind),
   Some(NavItem::GlobalRow) => vec![],
   ```
   Add arms for all 6 new `NavItem` variants. Replace `section_detail(kind)` call with the new data-driven dispatch.

6. Run `cargo build`.

**Session Prompt:**
```
You are implementing M1.5 right panel rendering for Soji. Soji is a Rust TUI (ratatui 0.29, crossterm, tokio, Rust 2024 edition).

Read these files first:
- docs/workspace/m1.5-workspace-observability/shaping.md — Part A10 (right panel rendering spec)
- docs/workspace/m1.5-workspace-observability/breadboard.md — U20–U27 affordances, Flow 3 and Flow 5 wiring
- src/tui/ui/right.rs — FULL file; you are extending section_detail() and draw_right()
- src/tui/app.rs — NavItem, SectionKind, App struct; new variants from Task 2.2
- src/domain/workspace.rs — all M1.5 domain types

Task: Implement all new section detail views and child row detail views in src/tui/ui/right.rs.

Color palette already defined: BG, PRI, DIM, MUT, BLU, GRN, YEL. Add RED = Rgb(255,100,100).

1. Extend draw_right() match to handle new NavItem variants: EnvVarRow, SecretRow, WorktreeRow, MilestoneRow, IssueRow, ConfigFileRow. Each calls a detail function.

2. Replace section_detail(kind) stub with real implementations:
   - SectionKind::Env → table of all declared vars (name, source, loaded ✓/✗, ks_path if any, sensitive flag). Missing vars highlighted in red. If no .envrc: empty_state("No .envrc found").
   - SectionKind::Secrets → manifest grouped by scope (shared first). Three-state per row (✓ loaded, ✗ not in keychain, ⚠ in keychain but not loaded). If unavailable: show reason.
   - SectionKind::Git → branch name (bold), ahead/behind, dirty/clean, stash count, then worktrees table. If None: empty_state("git unavailable").
   - SectionKind::Github → milestone summary table (title | open | closed) + labels panel below listing all labels_on_open_issues. If !auth_available: empty_state("gh auth required — run `gh auth login`").
   - SectionKind::Config → file list with ✓ (green) or — (dim) per file.

3. Child row detail views:
   - EnvVarRow(i): var name (bold), source, loaded status, ks_path (if any), "ks add <path>" hint if not loaded and ks_path present
   - SecretRow(i): ks_path (bold), scope badge, category, exists_in_keychain status, loaded status, "ks add <path>" hint if missing
   - WorktreeRow(i): path, branch, dirty status
   - MilestoneRow(i): milestone title (bold), progress ("N open · M closed"), then full issue list below
   - IssueRow(m,i): #number (bold blue), title, all label chips
   - ConfigFileRow(name): if name == ".envrc" and present → read file with std::fs::read_to_string and display contents (export lines in PRI, comment lines in DIM). If absent: empty_state("no .envrc found").

4. Helper: fn empty_state(reason: &str) -> Vec<Line<'static>> — centered dim text.

All data accesses must be bounds-safe. Use .get() not direct indexing.
Run cargo build to verify.
```

**Acceptance:**
- `cargo build` passes
- `SectionKind::Env` shows real env var table (not "Environment variables coming in M1.5")
- `SectionKind::Github` shows milestone table + labels panel
- `.envrc` content viewer reads the actual file
- All graceful degradation states render (no panics on None data)
- All 6 new `NavItem` variants have detail views

---

## Session Sizing

| Wave | Session | Complexity | Files | Breadboard Parts |
|------|---------|------------|-------|-----------------|
| W0   | 0.1 domain types | Medium | 1 | A1 |
| W1   | 1.1 env+secrets | Medium | 2+partial mod | A2, A3 |
| W1   | 1.2 git+github | Medium | 2+partial mod | A4, A5 |
| W2   | 2.1 config+wiring | Low | 2 | A6, A7 |
| W2   | 2.2 app+events | High | 2 | A8, A9, A11 |
| W3   | 3.1 left rendering | Medium | 1 | A9 |
| W3   | 3.2 right rendering | High | 1 | A10 |

---

## Dependency DAG

```
0.1 (domain types)
  ├─► 1.1 (env+secrets)
  ├─► 1.2 (git+github)
  └─► 2.2 (app+events) ─────────────────────► 3.1 (left)
                                            └─► 3.2 (right)
       1.1 ──────────┐
       1.2 ──────────┼─► 2.1 (config+wiring)
```

---

## Critical Path

`W0/0.1` → `W1/1.1` → `W2/2.1` → `W2/2.2` → `W3/3.2`

(4 sequential gates; parallel work in W1, W2, W3 reduces wall-clock time)

---

## Merge Strategy

| Session | Files owned | Conflicts? |
|---------|-------------|------------|
| 0.1 | domain/workspace.rs | None — only file touched |
| 1.1 | sources/env.rs (new), sources/secrets.rs (new), sources/mod.rs (mod declarations only) | Low risk — new files + 2 lines in mod.rs |
| 1.2 | sources/git.rs (new), sources/github.rs (new), sources/mod.rs (mod declarations only) | Low risk — 1.1 and 1.2 both touch mod.rs header (mod declarations) — ensure both add their declarations without stomping each other. Merge 1.1 first, rebase 1.2 on top. |
| 2.1 | sources/config.rs (new), sources/mod.rs (gather_snapshot body) | Merge 1.1 and 1.2 first; 2.1 adds to the body of gather_snapshot |
| 2.2 | tui/app.rs, tui/events.rs | No conflict — different files from 2.1 |
| 3.1 | tui/ui/left.rs | No conflict — only file |
| 3.2 | tui/ui/right.rs | No conflict — only file |

**Note on mod.rs:** Sessions 1.1 and 1.2 both add `mod` declarations to `src/sources/mod.rs` header. Merge one first, then rebase/merge the other. The declarations are distinct lines; conflict is trivially resolved.

---

## Vertical Slice Coverage

| Slice | Sessions |
|-------|---------|
| VS1: ENV section | 0.1, 1.1, 2.1, 2.2, 3.1, 3.2 |
| VS2: SECRETS section | 0.1, 1.1, 2.1, 2.2, 3.1, 3.2 |
| VS3: GIT section | 0.1, 1.2, 2.1, 2.2, 3.1, 3.2 |
| VS4: GITHUB section | 0.1, 1.2, 2.1, 2.2, 3.1, 3.2 |
| VS5: CONFIG section | 0.1, 2.1, 2.2, 3.1, 3.2 |

All 5 slices are covered. Each slice requires the full stack (domain → source → wiring → app → rendering).

---

## Notes

- **No new crates needed.** `serde_json` already in Cargo.toml. `.envrc` parsing uses string methods (`contains`, `starts_with`, `split`) — no regex crate needed.
- **Edition 2024 `let` chains.** The existing codebase uses `let Some(x) = ... && let Y = ...` syntax throughout. New code should follow this pattern.
- **`toggle_expand()` signature stays `(&mut self)`.** It reads `self.cursor` internally — no args needed. The Enter key dispatch in events.rs already calls `app.toggle_expand()` for Workspace mode; this remains correct after Task 2.2.
- **`.envrc` viewer reads from disk at render time.** Not cached — acceptable at M1.5 scale. File path: `workspace.cwd + "/.envrc"`.
- **GitHub issue-per-milestone fetch.** Task 1.2 calls `gh issue list --milestone "{title}"` once per milestone. At typical M1.5 scale (< 10 milestones), total latency ≈ 700ms as confirmed by spike.
- **Worktree dirty check.** `git worktree list --porcelain` gives paths but not dirty status per worktree. Task 1.2 must run `git -C <worktree_path> status --porcelain` per worktree. Acceptable performance for < 5 worktrees typical.
- **Snapshot refresh.** The existing watch-channel refresh loop calls `gather_snapshot()` on a timer. After Task 2.1, each refresh populates workspace observability data automatically — no changes to the refresh loop needed.

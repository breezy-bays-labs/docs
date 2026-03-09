---
pipeline: m1.5-workspace-observability
stage: bb-reflection
---

# Breadboard Reflection: Soji M1.5 — Workspace Observability

## Smell Audit

### Phase 1: Requirements trace (breadboard + requirements only)

**Tracing Journey A (Workspace Readiness Check):**
Path: launch in workspace CWD → `gather_snapshot()` (N6) → N1/N3 run → `update_snapshot()` (N14) → `update_nav_workspace()` (N10) → `draw_left()` (N16) renders ENV/GIT headers with badge text → user navigates to ENV header → Enter → `toggle_expand()` (N11) → child EnvVarRows emitted → `draw_left()` renders rows (U7) → right panel (N17) shows ENV detail table (U20).

Trace is coherent. Each step leads to the next. Badge text on ENV header shows "N vars · M missing" at a glance.
**Result: Coherent.**

**Tracing Journey B (Why is my code broken — missing env var):**
Path: navigate to ENV section → expand → navigate to missing var row (U7, ✗) → `draw_right()` (N17) shows var detail with `ks_path`, source, loaded status → developer sees "declared but not loaded; entry doesn't exist in keychain."

Flow works. However: U13 "Wires Out: → N10" — navigating should not trigger `update_nav_workspace()`. Navigation increments the cursor (S8), not the nav list. **Smell detected: U13 has wrong Wires Out. Should wire to N12 (cursor move), not N10 (nav rebuild).** The Mermaid already shows `U13 --> S8` directly, inconsistent with the table's `→ N10`.

**Tracing Journey D (GitHub state):**
Path: navigate to GITHUB section header → Enter → `toggle_expand()` → MilestoneRows emitted → `draw_left()` renders U10 rows → navigate to MilestoneRow → right panel shows milestone detail (U24, issue list with labels) → expand milestone → IssueRows emitted (U11) → cursor on IssueRow → right panel shows issue detail.

Enter on a MilestoneRow: U16 "Enter on MilestoneRow" wires directly to N11 (toggle_expand), bypassing N15 (handle_enter). U14 "Enter on SectionHeader" also wires directly to N11, bypassing N15. N15 is the declared event dispatcher — all Enter key events should route through it. **Smell detected: U14 and U16 bypass N15. Two Enter affordances should both route through handle_enter().**

Additionally, U16 "Enter on MilestoneRow — also expands issue list" is the same key event as U14 with the same ultimate mechanism (toggle_expand). The distinction (SectionHeader vs MilestoneRow) belongs inside N15's dispatch logic, not as separate UI affordances. **Smell detected: U16 is a redundant affordance. Merge into U14 as "Enter on expandable NavItem."**

**Tracing Journey E (Labels aggregate view):**
Path: navigate to GITHUB section → open → right panel (U23) should show milestone list + labels-on-open-issues panel.

U23 is defined as: "GITHUB section detail: milestone list with open/closed counts + progress indicators." No mention of labels. N4 gathers `labels_on_open_issues: Vec<String>` and stores it in S4. But there is no UI affordance wired to display this aggregate. U24 (Milestone detail) shows labels on individual issues — that is not the aggregate labels panel. R4 explicitly requires "labels view showing only labels that appear on currently-open issues." Journey E's "Labels panel shows 8 distinct labels on open issues" has no path to a rendering affordance. **Smell detected: Missing path — labels aggregate view is not wired. U23 needs to include the labels panel, or a separate U23b is needed.**

**Tracing Journey C (Secret provisioning — Enter on leaf rows):**
Path: navigate to SECRETS section → expand → navigate to a SecretRow (U8) → detail shown in right panel (N17 reads S2) → developer sees "ks add <path>" hint.

The right panel updates automatically as the cursor moves. Enter on a SecretRow (U15) "→ N12 (cursor state update)" is wired to cursor movement, not to any meaningful action. The right panel is already showing the detail before Enter is pressed. Enter on a leaf row does nothing new. **Smell detected: U15 is a phantom affordance. Enter on leaf rows has no distinct effect beyond cursor position. N15 (handle_enter) already declares "all other rows → detail driven by cursor position (right panel re-renders automatically)." U15 should be removed.**

**Tracing Flow 1 (N2 dependency on N1):**
N6 (`gather_snapshot()`) description: "Adds N1–N5 to `tokio::join!`. Shares EnvSource output with SecretsSource (avoids double `.envrc` parse)."

N2's signature is `SecretsSource::gather(cwd, env_vars)` — it takes the env_vars output of N1 as input. These two cannot run concurrently. N1 must complete before N2 can start. The wiring diagram (`Flow 1`) shows `N6` fanning out to `N1 & N2 & N3 & N4 & N5` concurrently. The Mermaid also shows `N6 --> N1 & N2 & N3 & N4 & N5`. **Smell detected: Incoherent wiring — N1 and N2 cannot be concurrent. N1 must run first; then N2, N3, N4, N5 can run concurrently.**

**Tracing Mermaid vs. Tables — S1–S5 to N16:**
Data Stores table descriptions for S1–S5 all say "Read by N10, N16, N17." N16 (draw_left) needs workspace data to render the section header badge texts — e.g., "ENV  12 vars · 2 missing" requires reading `workspace.env` (S1). But the Mermaid shows `S1 & S2 & S3 & S4 & S5 -.-> N10` and `S1 & S2 & S3 & S4 & S5 -.-> N17`, with no wire to N16. **Smell detected: Mermaid missing wires — S1–S5 → N16 not shown. Tables are correct; diagram is incomplete.**

---

### Naming Test

**Code Affordances:**

| Affordance | Caller | Step-level effect | One verb? |
|------------|--------|-------------------|-----------|
| N1: `EnvSource::gather()` | N6 | Parse .envrc and cross-reference env | gather — pass |
| N2: `SecretsSource::gather()` | N6 (sequential) | Enumerate ks refs and cross-reference state | gather — pass |
| N3: `GitSource::gather()` | N6 | Shell out git commands | gather — pass |
| N4: `GitHubSource::gather()` | N6 | Shell out gh CLI commands | gather — pass |
| N5: `ConfigSource::scan()` | N6 | Check file presence | scan — pass |
| N6: `gather_snapshot()` | watch channel | Orchestrate all sources | gather — pass |
| N10: `update_nav_workspace()` | N14, N11 | Rebuild nav list from workspace data + expanded state | rebuild — pass |
| N11: `toggle_expand()` | N15 | Toggle key in expanded set + rebuild nav | toggle — pass |
| N12: "Cursor state update" | U13 | Move cursor in nav list | **FAIL — "cursor state update" is not a verb phrase. Rename to `move_cursor(delta)`.** |
| N14: `App::update_snapshot()` | watch channel | Store new snapshot and rebuild nav | update — pass |
| N15: `handle_enter()` | events.rs | Dispatch Enter key to correct handler | dispatch (handle) — pass |
| N16: `draw_left()` | ratatui render loop | Render left nav panel | draw — pass |
| N17: `draw_right()` | ratatui render loop | Render right detail panel | draw — pass |

---

## Fixes

### Fix 1: N1 must precede N2 (incoherent wiring)

N2 takes `env_vars: &[EnvVar]` from N1's output. They cannot run concurrently. The gather sequence is:

```
N6: gather_snapshot()
    ├──► N1: EnvSource::gather(cwd)          → WorkspaceEnv
    │         ↓ (env_vars passed to N2)
    ├──► N2: SecretsSource::gather(cwd, &env_vars)  → WorkspaceSecrets
    │
    │    (then concurrently:)
    ├──► N3: GitSource::gather(cwd)           → Option<WorkspaceGit>
    ├──► N4: GitHubSource::gather(cwd)        → Option<WorkspaceGithub>
    └──► N5: ConfigSource::scan(cwd)          → ConfigFiles
         (N3, N4, N5 run concurrently via tokio::join! after N1 completes)
```

Updated N6 description: "Runs N1 first (synchronously in the async context). Passes N1 output to N2. Then runs N2, N3, N4, N5 concurrently via `tokio::join!(N2, N3, N4, N5)`."

Updated Flow 1 wiring diagram:
```
gather_snapshot()  ←N6
    ──► EnvSource::gather(cwd)              N1  ──► WorkspaceEnv          ──► workspace.env     S1
          │ env_vars
          ▼
    (then concurrently:)
    ├──► SecretsSource::gather(cwd, env_vars) N2  ──► WorkspaceSecrets     ──► workspace.secrets S2
    ├──► GitSource::gather(cwd)              N3  ──► Option<WorkspaceGit>  ──► workspace.git     S3
    ├──► GitHubSource::gather(cwd)           N4  ──► Option<WorkspaceGithub> ──► workspace.github S4
    └──► ConfigSource::scan(cwd)             N5  ──► ConfigFiles           ──► workspace.config  S5
         (N2–N5 run via tokio::join! after N1 returns)
```

### Fix 2: Remove U15 (phantom affordance)

U15 "Enter on EnvVar/Secret/Worktree/Milestone/Issue/Config row — show detail → N12" is a no-op. The right panel renders from cursor position automatically. N15 already documents this: "all other rows → detail is driven by cursor position (right panel re-renders automatically)."

**Remove U15 from the UI Affordances table.** No new behavior to implement.

### Fix 3: Route U14 and U16 through N15 (bypassed handler)

U14 and U16 both wire directly to N11, bypassing the declared event dispatcher N15.

**Fix:** U14 → N15 → N11 (N15 dispatches to toggle_expand for SectionHeader items).

### Fix 4: Remove U16 (redundant affordance)

U16 "Enter on MilestoneRow" is the same key event as U14 with the same mechanism. The distinction (SectionHeader vs MilestoneRow) lives inside N15's dispatch:

```
N15: handle_enter()
    match selected_item():
        SectionHeader(kind)  → toggle_expand(SectionHeader(kind))  N11
        MilestoneRow(i)      → toggle_expand(MilestoneRow(i))      N11
        _                    → no-op (detail shown by cursor)
```

**Remove U16.** Rename U14 to "Enter on expandable NavItem (SectionHeader or MilestoneRow)" to make its scope clear.

### Fix 5: Fix U13 Wires Out (wrong target)

U13 "Wires Out: → N10" is wrong. Navigation moves the cursor, not the nav list. Cursor movement writes to S8, triggering a re-render. Nav list is only rebuilt by N10 on expand/collapse events.

**Fix:** U13 Wires Out: → N12 (renamed `move_cursor(delta)`).

### Fix 6: Rename N12 to `move_cursor(delta)` (fails naming test)

"Cursor state update" is not a step-level verb phrase. The affordance moves the cursor in the nav list and writes to S8.

**Fix:** N12 renamed to `move_cursor(delta)`. Description: "Increments or decrements `app.cursor` by delta, clamped to `[0, app.nav.len() - 1]`. Writes to S8."

### Fix 7: Add labels aggregate to U23 (missing path for Journey E / R4)

N4 gathers `labels_on_open_issues: Vec<String>` (stored in S4). But U23 only shows "milestone list with progress indicators" — no labels panel. R4 requires "labels view showing only labels that appear on currently-open issues." Journey E's aggregate labels view has no display affordance.

**Fix:** Expand U23 to include the labels aggregate panel: "GITHUB section detail: milestone list with open/closed counts + progress indicators; **labels panel below: labels in use across all open issues**."

The labels panel is a subordinate section within the GITHUB right panel — not a separate place. When GITHUB section header is selected, the right panel shows both the milestone summary table and the active-labels panel.

### Fix 8: Add S1–S5 → N16 wires to Mermaid (diagram incomplete)

The tables correctly state N16 reads S1–S5 (needed to render badge texts). The Mermaid is missing these wires.

**Fix:** Add `S1 & S2 & S3 & S4 & S5 -.-> N16` to the Mermaid diagram.

---

## Verification

Re-tracing Journey A after fixes:
- N6 runs N1 first → passes env_vars to N2 → N2/N3/N4/N5 concurrent ✓ (Fix 1)
- U13 → N12 (move_cursor) → S8 → N17 re-renders right panel ✓ (Fixes 5+6)
- U14 (Enter on ENV header) → N15 → N11 → toggle_expand → N10 → nav rebuilt ✓ (Fix 3)
- U15 removed — no phantom affordance ✓ (Fix 2)

Re-tracing Journey D after fixes:
- U14 (Enter on MilestoneRow) → N15 → N11 → toggle_expand(MilestoneRow) → IssueRows emitted ✓ (Fixes 3+4)
- U16 removed — cleaner interaction model ✓ (Fix 4)
- U23 now shows labels panel → Journey E satisfied ✓ (Fix 7)

Re-tracing Mermaid consistency after fixes:
- S1–S5 -.-> N16 added ✓ (Fix 8)
- U13 → N12 → S8 in both tables and diagram ✓ (Fixes 5+6)
- N6 wiring shows N1 sequential, then N2–N5 concurrent ✓ (Fix 1)

---

## Quality Gate

- [x] All user stories trace through wiring coherently (post-fix)
- [x] No incoherent wiring (Fix 1 resolves N1/N2 concurrency)
- [x] No missing paths (Fix 7 resolves Journey E labels)
- [x] No phantom affordances (Fixes 2+4 remove U15, U16)
- [x] All affordances pass naming test (Fix 6 renames N12)
- [x] All Wires Out targets exist in tables (Fix 5 corrects U13)
- [x] Tables and diagram consistent (Fix 8 adds missing Mermaid wires)
- [x] Bypassed handler resolved (Fix 3 routes U14 through N15)

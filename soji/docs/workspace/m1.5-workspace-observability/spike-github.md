---
pipeline: m1.5-workspace-observability
stage: spike
spike: github
status: resolved
---

# Spike: GitHub Source (A5)

## Flagged Unknown

Part A5 (`GitHubSource`) is flagged because three things need confirmation before the mechanism is fully understood:

1. **`gh milestone list` JSON schema** — what are the exact field names for open/closed issue counts? `openIssues`? `open`? Need to confirm with a live call.
2. **Auth detection** — is `gh auth status` exit code reliable as an auth gate? Any edge cases (multiple accounts, enterprise)?
3. **Eager vs. lazy issue loading per milestone** — loading issues for every milestone at snapshot time could be slow if there are many milestones. Need to measure and decide.

---

## Investigation

### 1. `gh milestone list` JSON schema

```bash
gh milestone list --repo cmbays/soji --state open --json title,number,openIssues,closedIssues 2>&1
```

**Questions:**
- Are `openIssues` and `closedIssues` valid JSON fields?
- What does the field look like if a milestone has 0 issues?
- Is `number` the right field name for the milestone ID?

### 2. `gh issue list` JSON schema

```bash
gh issue list --repo cmbays/soji --state open --json number,title,labels 2>&1
```

**Questions:**
- Is `labels` an array of `{ name: String, ... }` objects or just strings?
- Does `--milestone` flag accept milestone title or number?

### 3. Auth detection

```bash
gh auth status; echo "exit: $?"
```

**Questions:**
- Exit code 0 = authed, non-zero = not authed?
- Does this work when multiple GitHub accounts are configured?
- Is there a `--json` flag for machine-readable output?

### 4. Latency measurement

```bash
time gh milestone list --repo cmbays/soji --state open --json title,number,openIssues,closedIssues
time gh issue list --repo cmbays/soji --milestone "M1.5" --state open --json number,title,labels
```

**Decision threshold:**
- If milestone list + all issue fetches complete in < 2s total → eager loading at snapshot time is fine.
- If > 2s → lazy load issues per milestone (only load when milestone row is expanded in TUI).

---

## Findings

### 1. `gh milestone list` — does not exist
`gh` has no `milestone` subcommand. Use `gh api` instead:
```bash
gh api repos/{owner}/{repo}/milestones
```
JSON fields (snake_case): `title`, `number`, `open_issues`, `closed_issues`. Confirmed.

### 2. `gh issue list` labels schema
Labels are objects: `{ id, name, description, color }`. Extract `name` field only.
`--milestone "title"` filter works with the full milestone title string. Confirmed.

### 3. Auth detection
`gh auth status` exits 0 when authenticated, non-zero when not. Reliable. Confirmed.

### 4. Latency
- `gh api repos/cmbays/soji/milestones` → ~290ms
- `gh issue list --state open --json number,title,labels` → ~400ms
- Total: ~700ms — well under 2s threshold

**Decision: eager loading at snapshot time.** Both calls together complete in < 1s. No lazy-load needed.

---

## Resolution

A5 mechanism updated:
- Use `gh api repos/{owner}/{repo}/milestones` (not `gh milestone list`)
- JSON fields: `open_issues`, `closed_issues` (snake_case)
- Labels: extract `.name` from label objects
- Auth gate: `gh auth status` exit code
- Loading: eager at snapshot time

⚠️ removed from A5. R4 passes. Fit Check updated to ✅.

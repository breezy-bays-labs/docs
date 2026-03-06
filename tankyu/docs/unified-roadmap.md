# Unified Roadmap — Research Intelligence Pipeline

## Milestone: Research Intelligence Pipeline

### Phase 0: Product Documentation

| Deliverable | Status |
|-------------|--------|
| Product design doc | ✅ |
| User journeys | ✅ |
| Design rationale (ADRs) | ✅ |
| Interaction design | ✅ |
| This roadmap | ✅ |

### Phase 1: Infrastructure (Epics 1 + 2)

**Goal:** Single URL ingest + insight node type

| Deliverable | Epic | Description |
|-------------|------|-------------|
| `manual` source type | 1 | New source type for conversational ingest |
| `detectEntryType()` | 1 | URL pattern → entry type mapping |
| `SourceManager.ingest()` | 1 | Create source + entry + edges atomically |
| `tankyu ingest <url>` | 1 | CLI command for URL ingestion |
| `InsightSchema` | 2 | Zod schema for insight nodes |
| `InsightStore` | 2 | File-per-node persistence in `~/.tankyu/insights/` |
| `IInsightStore` | 2 | Port interface for insight operations |
| `derived-from`, `synthesizes` edges | 2 | New edge types for citation chains |

### Phase 2: Research Workflow (Epic 3)

**Goal:** Dive sessions produce insight nodes with citation chains

| Deliverable | Epic | Description |
|-------------|------|-------------|
| `InsightWriter` | 3 | Creates insights + edges atomically |
| `writeResearchNote()` | 3 | Research note with derived-from edges |
| `writeSynthesis()` | 3 | Synthesis insight combining other insights |
| `ingestCitation()` | 3 | Citation URL → entry with provenance |
| `promoteSource()` | 3 | Entry → recurring source promotion |
| `/tankyu:dive` update | 3 | Skill produces insights, ingests citations |

### Phase 3: Smart Classification (Epic 4)

**Goal:** LLM fallback for unclassified entries

| Deliverable | Epic | Description |
|-------------|------|-------------|
| `LlmClassifier` | 4 | Claude Haiku classification with mock-friendly interface |
| `config.llmClassify` | 4 | Feature flag (default: off) |
| `EntryClassifier` Layer 3 | 4 | LLM fallback after source-rule + keyword |

### Phase 4: Digest + Export (Epic 6)

**Goal:** Research digests from insight store, Renshin integration

| Deliverable | Epic | Description |
|-------------|------|-------------|
| `/tankyu:digest` update | 6 | Reads from InsightStore, produces briefing insights |
| `/tankyu:send` update | 6 | Serializes insights with graph context |

### Future: Epics 5, 7-9

- **X Bookmarks Scanner** (Epic 5) — X API v2, bookmark polling
- **Autonomous Research** — Cron-triggered scan → dive → synthesize loops
- **Cross-Repo Integration** — Kata agents querying the knowledge graph
- **Work Ticket Enrichment** — GitHub issues tagged with research context

## Dependency Graph

```
Phase 0 (Docs)
  ↓
Epic 1 (Ingest)    ──┐
                      ├── Epic 3 (Dive + Citations) ── Epic 6 (Digest/Export)
Epic 2 (Insights)  ──┘
Epic 4 (LLM Classify) ── independent, parallel with 1-2
```

## Verification Protocol

After each phase:
1. `npm test` — all tests pass (baseline: 208)
2. `npm run typecheck` — clean
3. `npm run build` — clean

### End-to-end after Phase 1

```bash
tankyu ingest https://x.com/user/status/12345
tankyu status --json
# Verify: source (manual) + entry (tweet) + produced edge
```

### End-to-end after Phase 2

```bash
/tankyu:dive <topic>
# Verify: insight created, derived-from edges, citation entries
ls ~/.tankyu/insights/
```

## GitHub Milestone: Research Intelligence Pipeline

| Issue | Epic | Labels |
|-------|------|--------|
| Product documentation suite | A | `docs` |
| Single URL ingest (`manual` source + `tankyu ingest`) | 1 | `feature`, `cli` |
| Insight node type, schema, and store | 2 | `feature`, `domain` |
| InsightWriter + derived-from edges | 3 | `feature`, `research` |
| Update /tankyu:dive for citation chain + source discovery | 3 | `feature`, `skill` |
| LLM classification layer (Claude Haiku) | 4 | `feature`, `classify` |
| X bookmarks scanner | 5 | `feature`, `future` |
| Update /tankyu:digest + /tankyu:send for insight store | 6 | `feature`, `skill` |

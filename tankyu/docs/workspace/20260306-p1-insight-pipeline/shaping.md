---
shaping: true
pipeline_id: 20260306-p1-insight-pipeline
stage: shaping
selected_shape: A
---

# Shaping: P1 Insight Creation Pipeline

## Requirements

| ID | Requirement | Status |
|----|-------------|--------|
| R0 | Research reports in `research/*.md` can be indexed as insight nodes via a CLI command | Core goal |
| R1 | Reports stay as markdown files — the insight node is a graph pointer (index layer), not a copy of the content | Must-have |
| R2 | A YAML front matter schema defines the machine-readable contract between report authors (human or agent) and the indexer | Must-have |
| R3 | Indexing is idempotent — re-running on an already-indexed report updates the insight node in place, does not create a duplicate | Must-have |
| R4 | The command accepts either a single file path or a directory path | Must-have |
| R5 | Batch (directory) mode collects per-file errors and reports a summary; does not abort the batch on a single failure | Must-have |
| R6 | The UUID assigned on first indexing is written back into the report's YAML front matter, making the file the stable ID anchor | Must-have |
| R7 | Indexed insight nodes are linked to their topic via a graph edge (`tagged-with`) | Must-have |
| R8 | Related insights can be cross-linked from front matter (`relates_to: [uuid, ...]`), creating `relates-to` edges | Nice-to-have |
| R9 | A mechanism exists to add valid front matter to existing reports that have none (retroactive setup) | Must-have |
| R10 | The `/tankyu:dive` skill produces reports with YAML front matter — agents self-index their output | Must-have |
| R11 | `from-report` supports `--dry-run` — shows what would be created/updated without writing anything | Must-have |
| R12 | Insight nodes do not store a `topicId` field — topic association is graph-only via `tagged-with` edges; a report can belong to multiple topics | Must-have |

---

## Updated Fit Check (R0–R12, Shape A only — B and C already rejected)

| Req | Requirement | Status | A |
|-----|-------------|--------|---|
| R0 | Research reports can be indexed as insight nodes via a CLI command | Core goal | ✅ |
| R1 | Reports stay as markdown files — insight node is an index layer | Must-have | ✅ |
| R2 | YAML front matter schema defines the machine-readable contract | Must-have | ✅ |
| R3 | Indexing is idempotent — re-run updates in place, no duplicates | Must-have | ✅ |
| R4 | Command accepts single file or directory path | Must-have | ✅ |
| R5 | Batch mode collects per-file errors, does not abort on first failure | Must-have | ✅ |
| R6 | UUID written back into report's YAML front matter on first indexing | Must-have | ✅ |
| R7 | Indexed insights linked to topics via `tagged-with` graph edges | Must-have | ✅ |
| R8 | `relates_to` cross-links create `relates-to` edges between insights | Nice-to-have | ✅ |
| R9 | Mechanism exists to add front matter to existing reports | Must-have | ✅ |
| R10 | `/tankyu:dive` produces reports with valid YAML front matter | Must-have | ✅ |
| R11 | `--dry-run` flag on `from-report` — shows changes without writing | Must-have | ✅ |
| R12 | No `topicId` on insight nodes — graph-only topic association, multi-topic supported | Must-have | ✅ |

---

## Shape A: Two-Phase (Retroactive Script + Strict Indexer) — SELECTED

**Description:** Cleanest separation of concerns. A retroactive script (`scripts/add-front-matter.ts`) handles setup — extracts inline metadata and writes front matter. `tankyu insight from-report <path>` is strict: requires valid front matter, fails immediately if missing or invalid. The command only indexes; the script only generates front matter. The `/tankyu:dive` skill is updated to write report files with front matter natively.

### Parts

| Part | Mechanism | Flag |
|------|-----------|:----:|
| **A1** | `ReportFrontMatterSchema` (Zod v4) in `src/domain/types/report.ts` — fields: `id?: uuid`, `type: research-note\|synthesis\|briefing`, `title: string`, `date: YYYY-MM-DD`, `topics: string[]` (min 1), `subject: string`, `subject_url?: string`, `tags: string[]`, `key_points: string[]`, `relates_to: uuid[]`, `status: draft\|complete` | |
| **A2** | `report-parser.ts` — `parseReport(filePath)` slices YAML block between `---` delimiters, parses via `js-yaml`, validates with `ReportFrontMatterSchema`; `hasFrontMatter(filePath)` boolean check; `writeIdBack(filePath, id)` in-place string insertion after opening `---` | |
| **A3** | `InsightFromReport` class in `insight-from-report.ts` — `ingest(filePath)`: parse → resolve topic by name → upsert insight in `InsightStore` → create graph edges → write UUID back if new; `ingestDirectory(dir)`: glob `*.md`, process each, collect `IngestReportResult[]` and `BatchError[]` | |
| **A4** | `tankyu insight from-report <path> [--dry-run]` CLI subcommand — detects file vs directory via `fs.stat`; `--dry-run` prints what would happen with no writes; registers `insight` parent command with `from-report` subcommand in `program.ts` | |
| **A5** | `scripts/add-front-matter.ts` — `tsx`-runnable; args: `<path\|dir> --topic <name> [--type <type>]`; extraction: title from first `# ` heading, date from `**Date**:` line, subject from filename slug (strips `-research-report`), subject_url from `**Repository**:` line; key_points: bullets from "Executive Summary" or "Recommendations" section (limit 7), fall back to H2 section titles minus boilerplate (`Executive Summary`, `Linked Sources`, `Open Questions`, `References`); skips files that already have `---` front matter | |
| **A6** | `body` field computation at ingest time (not in front matter) — `key_points.map(p => '• ' + p).join('\n') \|\| 'Research report on ' + subject + '.'` — satisfies `InsightSchema.body: z.string().min(1)` | |
| **A7** | `docs/report-front-matter-spec.md` — authoritative front matter contract: field descriptions, types, examples, how the retroactive script populates each field, what the dive skill must produce | |
| **A8** | `skills/dive/SKILL.md` update — add "Writing a research report" step: after synthesis, write `research/<topic>-<date>-<slug>-dive-notes.md` with YAML front matter auto-populated from dive context (topic names, key_points from synthesis, date = today); front matter includes `status: complete`; after writing, run `tankyu insight from-report <file>` to index; skill lives at `/Users/cmbays/Github/tankyu/skills/dive/SKILL.md` | |
| **A9** | Schema cleanup for multi-topic — remove `topicId` from `InsightSchema` (Zod v4); remove `topicId` from `ResearchNoteInput` and `SynthesisInput` in `InsightWriter` (callers pass `topicIds: string[]` instead, writer creates one `tagged-with` edge per topic); remove or reimagine `IInsightStore.listByTopic` (callers query graph then fetch by ID — method removed from port interface); front matter field becomes `topics: string[]` min length 1 | |

---

## Other Shapes Considered

### Shape B: One-Phase Smart Command

`from-report` handles both setup and indexing — auto-generates front matter from inline metadata if missing.

**Rejected:** `--topic` becomes conditionally required based on hidden state (whether front matter exists). Callers cannot rely on a stable interface. Mixes setup and indexing concerns.

### Shape C: Three Subcommands

`tankyu insight scaffold` / `from-report` / `batch` as three separate CLI commands.

**Rejected:** Fails R4 (no single command handles both file and dir). Splits batch into a 3rd command for no real benefit over detecting dir vs file in `from-report`.

---

## Fit Check

| Req | Requirement | Status | A | B | C |
|-----|-------------|--------|---|---|---|
| R0 | Research reports can be indexed as insight nodes via a CLI command | Core goal | ✅ | ✅ | ✅ |
| R1 | Reports stay as markdown files — insight node is an index layer, not a content copy | Must-have | ✅ | ✅ | ✅ |
| R2 | YAML front matter schema defines the machine-readable contract | Must-have | ✅ | ✅ | ✅ |
| R3 | Indexing is idempotent — re-run updates in place, no duplicates | Must-have | ✅ | ✅ | ✅ |
| R4 | Command accepts single file or directory path | Must-have | ✅ | ✅ | ❌ |
| R5 | Batch mode collects per-file errors, does not abort on first failure | Must-have | ✅ | ✅ | ✅ |
| R6 | UUID written back into report's YAML front matter on first indexing | Must-have | ✅ | ✅ | ✅ |
| R7 | Indexed insights linked to topic via `tagged-with` graph edge | Must-have | ✅ | ✅ | ✅ |
| R8 | `relates_to` cross-links create `relates-to` edges between insights | Nice-to-have | ✅ | ✅ | ✅ |
| R9 | Mechanism exists to add front matter to existing reports | Must-have | ✅ | ✅ | ✅ |
| R10 | `/tankyu:dive` produces reports with valid YAML front matter | Must-have | ✅ | ❌ | ❌ |
| R11 | `--dry-run` flag on `from-report` — shows changes without writing | Must-have | ✅ | ❌ | ✅ |

**Notes:**
- C fails R4: directory handling is a separate `batch` command.
- B fails R10: B's smart command swallows the front matter problem without a stable authoring contract for the skill.
- B fails R11: B's combined parse+index path makes dry-run harder to define (does it dry-run the front matter generation too?).

---

## Decision Log

| # | Decision | Reasoning |
|---|----------|-----------|
| 1 | Shape A over B | B's `--topic` becomes conditionally required based on hidden state. Breaks composability for callers (agents, scripts). |
| 2 | Shape A over C | C adds `batch` as a separate command. A detects file vs dir in `from-report`. Same result, less surface area. |
| 3 | `scripts/` not `tankyu insight scaffold` | One-time retroactive tool. Permanent CLI command is premature. `tsx`-runnable script is sufficient. |
| 4 | R10 → Must-have | User confirmed skill update is in scope for P1. |
| 5 | `--dry-run` → Must-have | User confirmed. Required safety check before retroactive batch run. |
| 6 | Key_points extraction from sections | Try bullets in "Executive Summary" or "Recommendations" first; fall back to H2 section titles (minus boilerplate). Limit 7. Better than empty, doesn't require LLM. |
| 7 | No auto-topic creation | `from-report` fails with "create it with: tankyu topic create X" if topic not found. Explicit is better for graph integrity. |
| 8 | `body` computed at ingest time, not stored in front matter | `InsightSchema.body` is required but shouldn't be a separate authoring field. Derived from `key_points` at index time. |
| 9 | `js-yaml` over `gray-matter` | Already installed. Custom front matter slicing (between `---` delimiters) is trivial. Avoids extra dependency. |
| 10 | Skill location: `/Users/cmbays/Github/tankyu/skills/dive/SKILL.md` | Found in main repo. Currently stores notes in entry metadata — no report file written. A8 adds the report-writing step. |

---

## Topic Mapping: 26 Existing Reports

With R12 (multi-topic support), the either/or concern is gone — reports can belong to multiple topics. Three topics to create; cross-tagging where appropriate.

### Topics to create

```bash
tankyu topic create doodlestein-ecosystem --description "Doodlestein / Jeffrey Emanuel repo research"
tankyu topic create tankyu-v2 --description "Tankyu v2 architecture, graph backend, and schema decisions"
tankyu topic create agentic-research-workflow --description "Research workflow patterns, flywheel design, agent coordination"
```

### Assignment table

| Report | Primary topic | Also tagged |
|--------|--------------|-------------|
| asupersync | doodlestein-ecosystem | |
| beads-rust | doodlestein-ecosystem | |
| beads-viewer-rust | doodlestein-ecosystem | |
| cass-memory-system | doodlestein-ecosystem | tankyu-v2 |
| cass-search | doodlestein-ecosystem | tankyu-v2 |
| destructive-command-guard | doodlestein-ecosystem | agentic-research-workflow |
| franken-engine | doodlestein-ecosystem | |
| franken-node | doodlestein-ecosystem | |
| franken-whisper | doodlestein-ecosystem | |
| frankensearch | doodlestein-ecosystem | tankyu-v2 |
| frankensqlite | doodlestein-ecosystem | tankyu-v2 |
| frankenterm | doodlestein-ecosystem | agentic-research-workflow |
| frankentui | doodlestein-ecosystem | |
| mcp-agent-mail | doodlestein-ecosystem | agentic-research-workflow |
| meta-skill | doodlestein-ecosystem | agentic-research-workflow |
| pi-agent-rust | doodlestein-ecosystem | tankyu-v2 |
| sqlmodel-rust | doodlestein-ecosystem | tankyu-v2 |
| toon-rust | doodlestein-ecosystem | |
| ultimate-bug-scanner | doodlestein-ecosystem | |
| useful-tmux-commands | doodlestein-ecosystem | agentic-research-workflow |
| agentic-coding-flywheel-setup | agentic-research-workflow | |
| flywheel-connectors | agentic-research-workflow | tankyu-v2 |
| embedded-graph-db-landscape | tankyu-v2 | |
| nanograph | tankyu-v2 | |
| sqlite-graph-backend | tankyu-v2 | |
| tankyu-graph-scaling-analysis | tankyu-v2 | |

**Cross-tagging rationale:**
- `cass-memory-system` / `cass-search` → also `tankyu-v2` (CASS patterns directly inform our memory architecture)
- `frankensearch` / `frankensqlite` → also `tankyu-v2` (graph search and SQLite patterns relevant to our backend decision)
- `pi-agent-rust` / `sqlmodel-rust` → also `tankyu-v2` (Rust patterns for Phase 3 port)
- `destructive-command-guard` / `frankenterm` / `mcp-agent-mail` / `meta-skill` → also `agentic-research-workflow` (agent coordination / safety / tooling patterns)
- `flywheel-connectors` → also `tankyu-v2` (directly about tankyu ingestion pipeline)

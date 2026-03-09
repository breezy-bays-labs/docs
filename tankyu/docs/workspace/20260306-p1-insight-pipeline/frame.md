---
shaping: true
pipeline_id: 20260306-p1-insight-pipeline
stage: frame
---

# Frame: P1 Insight Creation Pipeline

## Source Material

### Codebase artifacts

- **26 research reports** in `research/*.md` — all end with: *"This report is pending conversion to an Insight node once the research graph document-linking feature is implemented."*
- **`InsightSchema`** (Zod v4): `{ id: uuid, type: research-note|synthesis|briefing, title, body, keyPoints: string[], citations: uuid[], topicId: uuid, createdAt, updatedAt, metadata? }`
- **`InsightStore`**: `create, get, list, listByTopic, update` — file-per-node JSON in `~/.tankyu/insights/`
- **`InsightWriter`** (existing): `writeResearchNote`, `writeSynthesis` — creates insight nodes programmatically from structured input
- **`EdgeTypeSchema`**: includes `tagged-with`, `relates-to`, `derived-from`, `synthesizes`, `informs` — no `has-insight`
- **`IGraphStore`**: `addEdge, removeEdge, getEdgesByNode, getNeighbors, query` — flat `edges.json`
- **`TopicManager.get(name)`**: resolves topic by name, throws `NotFoundError` if missing

### Architecture context (v2-architecture-vision.md)

- File + Graph Pointer Pattern: *"Research reports = markdown files (the content). Insight nodes = graph pointers with metadata, edges, search index. The graph is an index layer, not the primary content store."*
- P1 deliverable: *"Update /tankyu:dive to produce proper insight nodes. Retroactively process 26 existing reports."*
- North star: *"every research session, agent spike, and project exploration compounds into a queryable knowledge graph."*

### Memory

- P1 priority order: Insight Creation Pipeline is NEXT (decided 2026-03-06)
- Approach: *"thorough planning — interview, research, user stories, then build in one pass"*
- 26 research reports currently in `research/`, zero insight nodes indexed from them

### Current inline metadata format (sample from reports)

```
# Research Report: Nanograph -- On-Device Graph Database for AI Agent Memory
**Date**: 2026-03-05
**Author**: Andrew Altshuler (@aaltshuler / @1eo)
**Repository**: https://github.com/aaltshuler/nanograph
**Language**: Rust
**Stars**: 67 | **Forks**: 3
**License**: MIT
**Status**: Pre-1.0, actively developed
```

Not all reports follow this exact pattern. Non-repo reports may have `**Context**:` instead of `**Repository**:`, or omit some fields.

---

## Problem Statement

26 research reports represent months of synthesis work — per-repo deep dives into the Doodlestein ecosystem, graph backend evaluation, architecture research. None of it is queryable. The knowledge graph cannot:
- Traverse from a topic to its research insights
- Answer "what do we know about Nanograph?"
- Show provenance: which entries informed which insights
- Surface related research across repos and topics

The reports are dark matter: present but undetectable to the system. The flywheel cannot spin until insights are indexed. Every day they remain unindexed, new research sessions lack context that already exists.

Additionally, the `/tankyu:dive` skill (which agents use to produce research reports) has no contract for what it should output — no front matter spec, no machine-readable structure. Each dive produces a slightly different format.

---

## Outcome Definition

After P1 is complete:

1. Every research report in `research/` has YAML front matter with a stable UUID (`id`) written back to the file on first indexing
2. Running `tankyu insight from-report research/` indexes all reports as insight nodes, linked to their topics via graph edges
3. The front matter spec is documented and is the authoritative contract between report authors (human or agent) and the indexer
4. The `/tankyu:dive` skill produces reports with front matter that passes validation — agents can self-index their output
5. Re-running `from-report` on already-indexed reports is a safe no-op (updates in place)
6. Batch mode reports summary counts: indexed / updated / skipped / errored

**What is explicitly NOT in scope for P1:**
- `tankyu insight list` / `tankyu insight inspect` query commands (P3)
- Semantic search over insights (Phase 2 — Nanograph)
- LLM-generated key_points or body text (future `/tankyu:dive` evolution)
- Executive summary extraction from markdown body (too complex, low ROI)
- Automatic topic creation from front matter (explicit topic management is P1)

**Confirmed in scope after shaping (2026-03-06):**
- R10 (skill update) is Must-have — `/tankyu:dive` at `skills/dive/SKILL.md` must produce reports with YAML front matter
- `--dry-run` flag on `from-report` is Must-have — required safety check before retroactive batch
- key_points extraction from section headers is the retroactive strategy (bullets in Executive Summary/Recommendations, fall back to H2 titles)

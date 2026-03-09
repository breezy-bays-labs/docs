# Research Report Front Matter Spec

**Version:** 1.0
**Status:** Authoritative — this document is the contract between report authors and the indexer.

---

## Overview

Research reports in `research/*.md` are indexed as insight nodes via `tankyu insight from-report`. The machine-readable contract between report authors (human or agent) and the indexer is a YAML front matter block at the top of each report file.

The insight node is a **graph pointer** — it holds metadata and edges. The markdown file remains the primary content store.

---

## Schema

```yaml
---
id: 550e8400-e29b-41d4-a716-446655440000   # UUID — written back by indexer on first run (leave blank)
type: research-note                          # research-note | synthesis | briefing
title: "Nanograph — On-Device Graph Database"
date: 2026-03-05                             # YYYY-MM-DD
topics:
  - tankyu-v2                               # At least one topic name (must exist in graph)
  - agentic-research-workflow               # Multi-topic supported
subject: nanograph                          # Short slug — the thing being researched
subject_url: https://github.com/aaltshuler/nanograph  # Optional: primary URL
tags:
  - graph
  - rust
  - memory
key_points:
  - "Purpose-built for agent memory and knowledge graphs"
  - "BM25 + semantic + hybrid RRF search built-in"
  - "Schema-as-code (.pg files) — git-friendly"
relates_to:
  - 660e8400-e29b-41d4-a716-446655440000   # UUIDs of related insight nodes (optional)
status: complete                            # draft | complete
---
```

---

## Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | UUID string | No (auto-assigned) | Stable identifier. Left blank on first authoring — indexer writes it back on first `from-report` run. **Do not change after first indexing.** |
| `type` | enum | Yes | `research-note` (default), `synthesis`, or `briefing` |
| `title` | string | Yes | Human-readable title. Min 1 character. |
| `date` | YYYY-MM-DD | Yes | Date the report was authored or last substantially revised |
| `topics` | string[] | Yes (min 1) | Topic names that this report belongs to. Topics must exist (`tankyu topic create <name>`). Multi-topic supported — a report can belong to multiple topics. |
| `subject` | string | Yes | Short slug for the thing being researched (repo name, concept, project) |
| `subject_url` | URL string | No | Primary URL for the subject (repo, docs site, etc.) |
| `tags` | string[] | No | Free-form tags for search and filtering (may be empty) |
| `key_points` | string[] | No | 3–7 key findings. Used to construct the insight node `body`. May be empty if the report is a stub. |
| `relates_to` | UUID[] | No | UUIDs of related insight nodes. Creates `relates-to` graph edges. |
| `status` | enum | Yes | `draft` or `complete` |

---

## Topic Association

Topics are graph-only — there is no `topicId` field on the insight node. Instead, `tankyu insight from-report` creates `tagged-with` edges between the insight node and each topic in the `topics` array.

A report can belong to multiple topics:

```yaml
topics:
  - doodlestein-ecosystem
  - tankyu-v2
```

This creates two `tagged-with` edges: `insight → doodlestein-ecosystem` and `insight → tankyu-v2`.

Topics must exist before indexing. If a topic is missing:

```
Error: Topic not found: tankyu-v2
Create it with: tankyu topic create tankyu-v2
```

---

## Body Computation

The insight node `body` field is required but is not stored in front matter — it is computed at index time:

```
body = key_points.map(p => '• ' + p).join('\n')
     | 'Research report on ' + subject + '.'   # fallback if key_points empty
```

This means `key_points` is the primary authoring lever for insight body content.

---

## Idempotency

Re-running `tankyu insight from-report <file>` on an already-indexed report is safe:

- If `id` is present in front matter **and** the insight node exists in the store → **update** (title, body, keyPoints, metadata refreshed; edges re-synced)
- If `id` is absent → **create** new insight node and write UUID back to front matter
- If `id` is present but node does not exist (e.g., store was wiped) → **create** new node with the existing ID

---

## Dry Run

Before indexing, verify what would happen without writing:

```bash
tankyu insight from-report research/ --dry-run
```

Output shows `created`/`updated` per file. No writes occur.

---

## Retroactive Setup

For existing reports without front matter, use the retroactive script:

```bash
# Single file
tsx scripts/add-front-matter.ts research/nanograph-research-report.md --topic tankyu-v2

# Directory (all .md files)
tsx scripts/add-front-matter.ts research/ --topic doodlestein-ecosystem

# Multi-topic
tsx scripts/add-front-matter.ts research/cass-memory-system-research-report.md \
  --topic doodlestein-ecosystem \
  --topic tankyu-v2

# Dry run
tsx scripts/add-front-matter.ts research/ --topic doodlestein-ecosystem --dry-run
```

The script:
1. Extracts title from the first `# ` heading
2. Extracts date from `**Date**:` inline metadata
3. Derives subject from the filename (strips `-research-report` suffix)
4. Extracts subject_url from `**Repository**:` line
5. Extracts key_points from bullets in "Executive Summary" or "Recommendations" section (limit 7); falls back to H2 section titles minus boilerplate (`Executive Summary`, `Linked Sources`, `Open Questions`, `References`, `Recommendations`)
6. Skips files that already have front matter

After running the script, review the generated front matter, then index:

```bash
tankyu insight from-report research/ --dry-run   # verify first
tankyu insight from-report research/             # index
```

---

## Dive Skill Contract

The `/tankyu:dive` skill produces research reports with valid front matter natively. After synthesis, it writes `research/<topic>-<date>-<slug>-dive-notes.md` with front matter auto-populated from dive context:

- `type: research-note`
- `title`: derived from dive subject
- `date`: today's date (YYYY-MM-DD)
- `topics`: topic names from dive context
- `subject`: subject slug
- `subject_url`: repository or primary URL if available
- `key_points`: 3–7 points from synthesis
- `status: complete`

After writing the file, the skill runs:

```bash
tankyu insight from-report research/<file>
```

This self-indexes the output immediately.

---

## Graph Edges Created

| Edge type | From | To | When |
|-----------|------|----|------|
| `tagged-with` | insight | topic | One per entry in `topics` array |
| `relates-to` | insight | insight | One per UUID in `relates_to` array |

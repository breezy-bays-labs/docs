# v1 Product Design

## Problem Statement

Valuable information dies in browser tabs, bookmark folders, and social feeds. Manual research is episodic — you find something interesting, maybe bookmark it, and forget. No system captures, contextualizes, and compounds knowledge over time.

The cost isn't just lost links. It's lost *connections*. A GitHub repo you bookmarked last month relates to a blog post you read yesterday, but nothing ties them together. Research stays flat when it should be a graph.

## What Tankyu Is

Tankyu (探求) is research intelligence infrastructure for the Mentalify ecosystem. It captures raw information from the internet, contextualizes it through AI-powered research, and builds a knowledge graph that serves as a shared resource across all projects and agents.

Tankyu is not a bookmarking tool, a read-later app, or a note-taking system. It is a **knowledge compiler** — it takes scattered inputs and produces structured, queryable intelligence.

## North Star

A knowledge graph that gets smarter with every interaction. Every raw entry enriches the graph. Every research dive discovers new connections. Every insight compounds on what's already there. Autonomous agents and interactive sessions across the Mentalify ecosystem query Tankyu's graph for context.

## Mentalify Ecosystem Positioning

| Tool | Japanese | Role |
|------|----------|------|
| **Kata** | 形 | Development methodology engine. Encodes *how work gets done*. |
| **Tankyu** | 探求 | Research intelligence. Captures and contextualizes knowledge. |
| **Renshin** | 練心 | Mind vault. Brainstorming, reflection, knowledge synthesis. |

**Integration flows:**

- Tankyu feeds research into Renshin's inbox via `/tankyu:send`
- Kata agents query Tankyu's knowledge graph for project context
- Renshin's reflections can spawn new Tankyu research topics

## Design Principles

1. **Typed edges carry reasons** — every graph link has a type and rationale, not just a connection
2. **Progressive disclosure** — scan cheaply, triage with judgment, dive deep selectively
3. **Forgetting** — sources decay, entries archive, confidence needs reinforcement
4. **Value in topology** — insights emerge from connections, not from individual nodes
5. **Scripts for process, LLMs for judgment** — deterministic operations are code; inference is Claude
6. **Raw vs. inferred** — entries are observed facts; insights are reasoned analysis. Both are first-class graph nodes.
7. **Recursive enrichment** — research discovers sources, sources produce entries, entries trigger research
8. **Research as infrastructure** — the knowledge graph serves agents, interactive sessions, and automated jobs across all repos

## v1 Scope

### In scope (Epics 1–6)

| Epic | Name | Summary |
|------|------|---------|
| 1 | Ingest | Conversational source capture via `/tankyu:scan`, URL auto-detection, source-rule + keyword classification |
| 2 | Insights | Insight nodes with citation chains, `cites` edges linking insights back to entries |
| 3 | Dive + Citations | Deep research workflow — `/tankyu:dive` discovers new URLs, creates research notes, spawns new sources |
| 4 | LLM Classification | LLM fallback for entries that escape rule-based and keyword classification |
| 5 | X Bookmarks (future) | Bulk import of X bookmarks as sources |
| 6 | Digest + Export | Research digests, Renshin inbox integration via `/tankyu:send` |

### Out of scope

- **Autonomous research agent** — scheduled, unattended research loops
- **Cross-repo integration** — Kata/Renshin querying Tankyu's graph programmatically
- **Work ticket enrichment** — auto-enriching GitHub issues with research context
- **Temporal intelligence** — trend detection, change-over-time analysis
- **Source health scoring** — automated quality/reliability ratings

## Design Constraints

- **No database** — all persistence is JSON files in `~/.tankyu/`, one file per node
- **ESM-only** — TypeScript with `"type": "module"`, Zod v4 for schema validation
- **CLI + Skills** — core operations are deterministic CLI commands; LLM judgment lives in Claude Code skills
- **Offline-first** — the knowledge graph is local; no cloud dependencies for core operations
- **Schema-driven** — every node and edge validates against Zod schemas before persistence

## Success Criteria

1. A user can share a URL conversationally and have it ingested, classified, and linked to a topic — without manual type selection
2. Research sessions produce insight nodes with full citation chains back to source entries
3. The knowledge graph is queryable — given a topic, surface all related entries, insights, and discovered sources
4. Classification works in three layers: source-rule (instant) → keyword (fast) → LLM (accurate fallback)
5. Research compounds — a dive session that discovers new sources creates a feedback loop of future scans
6. Digests can be exported to Renshin's inbox for synthesis and reflection

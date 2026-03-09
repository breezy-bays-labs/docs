# v2 Architecture Vision — Nanograph-Backed Research Flywheel

**Date**: 2026-03-06
**Status**: Approved direction, pre-implementation
**Decision**: Nanograph as graph backend, Rust port after product maturity

---

## Executive Summary

Tankyu evolves from a JSON-file CLI tool into a Nanograph-backed research intelligence flywheel serving all projects in the Breezy Bays Labs ecosystem. The transition happens in three phases: (1) prove the product in TypeScript, (2) adopt Nanograph as the graph backend, (3) port to Rust. Research content, schemas, and queries carry forward through every phase.

The north star: **every research session, agent spike, and project exploration compounds into a queryable knowledge graph that makes every future decision better-informed.**

---

## Three-Phase Roadmap

### Phase 1: Prove the Product (TypeScript) — Months 1-2

**Goal**: Nail the data model, ingestion patterns, and research workflow. Build the flywheel in TypeScript where iteration is fast.

**Deliverables**:
- Schema design for all node types and edge types
- Ingestion pipeline: URL → source + entry + classification + edges
- Research report generation (manual + semi-automated)
- Synthesis pipeline: research notes → cross-cutting insights
- Query patterns: "what do we know about X?", provenance chains, topic exploration
- Research queue workflow (org-level issue queue → graph ingestion)
- Quick wins on current JSON stores: cache edges, validate on write only, parallelize list()

**Key question to answer**: What are the inputs, how do we get data in quickly and continuously, and how do we mine the graph effectively?

**Constraints**:
- No database migration yet — optimize JSON stores with quick wins
- Focus on usability: how does a researcher (human or agent) interact with the graph?
- Build viewer/query tools for exploring the graph
- Validate schema design against real research data (22+ Doodlestein reports, nanograph research, etc.)

### Phase 2: Adopt Nanograph (TypeScript + Nanograph) — Month 3

**Goal**: Replace JSON stores with Nanograph. Unlock graph-native queries, search, and CDC audit trail.

**Deliverables**:
- `.pg` schema file modeling all tankyu node/edge types
- `.gq` query files for all key operations
- `NanographStore` implementing `IGraphStore` port interface
- Migration script: JSON files → Nanograph `.nano` directory
- Graph-constrained search: "search insights within this topic"
- CDC ledger: every mutation tracked for audit and time-travel
- Dual persistence: Nanograph for queries, JSON/JSONL export for git history

**What this unlocks**:
- nanoQL graph traversal (vs manual array filtering)
- Built-in BM25 + semantic + hybrid RRF search
- Schema enforcement at query time (vs runtime Zod validation)
- Time-travel queries ("what did the graph look like last week?")
- Graph-constrained search ("search within this subgraph")

### Phase 3: Port to Rust — Months 4-6

**Goal**: Rewrite tankyu in Rust with Nanograph as a native crate dependency. Adopt battle-tested Rust patterns from the Doodlestein ecosystem research.

**What carries over unchanged**:
- `.pg` schema files (language-agnostic)
- `.gq` query files (language-agnostic)
- `.nano` data directory (same binary format)
- CDC ledger JSONL (same format)
- All research content (markdown reports, insight data)
- Architecture: domain → infrastructure → features → CLI

**What changes**:
- Zod schemas → Rust structs with serde
- Commander CLI → clap with derive
- `IGraphStore` interface → Rust trait
- `nanograph-db` NAPI bindings → `nanograph` Rust crate (direct)
- Node.js async → tokio async runtime

**Rust patterns to adopt from day 1** (from Doodlestein research):
- `forbid(unsafe_code)` double-layer: source attribute + Cargo.toml `[lints.rust]`
- Clippy pedantic + nursery lint groups as warnings
- `thiserror` domain error enums + `anyhow` at CLI boundary + contextual error hints
- Release profile: `opt-level = 3`, full LTO, `codegen-units = 1`, `panic = "abort"`, `strip = true`
- `insta` snapshot testing for CLI output and query results
- `clap` with derive for CLI structure

---

## Data Model — Graph Schema

### Node Types

```
node Topic {
    slug: String @key
    name: String
    description: String?
    tags: [String]?
    routing_keywords: [String]?
    routing_min_score: F64?
    status: enum(active, paused, archived)
    created_at: DateTime
    updated_at: DateTime
}

node Source {
    slug: String @key
    name: String
    url: String @unique
    source_type: enum(github_repo, github_user, x_account, x_post, web_page, rss_feed, manual, agent_report)
    status: enum(active, stale, dormant, pruned)
    check_count: U32
    hit_count: U32
    miss_count: U32
    discovered_via: String?
    discovery_reason: String?
    created_at: DateTime
    updated_at: DateTime
}

node Entry {
    slug: String @key
    title: String
    body: String
    body_embedding: Vector(1536) @embed(body) @index
    entry_type: enum(commit, pr, release, tweet, article, bookmark, spike_report, agent_finding, manual)
    content_hash: String?
    state: enum(unread, read, archived)
    signal_level: enum(noise, normal, interesting, critical)?
    source_id: String
    created_at: DateTime
    updated_at: DateTime
}

node Insight {
    slug: String @key
    title: String
    body: String
    body_embedding: Vector(1536) @embed(body) @index
    insight_type: enum(research_note, synthesis, briefing)
    key_points: [String]
    topic_id: String
    created_at: DateTime
    updated_at: DateTime
    metadata_json: String?
}

node Entity {
    slug: String @key
    name: String
    entity_type: enum(person, org, technology, concept, project)
    description: String?
    created_at: DateTime
}

node Project {
    slug: String @key
    name: String
    repo_url: String?
    description: String?
    status: enum(active, paused, archived)
}
```

### Edge Types

```
edge Monitors: Topic -> Source { reason: String? }
edge Produced: Source -> Entry { reason: String? }
edge TaggedWith: Entry -> Topic { score: F64?, method: enum(source_rule, keyword, llm)? }
edge DerivedFrom: Insight -> Entry { reason: String? }
edge DerivedFromSource: Insight -> Source { reason: String? }
edge Synthesizes: Insight -> Insight { reason: String? }
edge DiscoveredVia: Source -> Source { reason: String? }
edge Mentions: Entry -> Entity { reason: String? }
edge RelatesTo: Entity -> Entity { reason: String? }
edge Cites: Entry -> Entry { reason: String? }
edge Supersedes: Entry -> Entry { reason: String? }
edge InformsProject: Insight -> Project { reason: String?, relevance: F64? }
edge InspiresTicket: Insight -> Entry { ticket_url: String?, reason: String? }
```

### New Source Type: Agent Reports

Agent spike reports from other projects are a first-class input:

```
source_type: agent_report
entry_type: spike_report
```

When an agent in soji, kata, or any other project does research, it writes findings as a spike report entry. The entry's `derived-from` edges trace back to whatever sources the agent consulted. This creates cross-project knowledge flow:

```
soji agent spike on "TUI rendering patterns"
  → Entry (spike_report) in tankyu
    → DerivedFrom → Source (frankentui repo)
    → DerivedFrom → Source (ratatui docs)
    → TaggedWith → Topic ("Rust TUI Development")
    → InformsProject → Project (soji)
```

---

## Query Design — Why It Ports Over

nanoQL queries are stored in `.gq` files — plain text, language-agnostic, version-controlled. The queries themselves don't depend on TypeScript or Rust. They depend on the `.pg` schema.

### Key Queries for Tankyu

```
// What do we know about a topic?
query topic_knowledge($topic: String) {
    match {
        $t: Topic { slug: $topic }
        $t monitors $s
        $s produced $e
    }
    return { $s.name, $e.title, $e.entry_type, $e.created_at }
    order { $e.created_at desc }
}

// Full provenance chain for an insight
query insight_provenance($insight: String) {
    match {
        $i: Insight { slug: $insight }
        $i derivedFrom{1,3} $n
    }
    return { $n.slug, $n.name }
}

// Search insights with graph-constrained scope
query search_topic_insights($topic: String, $q: String) {
    match {
        $t: Topic { slug: $topic }
        $i: Insight { topic_id: $topic }
    }
    return { $i.title, $i.slug, bm25($i.body, $q) as score }
    order { bm25($i.body, $q) desc }
    limit 10
}

// Hybrid search across all insights
query search_insights($q: String) {
    match {
        $i: Insight {}
    }
    return { $i.title, $i.slug, rrf(nearest($i.body_embedding, $q), bm25($i.body, $q)) as score }
    order { rrf(nearest($i.body_embedding, $q), bm25($i.body, $q)) desc }
    limit 20
}

// Cross-project insight relevance
query project_insights($project: String) {
    match {
        $p: Project { slug: $project }
        $i informsProject $p
    }
    return { $i.title, $i.insight_type, $i.key_points }
    order { $i.created_at desc }
}

// Entity network — who/what connects to what?
query entity_network($entity: String) {
    match {
        $e: Entity { slug: $entity }
        $e relatesTo{1,2} $other
    }
    return { $other.name, $other.entity_type }
}

// Recent unprocessed entries (research queue)
query unprocessed_entries() {
    match {
        $e: Entry { state: "unread" }
    }
    return { $e.title, $e.entry_type, $e.source_id, $e.created_at }
    order { $e.created_at asc }
}

// What changed this week?
query recent_changes($since: DateTime) {
    match {
        $i: Insight {}
        not { $i.created_at < $since }
    }
    return { $i.title, $i.insight_type, $i.created_at }
    order { $i.created_at desc }
}
```

These queries are the same whether tankyu is TypeScript or Rust. The calling code changes; the queries don't.

In TypeScript:
```typescript
const db = await Database.open("tankyu.nano");
const results = await db.run(querySource, "topic_knowledge", { topic: "ai-research" });
```

In Rust:
```rust
let db = Database::open("tankyu.nano")?;
let results = db.run(&query_source, "topic_knowledge", &[("topic", "ai-research")])?;
```

The `.gq` file is identical in both cases.

---

## Ingestion Patterns — Getting Data In

### Input Types

| Input | Source Type | Entry Type | How It Gets In |
|-------|-----------|------------|---------------|
| URL shared conversationally | auto-detected | auto-detected | `tankyu ingest <url>` |
| GitHub repo for research | github_repo | commit, pr, release | `tankyu source add <url>` + `tankyu scan` |
| X account to follow | x_account | tweet | `tankyu source add <url>` + scan |
| X post/thread | x_post | tweet | `tankyu ingest <url>` (uses /read-x skill) |
| Agent spike report | agent_report | spike_report | Agent writes directly via CLI or SDK |
| Research queue item | manual | varies | Consumed from org research queue (GitHub issue) |
| X bookmark | x_post | bookmark | Future: automated X API polling |
| RSS feed | rss_feed | article | Future: RSS polling |

### The Research Queue Pattern

```
GitHub Issue (breezy-bays-labs/ops) labeled "research"
  → Human or agent creates issue with URL + context
  → Tankyu consumes: closes issue, creates source + entry + edges
  → If research needed: triggers dive → creates insight
  → If synthesis needed: links to existing insights → creates synthesis
```

This decouples "I found something interesting" from "it's been processed into the graph." The queue is the buffer.

### Automated Ingestion Flywheel

```
Level 1 (manual):     Human shares URL → tankyu ingest
Level 2 (triggered):  Agent completes spike → writes spike_report entry
Level 3 (scheduled):  Cron scans active sources → new entries
Level 4 (reactive):   New entry triggers classification → may trigger dive
Level 5 (autonomous): Unread entries auto-prioritized → agent dives on high-signal
```

Each level builds on the previous. Phase 1 gets us to Level 2. Phase 2 unlocks Levels 3-4. Phase 3 enables Level 5.

---

## Report Generation and Synthesis

### Research Report Structure (current, proven)

Every research report follows a consistent structure:

```markdown
# Research Report: {Name} — {Subtitle}

**Date / Author / Repo / Stars / License / Status**

## Executive Summary
## [Technical Sections — varies by topic]
## Relevance to Our Stack
### Patterns Worth Adopting (table)
## Open Questions & Risks
## Recommendations
## Linked Sources
```

This structure maps directly to an Insight node:
- Executive Summary → `body`
- Patterns Worth Adopting → `key_points` array
- Linked Sources → `citations` (source node UUIDs)

### Synthesis Pipeline

```
Individual research notes (1 per source/topic)
  ↑ derived-from edges
Source nodes

Cross-cutting synthesis (themes across multiple reports)
  ↑ synthesizes edges
Individual research notes

Briefings (actionable summaries for specific projects)
  ↑ synthesizes edges
Cross-cutting syntheses
  ↑ informs-project edges
Project nodes
```

### Automating Report Generation

**Phase 1 (now)**: Claude Code skill `/tankyu:dive` does the research, writes the report, creates the insight node and edges.

**Phase 2**: Deterministic pipeline wraps the skill:
1. Pull unprocessed entries from queue
2. Group by source/topic
3. For each group: run dive → generate report → create insight → link edges
4. After all reports: identify cross-cutting themes → generate synthesis
5. For each project: check for relevant new insights → update project briefing

**Phase 3**: The pipeline runs as a scheduled task. Human reviews synthesis outputs.

---

## Doodlestein Ecosystem Patterns — Integration Map

### Patterns Already Inside Nanograph

These don't need separate adoption — Nanograph implements them natively:

| Pattern | Doodlestein Source | Nanograph Implementation |
|---------|-------------------|--------------------------|
| BM25 lexical search | CASS Search, FrankenSearch, Meta Skill | `bm25()` predicate in nanoQL |
| Semantic vector search | FrankenSearch (MiniLM) | `nearest()` with `@embed` annotation |
| Hybrid RRF fusion | FrankenSearch, Meta Skill | `rrf()` combining nearest + bm25 |
| Dual persistence | Beads Rust (SQLite + JSONL) | Lance columnar + CDC JSONL ledger |
| Schema-as-code | Meta Skill (SKILL.md) | `.pg` schema files |
| Time-travel/audit | Beads Rust (JSONL history) | CDC ledger with version-gated visibility |

### Patterns to Layer On Top

These compose with Nanograph without conflict:

| Pattern | Source | How to Apply |
|---------|--------|-------------|
| Three-layer cognitive model | CASS Memory | Node types: EpisodicMemory, WorkingMemory, ProceduralMemory. Promotion pipeline as graph edges. |
| Confidence decay (90-day half-life) | CASS Memory | `confidence: F64` property on insights. Decay computed at query time. |
| SKILL.md structured artifacts | Meta Skill | Skill nodes with searchable body content via `@embed` |
| Bandit-optimized suggestions | Meta Skill | Thompson sampling state as node properties (`alpha`, `beta`) |
| Agent identity tracking | MCP Agent Mail | Agent nodes with `@key slug`. `authored` edges trace provenance. |
| Shared-identifier convention | MCP Agent Mail | Issue ID = thread ID = node `@key` = commit prefix |
| TOON output format | UBS, TOON Rust | Wrap `nanograph run --format json` output in TOON conversion |
| DCG command safety | DCG | Guard agent writes to graph via DCG subprocess |
| `--agent` output mode | Pi Agent Rust, FrankenTerm | `tankyu query --agent` auto-selects optimal output format |
| `doctor` health check | ACFS, UBS | `tankyu doctor` validates graph integrity, missing edges, orphan nodes |

### Patterns for the Rust Port Specifically

| Pattern | Source | Application |
|---------|--------|-------------|
| `forbid(unsafe_code)` double-layer | Pi Agent Rust | Source attribute + Cargo.toml `[lints.rust]` |
| Clippy pedantic + nursery | Pi Agent Rust | Enforce code quality from day 1 |
| `thiserror` + `anyhow` | Pi Agent Rust | Domain errors + CLI boundary |
| Release profile optimization | Pi Agent Rust | LTO, codegen-units=1, panic=abort, strip |
| `insta` snapshot testing | Pi Agent Rust | Test CLI output and query results deterministically |
| RAII terminal cleanup | FrankenTUI | If/when we add TUI views for graph exploration |
| Inline TUI mode | FrankenTUI | Graph exploration without full-screen takeover |
| Event-driven automation | FrankenTerm | Graph events → trigger downstream actions |

---

## Viewer and Exploration

### Phase 1: CLI Query Tool

```bash
# Search the graph
tankyu query "what patterns are relevant to TUI development?"

# Explore a node's neighborhood
tankyu explore <node-id> --depth 3

# Show provenance chain
tankyu provenance <insight-id>

# Project briefing
tankyu brief <project-slug>

# Research queue status
tankyu queue
```

### Phase 2: Graph Viewer

Options to evaluate:
- **Nanograph's own viewer** (if one develops)
- **Beads Viewer pattern** (TUI with graph analysis — PageRank, critical path)
- **Web viewer** (export graph to D3/Cytoscape visualization)
- **Obsidian integration** (export graph as Obsidian vault with wikilinks)

The viewer decision depends on the Rust port — a ratatui-based graph explorer would be natural in Rust tankyu.

---

## Extensibility — Topics and Projects

### Topic Management

Topics are lightweight categories, not rigid hierarchies:
- Topics can overlap (an entry can belong to multiple topics via `tagged-with` edges)
- Topics have routing rules (keywords, URL patterns, min score)
- Topics decay: if no new entries match for 90 days, topic transitions to `paused`
- New topics emerge from research: a dive session may reveal a new area worth tracking

### Project Integration

Projects are first-class nodes linked to insights via `informs-project` edges:
- Every project in our ecosystem (soji, kata, sanso, tankyu, renshin) is a Project node
- Insights automatically considered for project relevance during synthesis
- `tankyu brief soji` returns all insights relevant to soji, ranked by recency and relevance
- Agent spike reports from any project flow back through the graph

### Cross-Project Knowledge Flow

```
Project A agent does research spike
  → spike_report Entry in tankyu
    → classified to Topics
    → derived-from Sources
    → informs-project Project A

Later: Project B agent queries tankyu
  → "what do we know about graph databases?"
  → discovers Project A's spike report
  → new insight synthesizes findings for Project B's context
    → informs-project Project B
```

This is the flywheel. Every project's research enriches every other project.

---

## Migration Path

### JSON → Nanograph Migration

```bash
# 1. Export current state
tankyu export --format jsonl > tankyu-backup.jsonl

# 2. Initialize nanograph
nanograph init tankyu.nano --schema schema.pg

# 3. Load data
nanograph load tankyu.nano --file tankyu-backup.jsonl --mode overwrite

# 4. Verify
nanograph run --db tankyu.nano --query verify.gq --name count_all --format json

# 5. Switch backend (behind IGraphStore interface)
# Update config to point to NanographStore instead of JsonStore
```

### Nanograph TypeScript → Rust Migration

```bash
# Nothing to migrate — same .nano directory, same .pg/.gq files
# Just change the import:
# TypeScript: import { Database } from "nanograph-db"
# Rust:       use nanograph::Database
```

---

## Success Metrics

### Phase 1 (TypeScript)
- 50+ research reports in the graph
- 5+ projects connected via `informs-project` edges
- Agent spike reports flowing in from at least 2 projects
- Research queue → graph ingestion under 5 minutes per item
- "What do we know about X?" answerable in one command

### Phase 2 (Nanograph)
- Sub-100ms graph queries (vs current full-file-load)
- Semantic search returning relevant insights
- CDC ledger capturing all mutations
- Graph-constrained search working ("search within topic X")

### Phase 3 (Rust)
- Single binary, no Node.js dependency
- Native nanograph crate integration (no NAPI)
- All .pg/.gq files unchanged from Phase 2
- Performance: sub-10ms for all standard queries

---

## Open Questions

1. **Embedding model**: Nanograph defaults to OpenAI text-embedding-3-small. Do we want local embeddings (Ollama) for offline-first?
2. **Graph algorithms**: Nanograph doesn't have built-in PageRank/centrality. Do we need this? Compute externally?
3. **Multi-user**: Currently single-user. If we want team research, Nanograph's concurrency model needs evaluation.
4. **Backup strategy**: CDC ledger + `nanograph export` covers it, but what's the git-push cadence?
5. **Research queue UX**: GitHub issues vs dedicated queue vs Renshin inbox — where does the research queue live?

---

## Related Research

All research reports supporting this vision are in `research/`:

### Doodlestein Ecosystem (22 reports)
- `asupersync-research-report.md` — Cancel-correct async runtime
- `beads-rust-research-report.md` — Dual persistence, agent work queues
- `beads-viewer-rust-research-report.md` — Graph-aware TUI with PageRank
- `cass-memory-system-research-report.md` — Three-layer cognitive model
- `cass-search-research-report.md` — Session search, hybrid search patterns
- `destructive-command-guard-research-report.md` — Agent safety, command blocking
- `flywheel-connectors-research-report.md` — Zone trust, service integration
- `franken-engine-research-report.md` — Bayesian guardplane
- `franken-node-research-report.md` — Trust-native runtime
- `franken-whisper-research-report.md` — ASR orchestration, Bayesian routing
- `frankensearch-research-report.md` — Two-tier hybrid search library
- `frankensqlite-research-report.md` — Concurrent-writer SQLite
- `frankentui-research-report.md` — Composable TUI crates, RAII cleanup
- `frankenterm-research-report.md` — Terminal hypervisor, Robot Mode API
- `mcp-agent-mail-research-report.md` — Agent coordination, file leases
- `meta-skill-research-report.md` — Skill graph, bandit suggestions
- `pi-agent-rust-research-report.md` — Zero-unsafe Rust CLI patterns
- `sqlmodel-rust-research-report.md` — Derive macro ORM
- `toon-rust-research-report.md` — Token-optimized output format
- `ultimate-bug-scanner-research-report.md` — Multi-language static analysis
- `useful-tmux-commands-research-report.md` — Agent orchestration patterns
- `agentic-coding-flywheel-setup-research-report.md` — Environment bootstrapping

### Graph Backend Decision (4 reports + synthesis)
- `nanograph-research-report.md` — On-device graph DB deep-dive
- `sqlite-graph-backend-research-report.md` — Recursive CTEs, FTS5
- `embedded-graph-db-landscape-research-report.md` — SurrealDB, Cozo, Kuzu, etc.
- `tankyu-graph-scaling-analysis.md` — Growth projections, breaking points

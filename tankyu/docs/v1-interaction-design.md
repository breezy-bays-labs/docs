# v1 Interaction Design

## Architecture: Two Layers

### Layer 1: Core CLI (TypeScript)

Deterministic operations — CRUD, scanning, graph management. All commands support `--json` for machine-readable output.

| Command | Description |
|---------|-------------|
| `tankyu topic create <name>` | Create a research topic |
| `tankyu topic list` | List all topics |
| `tankyu topic inspect <name>` | Show topic details |
| `tankyu source add <topic> <url>` | Add source (auto-detects type) |
| `tankyu source list <topic>` | List sources for a topic |
| `tankyu source remove <id>` | Remove a source (marks pruned) |
| `tankyu scan <topic>` | Scan all active sources |
| `tankyu ingest <url>` | Ingest a URL (new in v1) |
| `tankyu status` | Dashboard with counts |

### Layer 2: Plugin Skills (Claude Code)

Skills wrap the CLI and add LLM judgment:

| Skill | Trigger | Description |
|-------|---------|-------------|
| `/tankyu:scan` | Manual | Runs CLI scan, triages results by signal level |
| `/tankyu:dive` | Manual | Deep research on high-signal entries, produces insight nodes |
| `/tankyu:digest` | Manual | Narrative synthesis of recent insights |
| `/tankyu:send` | Manual | Export insights to Renshin inbox |

## Data Flow

```
Capture               Contextualize              Compound              Export
─────────           ─────────────────         ──────────────        ──────────
URL → ingest        classify (3 layers)       dive → insight        digest
scan → entries      source-rule → keyword     derived-from edges    send → Renshin
                    → LLM fallback            citation chains
                                              synthesize insights
```

## Skill Workflow: /tankyu:dive

1. Read recent high-signal entries for the topic
2. For each entry, fetch and analyze content
3. Discover referenced URLs → ingest as citation entries
4. Identify quality recurring sources → recommend promotion
5. Write research note as insight node with `derived-from` edges
6. Transition processed entries to `state: 'read'`

## Skill Workflow: /tankyu:digest

1. Read recent insights for the topic (from InsightStore)
2. Synthesize into narrative briefing
3. Surface cross-topic themes where relevant
4. Store as insight (type: `briefing`)

## Skill Workflow: /tankyu:send

1. Gather insights and graph context for the topic
2. Serialize to markdown with citations and source links
3. Deliver to Renshin inbox format

## Source Lifecycle

```
active → stale → dormant → pruned
  │        │        │
  │        │        └─ No checks for dormantDays (default: 30)
  │        └─ No new content for staleDays (default: 7)
  └─ Actively producing content
```

Sources discovered through research start as `active` with `discoveredVia` provenance.
Manual sources start as `active` with `checkCount: 1` (already checked).

## Entry State Machine

```
new → scanned → triaged → read → archived
```

- `new`: Just ingested, not yet analyzed
- `scanned`: Content fetched and indexed
- `triaged`: Signal level assigned (high/medium/low/noise)
- `read`: Deep-dived via /tankyu:dive
- `archived`: No longer actively relevant

# v1 User Journeys

## US-1: Share What I Find (Epic 1)

> As a researcher, I share individual URLs (X posts, articles, repos) conversationally and they become tracked entries in my knowledge graph.

### Acceptance Criteria

- `tankyu ingest <url>` creates a `manual` source + entry
- Entry type auto-detected from URL pattern (tweet, commit, article, etc.)
- If `--topic` specified, entry tagged to that topic immediately
- Otherwise, classifier routes entry to matching topics
- Duplicate URL reuses existing source

## US-2: Go Deeper (Epic 3)

> As a researcher, I dive into entries to understand them deeply — Claude reads the content, researches connections, and produces a research note with citations.

### Acceptance Criteria

- `/tankyu:dive <topic>` reads high-signal entries
- Claude discovers referenced URLs → ingests as citation entries
- Claude identifies quality recurring sources → recommends adding
- Research note stored as `insight` node (type: `research-note`)
- Insight linked to all source entries via `derived-from` edges
- Triggering entry transitions to `state: 'read'`

## US-3: Build Lasting Knowledge (Epic 2)

> As a researcher, my research notes are first-class nodes in the knowledge graph — queryable, linkable, and traceable back to their raw data citations.

### Acceptance Criteria

- `insight` node type with schema: title, body (markdown), keyPoints, citations, topicId
- `derived-from` edges link insights → entries
- `synthesizes` edges link insights → other insights
- `~/.tankyu/insights/{id}.json` file-per-node storage
- Full citation graph traversable from any insight

## US-4: Discover What to Follow (Epic 3)

> As a researcher, the system discovers new sources through research — blogs mentioned in tweets, GitHub users whose repos keep appearing, X accounts that post about my topics.

### Acceptance Criteria

- Dive skill identifies potential recurring sources during research
- New sources created with `discoveredVia` and `discoveryReason: 'research-discovery'`
- `discovered-via` edges trace source provenance
- User can confirm or dismiss recommended sources

## US-5: Smart Classification (Epic 4)

> As a researcher, entries that don't match explicit rules or keywords still get classified to the right topics using LLM judgment.

### Acceptance Criteria

- LLM classification layer (Claude Haiku) as fallback after source-rule and keyword
- Gated behind config flag `llmClassify` (default off)
- Batch entries for cost efficiency
- Classification method recorded as `'llm'` on edges

## US-6: Get Briefed (Epic 6)

> As a researcher, I get narrative research digests that synthesize insights (not just raw entries) and export findings to my mind vault.

### Acceptance Criteria

- `/tankyu:digest` reads from insight store (not entry metadata)
- Digest output stored as insight (type: `briefing`)
- `/tankyu:send` serializes insights with graph context to Renshin inbox
- Cross-topic themes surfaced when relevant

## US-7: Scan My Bookmarks (Future — Epic 5)

> As a researcher, my X bookmarks are a source — new bookmarks automatically become entries.

### Acceptance Criteria (future)

- `x-bookmarks` source type with X API v2 integration
- Automatic bookmark polling on configurable interval
- New bookmarks ingested as entries with proper type detection

## US-8: Autonomous Research (Future)

> As a researcher, I set up autonomous research agents that scan, dive, and synthesize on a schedule — I review the output asynchronously.

### Acceptance Criteria (future)

- Cron-triggered research loop: scan → triage → dive → synthesize
- Configurable per-topic schedules
- Async review queue for human-in-the-loop approval

## US-9: Research-Enriched Work (Future)

> As a developer, my GitHub issues automatically get tagged with relevant research insights — Tankyu's graph enriches my work tickets before I start on them.

### Acceptance Criteria (future)

- Cron job scans new insights and open GitHub issues
- Matches based on topic overlap, keyword relevance, entity mentions
- Adds comment with research context + links to source entries/insights
- Agents in Kata query Tankyu's graph during research/plan gyo

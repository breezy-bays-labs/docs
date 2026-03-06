# v1 Design Rationale

Architecture Decision Records for the Research Intelligence Pipeline.

## ADR-1: Two-Tier Data Model (Entries = Raw, Insights = Inferred)

**Status:** Accepted

**Context:** The system ingests raw data (tweets, commits, articles) and produces reasoned analysis. These have fundamentally different trust levels and lifecycles.

**Decision:** Entries represent observed facts from the internet. Insights represent reasoned analysis derived from entries. Both are first-class graph nodes but with distinct schemas and stores.

**Consequences:**
- Clear provenance: you can always trace an insight back to the raw data that informed it
- Different validation rules: entries need URLs and source links; insights need citations and key points
- Separate stores enable independent lifecycle management (entries can archive while insights persist)

## ADR-2: Insight as Its Own Node Type (Not Entry Subtype)

**Status:** Accepted

**Context:** We could model insights as a special entry type or as a separate node type entirely.

**Decision:** Insights are a separate node type with their own schema (`InsightSchema`), store (`InsightStore`), and directory (`~/.tankyu/insights/`).

**Consequences:**
- Stronger type safety — insight-specific fields (body, keyPoints, citations) are schema-enforced
- Cleaner queries — `listByTopic` on InsightStore doesn't need to filter out raw entries
- Additional store to maintain, but follows the same JsonStore pattern

## ADR-3: Citation Chain Pattern

**Status:** Accepted

**Context:** Research naturally creates a web of references. A dive session reads entries, discovers new URLs, and produces analysis.

**Decision:** The recursive flywheel: Entry → dive → discover URLs → ingest as citation entries → write research note → insight node + derived-from edges → new sources produce future entries.

**Consequences:**
- Knowledge compounds automatically through research
- Citation provenance is fully traceable via `derived-from` and `discovered-via` edges
- Risk of unbounded recursion — managed by skill-layer depth limits

## ADR-4: Manual Source Type for Conversational Ingest

**Status:** Accepted

**Context:** Users share individual URLs in conversation. These are one-shot captures, not recurring sources to poll.

**Decision:** `manual` source type with `checkCount: 1, hitCount: 1`. Created automatically by `ingest()`. URL deduplication reuses existing sources.

**Consequences:**
- Seamless conversational capture — no explicit "create source then add entry" workflow
- Manual sources won't appear in scan loops (they're already checked)
- Source list may grow large — mitigated by lifecycle management (active → stale → dormant → pruned)

## ADR-5: Denormalized Citations Array on InsightSchema

**Status:** Accepted

**Context:** Citation relationships could live only in the graph (as edges) or be denormalized onto the insight node.

**Decision:** Both. `InsightSchema.citations: uuid[]` provides fast access. `derived-from` edges provide full graph traversal.

**Consequences:**
- Fast citation lookup without graph queries
- Must keep both in sync — InsightWriter handles this atomically
- Slight data duplication, but the source of truth is the graph; citations array is a convenience index

## ADR-6: Classification Pipeline Layering

**Status:** Accepted

**Context:** Entries need to be routed to topics. Different methods have different cost/accuracy/speed profiles.

**Decision:** Three-layer pipeline: source-rule (instant, high confidence) → keyword (fast, medium confidence) → LLM (slow, high accuracy). Each layer only runs if the previous layers produced no match.

**Consequences:**
- Most entries classified instantly via source-rule (they came from a monitored source)
- Keyword layer catches cross-topic entries with matching tags
- LLM layer is the expensive fallback, gated behind `config.llmClassify`
- Classification method recorded on edges for auditability

## ADR-7: Research as Shared Infrastructure

**Status:** Accepted

**Context:** The knowledge graph has value beyond a single user session. Agents in other repos could benefit from research context.

**Decision:** The graph is designed to be queryable by external agents. JSON files on disk, typed schemas, and edge-based traversal make it accessible without a database.

**Consequences:**
- Kata agents can read `~/.tankyu/` to enrich work tickets with research context
- No authentication/authorization needed (local filesystem)
- Future milestone: API layer for cross-process access

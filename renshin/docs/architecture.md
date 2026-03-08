---
title: Architecture
description: How Renshin's vault architecture works.
---

# Architecture

## Three-Space Separation

The vault enforces strict separation between three types of content:

### self/ — Identity Space

Persistent agent/user identity. Loaded at session start to establish context before action.

| File | Purpose | Update frequency |
|------|---------|------------------|
| `identity.md` | Who I am — values, approach, expertise | Rarely |
| `methodology.md` | How I work — quality standards, thinking patterns | Evolves with learnings |
| `goals.md` | Current threads — active, deferred, completed | Every session |

### notes/ — Knowledge Space

The knowledge graph. Permanent, compounds over time. This is where the value lives.

**Structure**:
- Flat folder (no subfolders for organization)
- Prose-sentence titles (each note makes one claim)
- Wiki-links (`[[note title]]`) create graph edges
- MOC hierarchy for navigation: Hub → Domain → Topic → Notes
- YAML frontmatter for structured metadata

**Progressive disclosure**:
1. Title gives the claim
2. Description (~150 chars) adds context beyond the title
3. Full note content for the complete argument

### ops/ — Operational Space

Temporal state. Flows through and gets processed. Never treated as permanent knowledge.

- `sessions/` — Point-in-time session logs
- `observations/` — Friction signals, operational learnings (pre-promotion)
- `queue/` — Processing queue state
- `health/` — Schema validation snapshots
- `methodology/` — Vault self-knowledge (why configured this way)

**Promotion rule**: Content moves from ops/ → notes/ (one-directional).

## Knowledge Graph Structure

### Wiki-Links as Edges

Every `[[wiki link]]` is a curated connection. Unlike embedding-based links, wiki-links carry reasons — you chose to connect these ideas because of a specific relationship.

Prefer inline links with context over footer links:
- Good: "Since [[spaced repetition compounds over weeks]], we should..."
- Avoid: "See also: [[spaced repetition compounds over weeks]]"

### MOC Hierarchy

Maps of Content manage attention:

```
index.md (Hub)
├── domain-moc.md
│   ├── topic-moc.md
│   │   └── atomic notes
│   └── topic-moc.md
└── domain-moc.md
```

Each MOC contains:
- Brief orientation (what this area covers)
- Core ideas with context for each link
- Tensions (unresolved conflicts)
- Open questions (unexplored directions)

## Integration Points

### Inbox ← Tankyu

Tankyu research outputs land in `inbox/` as markdown files. Process with `/reduce` → `/reflect` → `/reweave`.

### notes/ → Projects

Project agents read relevant MOCs and notes from `~/Github/renshin/notes/`. Progressive disclosure: load MOC first, drill into specifics.

### ops/observations/ → self/methodology.md

Friction signals and process learnings captured during sessions can be promoted to methodology updates.

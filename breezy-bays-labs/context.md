# Context Management — San-Sō (三層)

> How we organize information for AI-assisted development. Information has durability. Structure that ignores durability forces agents to re-read stale context on every turn, and buries durable knowledge under temporal noise. San-Sō prevents this structurally.

---

## The Principle

Not all information has the same durability. A core behavioral rule is permanent. A project's accumulated learnings grow steadily. The current task state changes every session. Treating these the same — injecting everything into every context window — means paying to re-read history when you just need to work.

**San-Sō (三層)** — three strata — organizes information by durability so each tier is loaded at the right time, in the right way.

---

## Three Strata

### Self (shin 心) — Permanent

> "Who am I? How do I work?"

Always loaded. Changes only through deliberate methodology evolution, not during normal sessions.

**Content**: Identity, behavioral rules, non-negotiable constraints, build commands, workflow rules, tool preferences.

**Test**: Would a completely fresh session break without this? If yes — it belongs here.

**In Claude Code**: `CLAUDE.md` (global and project-level).

**Warning signs**: The file is over 150 lines. It contains summaries of documents that exist elsewhere. It describes things that rarely change and never affect behavior mid-session.

---

### Knowledge (chi 智) — Stable

> "What have we learned? Where do I look?"

Loaded on demand — read explicitly when the current task touches that domain. Never auto-injected.

**Content**: CLI references, architectural decisions, design principles, roadmaps, tool overviews, accumulated learnings, org philosophy (like this document).

**Test**: Is this needed right now, or only when working on a specific thing? "Only when..." → knowledge tier.

**In Claude Code**: Topic files in the memory directory, project docs, org docs site. `MEMORY.md` holds a reference map — a pointer table to knowledge files — rather than inlining their content.

**Warning signs**: You summarized a document that already exists rather than pointing to it. The same information appears in both an always-injected file and a topic file.

---

### Ops (do 動) — Temporal

> "What's happening right now?"

Auto-injected but kept ruthlessly thin. Valid for one session to one cycle. Decays aggressively.

**Content**: Current task state, active issues, recent decisions from the last session, working shortcuts, what was done last time.

**Test**: Will this still matter three sessions from now? If no — it's ops. If it proves durable, promote it to a knowledge file.

**In Claude Code**: `MEMORY.md` — the only always-injected file that changes frequently.

**Target size**: Under 80 lines. Every line should answer "what's happening right now." Anything else has a better home.

---

## The Reference Map Pattern

`MEMORY.md` (ops) tells you *where* to find things rather than *what* they say:

```markdown
## Reference Map

| Topic | Location |
|-------|----------|
| CLI lexicon | `memory/lexicon.md` |
| Architecture vision | `docs/vision/three-space-architecture.md` |
| Org context management | breezy-bays-labs.mintlify.app → Philosophy → Context |
```

Read the file when you need it. You don't pay to re-read it on turns where it's irrelevant.

---

## Applying to Any Project

When setting up a new project for AI-assisted development:

**`CLAUDE.md` (self)** — one screenful maximum:
- What the project is (one paragraph)
- Build/test commands
- Key architectural rules and non-negotiables
- A pointer to org knowledge: `Org knowledge index: ~/.claude/knowledge/index.md`

**`MEMORY.md` (ops)** — current state only:
- What's in progress
- Immediate open issues
- Recent decisions from last session
- A reference map pointing to topic files

**Topic files (knowledge)** — on demand:
- CLI or domain vocabulary
- Architecture deep-dives
- Design decisions
- Anything stable that would bloat MEMORY.md

**What the codebase provides for free** — don't duplicate in context:
- Service APIs → read source files
- Schema definitions → read the type files
- Test patterns → read adjacent test files

---

## Promotion

When an ops entry in `MEMORY.md` proves durable — it keeps being relevant across multiple sessions — promote it:

- Promote to a **topic file** if it's project-specific reference (CLI lexicon, workflow patterns)
- Promote to **project docs** if it's worth human eyes (architecture decisions, design choices)
- Promote to **org docs** if it applies across projects (like this document)

Remove the inline entry once promoted. The pointer in the reference map replaces it.

This mirrors the knowledge promotion concept in data storage (San-Ma): temporal operational data flows through and gets distilled into durable knowledge at natural intervals.

---

## Sizing Targets

| File | Stratum | Target |
|------|---------|--------|
| Global `CLAUDE.md` | self | < 120 lines |
| Project `CLAUDE.md` | self | < 80 lines |
| `MEMORY.md` | ops | < 80 lines |
| Topic files | knowledge | No constraint — read on demand |

---

## Relationship to San-Ma (三間)

San-Sō governs context *injection* — what gets loaded into an agent turn.

San-Ma (三間) governs data *storage* — how a project organizes its persistent data. In Kata, this means structuring `.kata/` into `self/` (identity and methodology), `knowledge/` (accumulated learnings), and `ops/` (active execution data).

The two principles mirror each other because they solve the same underlying problem in different domains: information with different durabilities degrades when stored or loaded the same way.

**See also**: Kata's `docs/vision/three-space-architecture.md` — the detailed application of both San-Ma and San-Sō to the Kata methodology engine.

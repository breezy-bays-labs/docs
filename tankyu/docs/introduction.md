# Tankyu (探求) — Research Intelligence

Tankyu is a tool for **continuous research intelligence** — tracking sources across the internet, scanning for new content, and building a knowledge graph of discoveries.

## What it does

- **Track sources**: Add GitHub repos, X accounts, blogs, and web pages to research topics
- **Scan for updates**: Fetch new commits, PRs, releases, articles, and posts
- **Build connections**: Every relationship in the system is a typed edge with a reason
- **Progressive disclosure**: Scan cheaply, triage with judgment, dive deep selectively
- **Forget gracefully**: Sources decay from active → stale → dormant → pruned

## Architecture

Tankyu has two layers:

### Layer 1: Core CLI (TypeScript)

Deterministic operations — CRUD, scanning, graph management. All commands support `--json` for machine-readable output.

```
tankyu topic create "AI Research" --tags "ai,ml"
tankyu source add "AI Research" https://github.com/anthropics/claude-code
tankyu scan "AI Research" --json
tankyu status
```

### Layer 2: Plugin Skills (Claude Code)

Skills wrap the CLI and add LLM judgment:

- `/tankyu:scan` — runs CLI scan, then triages results by signal level

## Data Model

All data lives in `~/.tankyu/` as JSON files:

- **Topics** — research areas you're tracking
- **Sources** — URLs that produce content (repos, accounts, blogs)
- **Entries** — individual pieces of content discovered by scanning
- **Edges** — typed relationships with reasons (monitors, produced, relates-to, etc.)

## Design Principles

1. **Typed edges carry reasons** — every graph link has a type and rationale
2. **Progressive disclosure** — scan → triage → deep dive
3. **Forgetting** — sources decay, entries archive, confidence needs reinforcement
4. **Value in topology** — insights emerge from connections
5. **Scripts for process, LLMs for judgment** — deterministic operations are code; inference is Claude

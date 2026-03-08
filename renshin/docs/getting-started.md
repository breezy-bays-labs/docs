---
title: Getting Started
description: How to set up Renshin with arscontexta.
---

# Getting Started

Renshin is powered by [arscontexta](https://github.com/agenticnotetaking/arscontexta), a Claude Code plugin that generates personalized knowledge vaults.

## Prerequisites

- Claude Code v1.0.33+
- `tree` and `ripgrep` (`rg`) installed
- Optional: `qmd` for semantic search

## Setup Steps

### 1. Clone the repo

```bash
cd ~/Github
git clone git@github.com:cmbays/renshin.git
cd renshin
```

### 2. Install the arscontexta plugin

In a Claude Code session:

```
/plugin marketplace add agenticnotetaking/arscontexta
```

### 3. Run the setup conversation

```
/arscontexta:setup
```

This starts a ~20-minute interactive conversation where arscontexta derives your personalized knowledge system. It will ask about:

- Your domain and work style
- How you think and connect ideas
- What level of processing automation you want

**Recommended preset**: Experimental (co-design each dimension for full understanding of the system).

**Key configuration choices for Renshin**:
- **self/ space**: ON — persistent identity across sessions
- **Processing depth**: Heavy — full 6 Rs pipeline
- **Linking**: Explicit + semantic (wiki-links plus optional search)
- **Maintenance**: Condition-based triggers (not time-based)

### 4. Commit the generated vault

After setup, arscontexta generates your complete vault structure. Commit it:

```bash
git add -A
git commit -m "feat: initialize vault from arscontexta derivation"
git push
```

### 5. Connect to other projects

In each project's CLAUDE.md, add a reference:

```markdown
## External Knowledge Vault (Renshin)
Brainstorming vault: ~/Github/renshin/
Topic navigation: ~/Github/renshin/notes/index.md
Current goals: ~/Github/renshin/self/goals.md
Methodology: ~/Github/renshin/self/methodology.md
```

## Plugin Updates

When arscontexta releases updates:

```
/arscontexta:upgrade
```

This applies knowledge base updates to your vault without breaking your data.

## Daily Usage

### Start a brainstorming session

```bash
cd ~/Github/renshin
# Open Claude Code and think
```

### Process incoming research (from Tankyu)

Drop research outputs into `inbox/`, then:

```
/reduce    # Extract insights
/reflect   # Find connections
/reweave   # Update existing notes
```

### Check vault health

```
/arscontexta:health
```

### Find your next action

```
/next      # Queue-based recommendation
```

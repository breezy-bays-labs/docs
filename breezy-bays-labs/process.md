---
title: 'Process & Methodology'
description: 'How we work — Shape Up adapted for a solo developer building with AI agents'
---

# Process & Methodology

Our process is Shape Up — adapted for a team of one human and many AI agents. The core insight: when your "team" has perfect execution capability but zero memory, methodology structure becomes the primary coordination mechanism.

[Kata](https://github.com/cmbays/kata) is the tool we're building to encode and execute this methodology. What follows are the principles; Kata is the implementation.

---

## Why Shape Up

Shape Up, created by Basecamp, solves a problem that most agile methods don't: **it treats time as a fixed constraint and scope as the variable.** Instead of estimating how long something will take (and being wrong), you decide how much time a problem deserves — its appetite — and then shape the work to fit.

For a solo developer with AI agents, this is transformative:
- **No estimation theater.** AI can execute fast, but the human still has limited time for betting, reviewing, and testing.
- **Forced scoping.** Shaping work to fit an appetite prevents the "just one more feature" trap.
- **Cool-down cycles.** Mandatory breathing room between bets for reflection, polish, and forward planning.
- **Clear handoff points.** The pipeline stages create natural boundaries where the human checks in and agents can self-orient.

---

## The 8-Stage Pipeline

Every significant piece of work follows a pipeline. Not every pipeline uses every stage — the pipeline type determines which stages apply.

| Stage | Purpose | Who drives |
|-------|---------|-----------|
| **Research** | Understand the problem space, competitors, existing solutions | Agent (with human direction) |
| **Interview** | Talk to real users, gather requirements from the domain | Human (irreplaceable) |
| **Shape** | Define requirements and explore solution shapes in parallel (R x S) | Agent + human collaboration |
| **Breadboard** | Map affordances, wiring, and places — the blueprint before code | Agent + human collaboration |
| **Plan** | Break the shape into sequenced implementation waves | Agent (with human approval) |
| **Build** | Write code, tests, and documentation | Agent (with human checkpoints) |
| **Review** | Quality gate — automated checks + human verification | Agent + human |
| **Wrap-up** | Capture learnings, update documentation, close the pipeline | Agent + human |

### Pipeline Types

| Type | Stages used | When |
|------|------------|------|
| **Vertical** | All 8 stages | New feature end-to-end |
| **Horizontal** | Research through Wrap-up (skip Interview) | Cross-cutting infrastructure |
| **Polish** | Build through Wrap-up | UX refinement on existing features |
| **Bug-fix** | Build through Review | Defect resolution |

---

## Shaping: R x S Methodology

Requirements (R) and Shapes (S) are iterated in parallel. Neither comes first — they inform each other. This prevents two failure modes:

- **Requirements without shapes** produce wish lists that can't be built in the appetite
- **Shapes without requirements** produce solutions looking for problems

Key shaping principles:
- **Fit checks are binary.** Pass or fail. No "partially meets requirements."
- **Parts must be mechanisms, not intentions.** Shape parts describe what you build, not what you want.
- **Parts should be vertical slices.** Co-locate data models with the features they support. No horizontal layers.
- **Spikes investigate mechanics, not effort.** The spike gathers information; decisions happen afterward.
- **Max 9 top-level requirements.** Beyond that, group into meaningful chunks.

---

## Breadboarding: Design Before Pixels

Before any visual design or code, map the interaction structure:
- **UI affordances** — what the user can see and interact with
- **Code affordances** — what the system does in response
- **Wiring** — how affordances connect to each other and to data

The breadboard is the truth. Visual mockups and code are implementations of the breadboard, not replacements for it.

Key principles:
- **The Naming Test.** Name each affordance with one idiomatic verb. Needing "or" signals two affordances bundled together.
- **Every affordance connects to something.** Otherwise it's decoration.
- **Tables are truth; diagrams are optional visualization.** When they conflict, tables win.

---

## The Automation Trajectory

Agent autonomy increases through defined levels. Each level builds on the previous — we don't skip levels.

| Level | Name | What it means |
|-------|------|--------------|
| **L0** | Manual | Human creates issues, assigns work, manages everything |
| **L1** | Structured | Label taxonomy, issue templates, standardized pipeline stages |
| **L2** | Instrumented | Project board, auto-labeling, groomed backlog |
| **L3** | Self-Orienting | Agents read board state to find work. Pipeline stages tracked |
| **L4** | Self-Organizing | Agents propose stages, create sub-issues, update board status |
| **L5** | Autonomous | Agents detect when work is needed, self-shape, self-plan. Human gates only at Bet and Verify |

**Current position:** Between L2 and L3 across projects. The goal is to reach L4 for routine pipelines while keeping L5 as a long-term aspiration that requires significant trust infrastructure.

PM infrastructure is not just tracking — it's **the coordination layer that enables agent autonomy.** Clean taxonomy + structured templates + automated sync = agents that can self-orient without human hand-holding.

---

## GitHub as Platform

We use GitHub Issues, Projects, and PRs as the PM platform. Not Linear, not Jira, not Notion.

**Why:**
- Co-located with code — issues reference commits, PRs close issues, everything links
- `gh` CLI is agent-native — agents read board state, create issues, update labels without a browser
- No sync tax — no need to keep a separate tool in sync with the repository
- Lower lock-in — Markdown issues are portable; proprietary tool data is not

**Label taxonomy is the organizational backbone.** Every issue carries type, priority, and domain labels. Labels are machine-readable via `gh issue list -l`, making them the primary way agents discover and filter work.

---

## Control Plane and Runtime

We separate org automation into two repos:

- **`ops` (control plane)** — standards, decisions, queue definitions, and automation intent
- **`relay` (runtime)** — scheduled execution, adapters, retries, and proof-of-work outputs

This keeps governance stable while allowing runtime iteration speed and tighter credential boundaries.

---

## Cool-Down Cycles

Between bets, cool-down cycles provide structured space for:

1. **Reflection** — What worked? What was painful? What should change?
2. **Learnings capture** — Extract patterns, update principles, refine methodology
3. **Polish** — Address the small things that weren't worth stopping a bet for
4. **Forward planning** — Shape the next cycle's bets with fresh perspective

Cool-down is not optional slack time. It's where the system improves itself — the recursive loop that makes each cycle better than the last.

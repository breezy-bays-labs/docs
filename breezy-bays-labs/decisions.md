---
title: 'Decision Framework'
description: 'How we make, document, and learn from decisions — plus growth areas we are actively developing'
---

# Decision Framework

Every non-trivial decision should be visible, traceable, and learnable. Not because we're bureaucratic — because agents with amnesia will face the same decision again next session, and without a record, they'll decide differently.

---

## Lightweight ADRs

Architecture Decision Records capture the "why" behind significant technical choices. Our format is deliberately minimal:

**When to write one:**
- Choosing between two or more reasonable approaches
- Adopting or removing a dependency
- Establishing a pattern that other code will follow
- Making a trade-off that future developers (or agents) might question

**When NOT to write one:**
- The decision is obvious and uncontroversial
- It's a one-off choice that won't recur
- It's already documented in the project's operational rules

**Format:**

```
## ADR-NNN: [Decision Title]

**Status:** Proposed | Accepted | Deprecated | Superseded by ADR-XXX

**Context:** What situation prompted this decision? What constraints exist?

**Decision:** What did we decide, and why this option over the alternatives?

**Consequences:** What follows from this decision — both positive and trade-offs?
```

Keep ADRs under one page. If it takes longer to explain, the decision might be too complex — consider breaking it into smaller decisions.

---

## How Decisions Compound Into Principles

Individual decisions, repeated across projects and contexts, gradually reveal principles:

1. **First occurrence:** Make a decision. Record it in an ADR.
2. **Second occurrence:** Face a similar decision. Notice the pattern. The delta between the two choices is a learning.
3. **Third occurrence:** Elevate the pattern to a principle. It moves from project-level ADR to org-level philosophy.

This is the mechanism by which the org-level philosophy documents grow. They are not written speculatively — they emerge from the pattern of real decisions made under real constraints.

**The second time is the most valuable.** The most important learnings come from things done a second time differently. The gap between the first and second approach reveals what you actually learned.

---

## The Decision Record Chain

Decisions live at different levels with different shelf lives:

| Level | Where it lives | Shelf life | Example |
|-------|---------------|------------|---------|
| **Operational** | Project CLAUDE.md, code comments | Until next refactor | "Use CTE pre-aggregation for this query" |
| **Strategic** | Knowledge Base pipeline docs, ADRs | Months to years | "We chose clean architecture because..." |
| **Constitutional** | Org philosophy docs (these pages) | Years | "Financial precision is non-negotiable" |

Content flows upward over time. An operational decision that keeps recurring becomes strategic. A strategic pattern that survives across projects becomes constitutional.

---

## Growth Areas

These are domains where we have positions but they're not yet mature enough to document as full principles. They're listed here honestly as areas we're actively developing.

### Cost Consciousness
**Current position:** Optimize for $0 development cost. Production costs must be justified per-service. Free tiers first; pay only when usage demands it.

**What's underdeveloped:** A formal framework for evaluating when to upgrade from free to paid tiers. Decision criteria for "build vs. buy" on infrastructure.

### Security Posture
**Current position:** Validate at boundaries, never trust client input, always use authenticated user identity (not session tokens). Never commit secrets.

**What's underdeveloped:** A comprehensive security philosophy (defense in depth, principle of least privilege, threat modeling approach). Currently addressed per-project as specific rules rather than org-level principles.

### Performance Philosophy
**Current position:** Optimize after measurement, not speculatively. Build infrastructure just ahead of the vertical that needs it.

**What's underdeveloped:** A performance budget framework. When to accept latency trade-offs. How to think about perceived performance vs. actual performance.

### Error Handling
**Current position:** Validate at boundaries, trust internally. Error messages should explain how to fix the problem.

**What's underdeveloped:** A unified error taxonomy. How domain errors propagate through layers. The philosophy of graceful degradation vs. fail-fast in different contexts.

### Code Review Philosophy
**Current position:** When we identify patterns of mistakes, we create rules and automated checks. 85+ review rules, 15 concerns, 4 specialized review agents.

**What's underdeveloped:** The principles behind the rules — what makes a good review rule, how to calibrate severity, when human review adds value over automated checks.

### API Design
**Current position:** Contract-first, adapter pattern, provider-agnostic interfaces.

**What's underdeveloped:** A formal API design philosophy (REST conventions, error response standards, versioning strategy, rate limiting approach).

---

## Review Triggers

This document — and the other org philosophy pages — should be reviewed when:

- **Starting a new project.** Do the principles still apply? Are there gaps that this project reveals?
- **Completing a cool-down cycle.** What did this cycle's learnings teach us about our principles?
- **Quarterly, regardless.** Principles that haven't been tested against real work in 90 days may be aspirational rather than practical.
- **When a principle fails.** If a principle led to a bad decision, update it. Principles serve us; we don't serve principles.

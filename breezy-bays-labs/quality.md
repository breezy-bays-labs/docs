---
title: 'Quality Contract'
description: 'How we define and enforce quality — testing philosophy, UX heuristics, and the automation-first quality model'
---

# Quality Contract

Quality is not a phase. It's a constraint that shapes every decision from the start. Our quality model is built around a simple principle: **automate what can be automated, then focus human attention on what can't.**

---

## Testing Philosophy

### The Shape: Risk-Proportional Coverage

Not all code deserves the same level of testing. Coverage targets scale with risk:

| Risk Level | What it covers | Coverage expectation |
|------------|---------------|---------------------|
| **Critical** | Financial calculations, pricing logic, monetary arithmetic | 100% — zero tolerance |
| **High** | Domain rules, business logic, core services | 90%+ |
| **Medium** | Infrastructure (repositories, API routes, server actions) | 80%+ |
| **Standard** | UI components (pure logic and behavior only) | 70%+ |

The reasoning: a rounding error in a price calculation costs real money and erodes trust instantly. A minor UI glitch is noticeable but recoverable. Testing effort should match the cost of failure.

### What We Test

**Test behavior, not implementation.** Assert what the user sees or the API returns — not internal state, not function call counts, not implementation details. Tests that break when you refactor (without changing behavior) are tests that slow you down.

**Integration over mocking.** Real database queries, real API calls (against test instances), real component rendering. Mocks are acceptable only when the real thing is genuinely unavailable or prohibitively slow.

**Coverage is a floor, not a ceiling.** Meeting a coverage threshold doesn't mean the code is well-tested. It means the minimum bar has been cleared. The goal is confidence that the system works, not a percentage on a dashboard.

### TDD for Domain Logic

Domain rules and financial calculations are written test-first. The test defines the expected behavior before the implementation exists. This is not optional for critical-risk code — it's the only way to ensure the implementation matches the specification rather than the other way around.

---

## UX Quality: 10 Heuristics

Every screen is evaluated against these ten heuristics. They are tech-stack-agnostic and apply to any interactive interface.

1. **Primary task reachable within 3 interactions** from the main entry point
2. **Current state always visible** — the user never has to guess what mode they're in, what's selected, or what's pending
3. **User can undo or recover from mistakes** — destructive actions require confirmation; recoverable actions are one step away
4. **Keyboard shortcuts are discoverable** — available for frequent actions, shown in tooltips or help panels
5. **Progressive disclosure works** — detail available on demand without cluttering the default view
6. **Empty states are helpful** — first-time views guide the user toward their first action, not just show a blank screen
7. **Loading states are informative** — skeletons or progress indicators, not spinners that convey no information
8. **Error messages explain how to fix the problem** — not just what went wrong, but what to do next
9. **Help is accessible without leaving context** — tooltips, inline hints, contextual guidance — never "click here to read the docs"
10. **The interface feels consistent** — same patterns for same actions across the entire application

---

## Visual Quality: The Screen Audit

Before any screen ships, it passes a visual quality audit organized into four dimensions:

### Visual Integrity
- **Hierarchy is clear.** Primary action is most prominent. Secondary elements recede.
- **Spacing uses a consistent scale.** No magic numbers.
- **Typography is restrained.** Maximum 3-4 sizes, maximum 3 weights.
- **Color serves meaning.** Monochrome base, status colors only for semantic purpose.
- **Alignment is pixel-precise.** Elements that should share a lane actually share it.

### Component Consistency
- **Shared component library.** No one-off implementations of standard patterns.
- **Icon system is uniform.** Same library, consistent sizes, no mixing sources.
- **Interactive states complete.** Every interactive element has hover, focus-visible, active, and disabled states.
- **Motion uses consistent tokens.** Timing and easing are standardized, not per-component.

### State Handling
- **All five states designed.** Default, empty, loading, error, success — no missing states.
- **Error states are recoverable.** Clear path back to a working state.
- **Loading states prevent layout shift.** Skeletons match the shape of the content they replace.

### Accessibility
- **4.5:1 contrast minimum** for all text.
- **Keyboard navigation works** end-to-end.
- **Focus indicators are visible** and follow a logical tab order.
- **ARIA labels are meaningful** — not just present, but accurate.

---

## Where Human Review Matters

Most quality enforcement is automated — linters, test suites, CI gates, design system tokens. But some quality dimensions resist automation:

**Aesthetic judgment.** Does this feel right? Does it match the project's personality? Does it spark the right emotional response? AI can follow rules; only the human can say "this is good but it doesn't feel like us."

**Domain appropriateness.** Does this workflow match how the real user actually works? Not how we imagine they work, but how we've observed them work? This requires direct experience with the domain.

**Strategic coherence.** Does this feature support the product direction? Should we be building this at all? This is a betting decision, not a quality decision.

The quality contract's job is to automate everything below these human-judgment moments, so that when the human does review, they're focused on the irreducible questions — not catching spacing errors.

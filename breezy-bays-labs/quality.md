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

### CRAP Analysis: Identifying Risky Functions

Before running mutation tests, identify which functions are most at risk using CRAP analysis.
[crap4ts](https://github.com/breezy-bays-labs/crap4ts) computes per-function CRAP scores
that combine cyclomatic complexity with test coverage.

**CRAP(m) = CC(m)² × (1 - cov(m)/100)³ + CC(m)**

A high CRAP score means a function is both complex AND poorly tested — risky to change.
The formula has a critical property: you cannot test your way out of complexity.
At CC=12, even 100% coverage produces CRAP=12.

| Risk Level | CRAP Score | Action |
|------------|:----------:|--------|
| Low | ≤ 5 | No action needed |
| Acceptable | ≤ 8 | Monitor |
| Moderate | ≤ 30 | Prioritize for refactoring or better test coverage |
| High | > 30 | Decompose — function is too complex to safely change |

CRAP analysis fits between TDD and mutation testing in the quality loop:
BDD scenarios → TDD → **CRAP ≤ 8** → Mutation testing → Architecture enforcement

### Mutation Testing: Verifying Test Quality

Coverage tells you what code your tests *execute*. Mutation testing tells you what code your tests *actually verify*. A test suite can hit 100% line coverage while asserting nothing meaningful — mutation testing catches this.

**How it works:** The mutation testing tool (Stryker) makes small, systematic changes to your source code — called "mutants" — like flipping `>` to `>=`, removing a condition, swapping `true` for `false`, or replacing `+` with `-`. For each mutant, it runs your test suite. If a test fails, the mutant is "killed" (good — your tests caught the change). If all tests pass, the mutant "survived" (bad — your tests didn't notice the behavior change).

**The mutation score** is the percentage of mutants killed. A survived mutant means there's a code path where behavior could change silently without any test catching it.

#### When to Use Mutation Testing

Mutation testing is computationally expensive — it runs your full test suite once per mutant, which can mean hundreds or thousands of runs. Use it strategically:

| Risk Level | Mutation testing? | Target score |
|------------|------------------|-------------|
| **Critical** | Required — run on every PR touching this code | 90%+ |
| **High** | Required — run before release | 80%+ |
| **Medium** | Recommended — run periodically | 70%+ |
| **Standard** | Optional — use to audit test quality when uncertain | No threshold |

#### Setup Pattern (All Repos)

Every TypeScript/JavaScript repo should include Stryker configuration. The standard setup:

1. **Install dependencies:** `@stryker-mutator/core`, the runner plugin for your test framework (e.g., `@stryker-mutator/vitest-runner`), and `@stryker-mutator/typescript-checker`
2. **Config file:** `stryker.config.json` at repo root
3. **npm scripts:** `test:mutation` (full run) and `test:mutation:incremental` (faster reruns, only re-tests changed files)
4. **Gitignore:** `.stryker-tmp/` and `reports/mutation/`

#### Configuration Best Practices

- **Scope the `mutate` glob to domain and feature code.** Exclude CLI commands, program entry points, and pure wiring code — these are best tested by integration/E2E tests, and mutating them generates noise.
- **Use `coverageAnalysis: "perTest"`** for faster runs — Stryker only runs tests that cover each mutant.
- **Set `thresholds.break`** to fail CI when the score drops below a minimum. Start at 50% and ratchet up as test quality improves.
- **Use incremental mode** (`--incremental`) during development. It caches results and only re-tests mutants affected by changes.
- **Use `concurrency`** to parallelize — set to your CPU core count minus 1.

#### Interpreting Results

When a mutant survives, ask:

1. **Is this a meaningful behavior change?** If flipping a condition from `>` to `>=` doesn't change any observable behavior, the mutant is equivalent (false positive). Mark it as ignored.
2. **Is there a missing test?** Most survived mutants indicate a gap — a branch not tested, a return value not asserted, or an edge case not covered. Write the test.
3. **Is the code itself unnecessary?** Sometimes a survived mutant reveals dead code or a redundant condition. If removing the code doesn't change behavior, remove it.

#### CI Integration

- **PR checks (optional):** Run mutation testing on changed files only (`--incremental` + `--mutate` scoped to changed paths). This keeps CI fast while catching test quality regressions on new code.
- **Nightly/weekly (recommended):** Full mutation run on critical and high-risk modules. Report the score and trend over time.
- **Release gates (for critical code):** Block release if mutation score for critical modules drops below the threshold.

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

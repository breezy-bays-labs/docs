---
title: 'Design Philosophy'
description: 'What our software should feel like — principles that guide every visual and interaction decision'
---

# Design Philosophy

Our design philosophy serves two purposes: it provides guardrails that help AI agents produce consistently high-quality design decisions, and it encodes the aesthetic taste and quality bar that the human brings to every project.

These principles are tech-stack-agnostic. Specific tokens, components, and frameworks vary per project — but the thinking behind them stays the same.

---

## The Jobs Filter

Every element must earn its place. This is the single most important design principle.

Before anything ships, apply these questions:

1. **"Would a user need to be told this exists?"** If yes, redesign it until obvious.
2. **"Can this be removed without losing meaning?"** If yes, remove it.
3. **"Does this feel inevitable, like no other design was possible?"** If no, it's not done.
4. **"Is this detail as refined as the details users will never see?"** Paint the back of the fence.
5. **"Say no to 1,000 things."** Cut good ideas to keep great ones.

The Jobs Filter is recursive — it applies to itself. If a design principle can be removed without losing meaning, remove it.

---

## Three Layers: Calm, Polish, Delight

Our design language operates in three layers that create contrast through restraint:

**Calm by Default.** The base layer is monochrome, restrained, and quiet. Most of the interface is neutral — backgrounds, text, borders, containers. This calm base is not boring; it's a canvas that makes everything else stand out.

**Polished in Interaction.** When users interact with elements, the response should feel precise and native — smooth transitions, appropriate feedback, responsive touch targets. Polish is invisible when done right; you notice its absence, not its presence.

**Delightful Where It Matters.** Bold color, spring animations, and attention-grabbing moments are reserved for the few elements that truly deserve them. The contrast between calm base and bold accents makes delight moments pop harder. Delight that's everywhere is delight that's nowhere.

---

## Color Communicates Meaning, Never Decorates

Color is a functional tool, not a styling choice. In our interfaces:

- **Status colors** (success, error, warning) convey system state
- **Action colors** identify interactive elements and primary calls to action
- **The base palette** is monochrome — everything else is built on this neutral foundation

No decorative gradients. No color for visual interest. If you remove all color from a screen and it loses information, the color was doing its job. If the screen looks the same in grayscale, the color was decoration.

---

## Typography Creates Hierarchy, Not Noise

Restraint in typography produces clarity:

- **Maximum 3-4 font sizes per screen.** If you need more, the information architecture is wrong.
- **Maximum 3 font weights.** Regular for body, medium for emphasis, bold for headings. That's enough.
- **Opacity for secondary hierarchy.** Use reduced opacity to de-emphasize text rather than introducing new colors. Two opacity levels (high-emphasis and medium-emphasis) handle most cases.
- **Type hierarchy must be scannable in under 2 seconds.** A user should understand what's most important, what's secondary, and what's tertiary at a glance.

---

## Generous Whitespace

When in doubt, add more space. Generous spacing signals confidence and clarity. Cramped layouts signal that the designer was afraid to waste pixels.

Use an 8px base scale for consistency. Group related elements with tighter spacing; separate distinct sections with generous spacing. The rhythm of tight and loose spacing creates visual hierarchy without any additional decoration.

---

## Motion Is Meaningful

Animation serves three purposes and no others:

1. **Orientation** — help users understand where they are and where things went (page transitions, list reordering)
2. **Feedback** — confirm that an action was received (button press, form submission)
3. **Delight** — spring-based transitions on key moments that reward interaction

All motion respects reduced-motion preferences. Animation that doesn't serve one of these three purposes gets removed.

---

## Accessibility Is the Baseline, Not a Feature

Accessibility is not a checklist item to complete after the design is "done." It's a constraint that shapes design from the start:

- **4.5:1 contrast minimum** for all text. No exceptions for aesthetic preference.
- **Keyboard navigable.** Every interactive element reachable without a mouse.
- **Visible focus states.** Users who navigate by keyboard must always know where they are.
- **Meaningful ARIA labels.** Screen readers get the same information as sighted users.
- **Logical heading hierarchy.** h1 → h2 → h3, always in order, never skipping levels.

---

## Five States Per Screen

Every screen has at least five states. Designing only the "happy path" is not designing — it's sketching.

| State | What it shows | Why it matters |
|-------|--------------|----------------|
| **Default** | Normal view with data | The primary design |
| **Empty** | No data yet, first-time experience | First impressions; guides the user toward their first action |
| **Loading** | Data is being fetched | Reduces perceived wait time; prevents layout shift |
| **Error** | Something went wrong | Must explain what happened AND how to fix it |
| **Success** | Action completed | Confirms the result; provides a clear next step |

If a screen doesn't have all five states designed, it's not ready to build.

---

## The 5-Second Rule

A user should understand the screen's current state within 5 seconds. Not after reading labels, not after scrolling, not after clicking — at first glance.

If a screen fails this test, the visual hierarchy is wrong. The primary action should be the most prominent element. Status information should be immediately visible. The user's next step should be obvious.

---

## Progressive Disclosure Over Information Overload

Start with the essential. Reveal details on demand. This applies at every level:

- **Screen level:** Show the summary first, expand details on click
- **Form level:** Show required fields first, reveal optional fields when needed
- **Navigation level:** Primary actions visible, secondary actions in menus
- **Documentation level:** Principles first, implementation details behind links

The right amount of information is the minimum needed to make a decision. Everything else is available one interaction away.

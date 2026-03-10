---
title: 'Foundations'
description: 'Public foundations for color, type, spacing, motion, and state-aware interface design'
---

# Foundations

## Core Principles

- Calm by default
- Color communicates meaning
- Typography creates hierarchy
- Whitespace is structural
- Motion is meaningful

## Color

The base palette is restrained and neutral.

Accent color is reserved for action and system meaning, not decoration. Status colors exist to communicate state, risk, or success. Decorative color should be rare.

## Typography

Type should be readable, scannable, and restrained.

- Limit the number of sizes on a screen
- Limit the number of font weights in a flow
- Use hierarchy to make the screen legible within a few seconds

## Spacing

Spacing is part of the system, not an afterthought.

- Use a consistent spacing scale
- Group related elements tightly
- Separate distinct sections generously

## Motion

Motion should support:

- orientation
- feedback
- carefully chosen delight

Motion that does not clarify or improve the experience should be removed.

## States

Every screen and reusable pattern should account for:

- default
- empty
- loading
- error
- success

The system is not complete if only the happy path has been designed.

## Token Architecture

The system uses a four-layer token model:

1. **Foundation** — Mode-aware tokens for surfaces, text hierarchy, status colors, borders, and spacing. These form the baseline that all products inherit.
2. **Categorical** — Entity and service identity colors. Each domain concept receives a stable color that persists across modes and personalities.
3. **Semantic** — Purpose-driven tokens that extend the foundation for gaps the base layer does not cover. Names stay stable; values adapt to personality and mode.
4. **Personality** — Visual expression overrides applied at the root level. A personality changes how the system renders without changing what the system communicates.

Tokens are consumed through utility classes. Raw values are never used in component code.

## Personality System

The system supports multiple visual personalities — distinct aesthetic treatments that share the same underlying architecture. Each personality defines its own surface treatments, accent styles, shadow behavior, and motion character.

Personalities are additive: the foundation and categorical layers remain stable, while the semantic and personality layers adapt. Adding a personality requires one override definition and one registry entry. No component code changes.

This allows a single product to offer distinct visual experiences — or multiple products to share the same design system with different brand expressions.

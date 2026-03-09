---
title: 'Implementation Stack'
description: 'Public implementation defaults used to build design-system-aligned product UI'
---

# Implementation Stack

## Goal

We prefer an implementation stack that keeps UI work fast, inspectable, and easy to evolve.

## Current Defaults

- Next.js App Router
- Server Components by default
- Tailwind for utility styling and token consumption
- Radix primitives for accessible interaction behavior
- shadcn/ui as the baseline open-code component model

## Forms

For forms, the default pattern is:

- Zod for validation
- React Hook Form for field state and wiring

## Data And Mutations

The default UI-facing data boundary favors:

- Server Actions for straightforward mutations
- typed contracts where richer interaction requires them

## Why This Matters

This stack keeps the source code local and readable. That matters for maintainability, and it matters for AI-assisted workflows because the implementation is visible instead of hidden behind opaque abstractions.

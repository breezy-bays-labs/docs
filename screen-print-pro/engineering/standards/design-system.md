---
title: 'FRONTEND_GUIDELINES'
description: 'Design tokens, component patterns, animation, accessibility, and layout standards. Tailwind v4 + shadcn/ui implementation.'
category: reference
status: active
phase: all
last_updated: 2026-02-07
last_verified: 2026-02-07
depends_on:
  - docs/TECH_STACK.md
---

# Frontend Guidelines

---

## Design Philosophy

### The Standard: "Linear Calm + Raycast Polish + Neobrutalist Delight"

Our aesthetic combines three influences into a cohesive system:

| Layer         | Influence    | Treatment                                               |
| ------------- | ------------ | ------------------------------------------------------- |
| **Base**      | Linear       | Monochrome, opacity hierarchy, extreme restraint        |
| **Polish**    | Raycast      | OS-native feel, subtle glass, responsive transitions    |
| **Attention** | Neobrutalist | Bold borders, vibrant status colors, springy animations |

**Core Insight**: The contrast between calm base and bold accents makes attention elements pop harder.

### Design Principles

1. **Calm by Default**: Most UI should be monochrome and restrained. No color unless it serves a purpose.
2. **Delightful When It Matters**: Primary CTAs, success moments, and interactive states get the bold treatment.
3. **Opacity Over Color**: Use opacity levels (87%, 60%, 38%) for text hierarchy instead of different colors.
4. **Status-Driven Color**: Color communicates meaning (action, success, error, warning), not decoration.
5. **Remove Until It Breaks**: Apply the Jobs filter — if an element can be removed without losing meaning, remove it.

### Anti-Patterns (Avoid)

- Multiple accent colors competing for attention
- Gradients and decorative elements that don't serve function
- Dense layouts with insufficient whitespace
- Color used for decoration rather than communication
- Inconsistent component styling across pages

### The Jobs Filter

Ask these questions of every UI element:

- "Would a user need to be told this exists?" -> If yes, redesign it until obvious
- "Can this be removed without losing meaning?" -> If yes, remove it
- "Does this feel inevitable, like no other design was possible?" -> If no, it's not done

---

## Architecture

### Tailwind v4 + shadcn/ui

All styling uses Tailwind utilities. No separate CSS files, no CSS modules, no styled-components.

```
app/
  globals.css               # @theme inline (design tokens) + Tailwind directives
  layout.tsx                # Root layout (fonts via next/font)
components/
  ui/                       # shadcn/ui primitives (button, card, dialog, etc.)
  features/                 # Domain components (StatusBadge, KanbanBoard)
  layout/                   # Shell components (Sidebar, Topbar)
```

### How Styling Works

1. **Design tokens** live in `globals.css` via Tailwind v4's `@theme inline` block
2. **Component variants** use `class-variance-authority` (CVA) via shadcn/ui
3. **Conditional classes** use `cn()` from `lib/utils.ts` (clsx + tailwind-merge)
4. **Never** concatenate className strings — always use `cn()`

```tsx
// Correct
<div className={cn("rounded-md border", isActive && "border-action")} />

// Wrong
<div className={"rounded-md border " + (isActive ? "border-action" : "")} />
```

### Fonts

Loaded via `next/font` in `app/layout.tsx`. No Google Fonts `<link>` tags.

```tsx
import { Inter, JetBrains_Mono } from 'next/font/google'

const inter = Inter({ subsets: ['latin'], variable: '--font-inter' })
const jetbrainsMono = JetBrains_Mono({ subsets: ['latin'], variable: '--font-jetbrains-mono' })
```

- **Inter**: All UI text (headings, body, labels, buttons)
- **JetBrains Mono**: Code blocks and technical values only (job numbers, SKUs)

---

## Color System

### Design Philosophy: Monochrome + Status

**Base Layer**: Near-black monochrome with opacity-based text hierarchy (Linear-inspired)
**Attention Layer**: Vibrant status colors for elements that need attention (Neobrutalist)

### Design Tokens

Defined in `app/globals.css` via `@theme inline`:

```css
@theme inline {
  /* Background - Ghostty Niji warm dark */
  --color-bg-primary: #141515;
  --color-bg-secondary: #111213;
  --color-bg-elevated: #1c1d1e;
  --color-bg-surface: #232425;

  /* Text - Opacity-based hierarchy */
  --color-text-primary: rgba(255, 255, 255, 0.87);
  --color-text-secondary: rgba(255, 255, 255, 0.6);
  --color-text-muted: rgba(255, 255, 255, 0.38);

  /* Borders */
  --color-border-subtle: rgba(255, 255, 255, 0.08);
  --color-border-default: rgba(255, 255, 255, 0.12);
  --color-border-strong: rgba(255, 255, 255, 0.2);

  /* Status colors - Niji palette attention layer */
  --color-action: #2ab9ff; /* Niji blue - primary CTAs */
  --color-action-hover: #1da0e0;
  --color-success: #54ca74; /* Niji green - completions */
  --color-success-hover: #43a860;
  --color-error: #d23e08; /* Niji red - failures */
  --color-error-hover: #b33407;
  --color-warning: #ffc663; /* Niji gold - cautions */
  --color-warning-hover: #e6b050;

  /* Effects */
  --glass-bg: rgba(255, 255, 255, 0.05);
  --glass-blur: blur(12px);

  /* Neobrutalist */
  --shadow-brutal: 4px 4px 0px;

  /* Radii */
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;
}
```

### Color Usage

| Use Case        | Tailwind Class              | Notes               |
| --------------- | --------------------------- | ------------------- |
| Page background | `bg-background`             | Niji dark (#141515) |
| Card/panel      | `bg-card`                   | Elevated (#1c1d1e)  |
| Body text       | `text-foreground`           | 87% opacity white   |
| Secondary text  | `text-muted-foreground`     | 60% opacity white   |
| Primary CTA     | `text-action` / `bg-action` | Niji blue           |
| Success         | `text-success`              | Niji green          |
| Error           | `text-error`                | Niji red            |
| Warning         | `text-warning`              | Niji gold           |

### When to Use Color

**Use monochrome for:**

- All non-interactive text
- Borders and dividers
- Backgrounds
- Secondary buttons

**Use status colors for:**

- Primary action buttons (Niji blue)
- Production state badges (varies by state)
- Error messages (Niji red)
- Warnings and pending states (Niji gold)
- Completions and shipped states (green)

---

## Typography

### Type Scale

Use Tailwind's built-in text utilities. Max 3-4 distinct sizes per screen.

| Tailwind Class | Size | Use                                  |
| -------------- | ---- | ------------------------------------ |
| `text-xs`      | 12px | Captions, badge labels               |
| `text-sm`      | 14px | Secondary text, table cells, buttons |
| `text-base`    | 16px | Body text                            |
| `text-lg`      | 18px | Emphasis, card titles                |
| `text-xl`      | 20px | Section headings (H3)                |
| `text-2xl`     | 24px | Page headings (H2)                   |

### Font Weights

Max 3 weights per screen:

| Weight | Tailwind Class  | Use                        |
| ------ | --------------- | -------------------------- |
| 400    | `font-normal`   | Body text, descriptions    |
| 500    | `font-medium`   | Labels, nav items, buttons |
| 600    | `font-semibold` | Headings, emphasis         |

### Heading Pattern

```tsx
// Page header
<h1 className="text-2xl font-semibold tracking-tight">Dashboard</h1>
<p className="text-sm text-muted-foreground">Production overview for 4Ink</p>

// Section header
<h2 className="text-lg font-semibold">In Progress</h2>

// Card title
<h3 className="text-sm font-medium text-muted-foreground">Blocked</h3>
```

---

## Spacing

### Spacing Scale (8px base)

Use Tailwind spacing utilities exclusively. No hardcoded pixel values.

| Tailwind        | Value | Use                             |
| --------------- | ----- | ------------------------------- |
| `p-1` / `gap-1` | 4px   | Tight spacing (badge padding)   |
| `p-2` / `gap-2` | 8px   | Related elements (icon + label) |
| `p-3` / `gap-3` | 12px  | Component padding               |
| `p-4` / `gap-4` | 16px  | Standard gap between elements   |
| `p-6` / `gap-6` | 24px  | Section padding, card padding   |
| `p-8` / `gap-8` | 32px  | Large section gaps              |

### Spacing Philosophy

**Japanese Minimalism (Ma)**: Use generous spacing. When in doubt, add more space.

```tsx
// Too dense - avoid
<div className="space-y-2">

// Better - let content breathe
<div className="space-y-4">
```

---

## Components

### Using shadcn/ui

Always check `components/ui/` before creating custom components. shadcn/ui provides accessible, styled primitives.

**Adding a component**: `npx shadcn@latest add <component>`

**Installed**: button, card, dialog, input, table, badge, dropdown-menu, tabs, separator, tooltip, label, select, textarea, sheet, breadcrumb, avatar, form

### Component Styling Patterns

#### Primary Button (Neobrutalist)

The primary CTA gets the neobrutalist treatment: bold shadow, spring hover, dark text on Niji blue.

```tsx
<Button className="bg-action text-black font-semibold border-2 border-current shadow-brutal shadow-action hover:translate-x-[-2px] hover:translate-y-[-2px] hover:shadow-brutal-lg active:translate-x-0 active:translate-y-0 active:shadow-brutal-sm transition-all">
  New Quote
</Button>
```

#### Cards

Three card treatments matching design layers:

```tsx
// Base card (default shadcn/ui) — calm, monochrome
<Card>
  <CardHeader>
    <CardTitle>In Progress</CardTitle>
  </CardHeader>
  <CardContent>...</CardContent>
</Card>

// Glass card — Raycast polish
<div className="rounded-lg border border-white/10 bg-white/5 backdrop-blur-xl p-6">
  ...
</div>

// Interactive card — Neobrutalist hover
<div className="rounded-lg border-2 border-border bg-card p-6 cursor-pointer transition-all hover:translate-x-[-2px] hover:translate-y-[-2px] hover:shadow-brutal hover:shadow-action hover:border-action">
  ...
</div>
```

#### Status Badges

Map production states to colors via constants:

```tsx
// From lib/constants.ts
const PRODUCTION_STATE_COLORS: Record<ProductionState, string> = {
  design: 'text-muted-foreground',
  approval: 'text-warning',
  burning: 'text-action',
  press: 'text-action',
  finishing: 'text-success',
  shipped: 'text-success',
}

// Usage
;<Badge variant="outline" className={PRODUCTION_STATE_COLORS[job.status]}>
  {PRODUCTION_STATE_LABELS[job.status]}
</Badge>
```

---

## Animations & Motion

### Motion Philosophy

- **Subtle by default**: Tailwind `transition-*` utilities for hover/focus
- **Springy for delight**: Framer Motion springs for layout changes and celebrations
- **Respect preferences**: Always check `prefers-reduced-motion`

### Tailwind Transitions (most interactions)

```tsx
// Hover effects — use Tailwind, not Framer Motion
<button className="transition-colors hover:bg-accent">
<div className="transition-all hover:translate-y-[-2px]">
```

### Framer Motion (layout changes)

```tsx
// Page transitions, card enter/exit, Kanban column moves
import { motion, AnimatePresence } from 'framer-motion'
;<AnimatePresence>
  <motion.div
    initial={{ opacity: 0, y: 10 }}
    animate={{ opacity: 1, y: 0 }}
    exit={{ opacity: 0, y: -10 }}
    transition={{ type: 'spring', stiffness: 300, damping: 30 }}
  >
    {children}
  </motion.div>
</AnimatePresence>
```

### Reduced Motion

```tsx
// Framer Motion respects this automatically via useReducedMotion()
// For custom animations, wrap in media query:
<motion.div
  animate={{ y: 0 }}
  transition={{
    type: prefersReducedMotion ? 'tween' : 'spring',
    duration: prefersReducedMotion ? 0 : undefined,
  }}
/>
```

---

## UI States

Every page must handle these states explicitly.

### Empty State

```tsx
import { Package } from 'lucide-react'
;<div className="flex flex-col items-center justify-center py-12 text-center">
  <Package className="h-12 w-12 text-muted-foreground/50 mb-4" />
  <h3 className="text-lg font-semibold text-muted-foreground">No jobs yet</h3>
  <p className="text-sm text-muted-foreground/60 mt-1 max-w-xs">
    Jobs will appear here once they're created.
  </p>
</div>
```

### Loading State (Phase 2+)

```tsx
// Skeleton pattern for future API integration
<div className="animate-pulse space-y-3">
  <div className="h-4 w-3/4 rounded bg-muted" />
  <div className="h-4 w-1/2 rounded bg-muted" />
</div>
```

### Error State

```tsx
<div className="rounded-md border border-error/30 bg-error/10 p-4" role="alert">
  <p className="text-sm font-medium text-error">Job not found</p>
  <p className="text-sm text-muted-foreground mt-1">
    This job may have been removed.{' '}
    <Link href="/jobs" className="text-action underline">
      Back to Jobs
    </Link>
  </p>
</div>
```

---

## Accessibility (WCAG AA)

### Focus Indicators

All interactive elements must have visible `:focus-visible` states. shadcn/ui handles this by default.

```tsx
// Custom focus when needed
<button className="focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-action">
```

### Color Contrast

| Combination                    | Ratio | Status |
| ------------------------------ | ----- | ------ |
| White (87%) on #141515         | 15:1  | AAA    |
| White (60%) on #141515         | 10:1  | AAA    |
| White (38%) on #141515         | 6.5:1 | AAA    |
| Niji blue (#2ab9ff) on #141515 | 9:1   | AAA    |

### Keyboard Navigation

| Key        | Action                                 |
| ---------- | -------------------------------------- |
| Tab        | Move to next focusable element         |
| Shift+Tab  | Move to previous element               |
| Enter      | Activate button, submit form           |
| Escape     | Close modal/dialog                     |
| Arrow keys | Navigate within tabs, menus, dropdowns |

### ARIA Requirements

shadcn/ui components handle ARIA automatically (built on Radix primitives). For custom components:

```tsx
// Dynamic status messages
<div role="status" aria-live="polite">Quote saved</div>
<div role="alert" aria-live="assertive">Validation error</div>

// Icon-only buttons
<Button variant="ghost" size="icon" aria-label="Close dialog">
  <X className="h-4 w-4" />
</Button>
```

---

## Responsive Design

### Desktop-First

Screen Print Pro is designed for shop office desktop use. Optimize for cursor interaction and comfortable reading at typical desktop widths.

**Design Priority**: Desktop (primary) -> Tablet (Phase 2) -> Mobile (Phase 2+)

### Breakpoints

| Width   | Target            |
| ------- | ----------------- |
| 1280px  | Most laptops      |
| 1440px  | External monitors |
| 1920px+ | Large displays    |

### Layout Patterns

```tsx
// Sidebar + main content (dashboard layout)
<div className="flex h-screen">
  <Sidebar />  {/* w-60, fixed */}
  <main className="flex-1 overflow-y-auto p-6">{children}</main>
</div>

// Summary cards grid
<div className="grid grid-cols-4 gap-4">

// Two-column detail layout (job detail)
<div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
  <div className="lg:col-span-2">...</div>  {/* Main content */}
  <div>...</div>                              {/* Sidebar info */}
</div>
```

---

## Dark Mode

Dark mode is the default. Applied via `className="dark"` on `<html>` in `app/layout.tsx`.

shadcn/ui handles dark mode token mapping automatically. Custom elements should use Tailwind's semantic color classes (`bg-background`, `text-foreground`, `border-border`) rather than hardcoded hex values.

---

## Related Documents

- [CLAUDE.md](../../CLAUDE.md) — Design system summary, quality checklist
- [TECH_STACK.md](../TECH_STACK.md) — Tool choices including styling stack
- [SCREEN_AUDIT_PROTOCOL.md](./SCREEN_AUDIT_PROTOCOL.md) — 15-point quality audit
- [UX_HEURISTICS.md](./UX_HEURISTICS.md) — 10-point UX quality checklist

---

## Version History

| Date       | Change                                                            |
| ---------- | ----------------------------------------------------------------- |
| 2026-02-04 | Initial guidelines (dbt-playground context)                       |
| 2026-02-07 | Adapted for Screen Print Pro: Tailwind v4 + shadcn/ui + next/font |

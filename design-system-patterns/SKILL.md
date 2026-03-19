---
name: design-system-patterns
description: Explains design system patterns including design tokens, component layers, API design, accessibility defaults, theming, documentation, and governance. Use when structuring a reusable component system, defining tokens and variants, or deciding how teams should build and evolve shared UI.
---

# Design System Patterns

## Overview

A design system is not just a component library.

It is the shared contract between:

- design decisions
- reusable UI primitives
- accessibility expectations
- documentation
- contribution rules
- release and adoption workflows

Good design systems make teams faster and more consistent.

Bad design systems become a second product that nobody fully trusts.

## What a Design System Usually Contains

| Layer | Purpose | Examples |
|------|---------|----------|
| Foundations | Shared visual decisions | Type scale, color, spacing, motion |
| Tokens | Encoded design values | `--space-3`, `--color-text-muted` |
| Primitives | Low-level reusable building blocks | `Box`, `Text`, `Stack`, `Button` |
| Composed components | Higher-order UI patterns | `Modal`, `FormField`, `Table`, `Toast` |
| Guidance | Usage rules and examples | Docs, do/don’t guidance, recipes |
| Governance | Change control | ownership, review, versioning, migration |

If one of these is missing, teams usually compensate with ad hoc workarounds.

## Goals of a Design System

A healthy design system should help teams:

- build consistent UI faster
- reduce duplicate component logic
- encode accessibility as a default
- make design decisions reusable rather than one-off
- support change without rewriting every app screen

It should not try to eliminate all flexibility.

## Tokens First

Design tokens are the smallest reusable design decisions.

Common token categories:

- color
- spacing
- typography
- radius
- shadow
- motion
- z-index

### Raw Tokens vs Semantic Tokens

Raw tokens:

```css
--gray-100: #f5f5f5;
--blue-600: #2563eb;
--space-4: 16px;
```

Semantic tokens:

```css
--color-surface: var(--gray-100);
--color-action-primary: var(--blue-600);
--space-control-padding: var(--space-4);
```

Raw tokens define a scale.
Semantic tokens define meaning.

Semantic tokens are what make themes and redesigns easier, because components depend on meaning rather than a hardcoded visual choice.

## Token Rules of Thumb

- Prefer semantic names where components consume tokens
- Keep scales predictable
- Avoid one-off tokens for single components unless they are proven patterns
- Use CSS variables or an equivalent runtime-friendly token format when theming matters
- Keep token naming boring and stable

If token names encode current visual taste instead of intent, the system becomes fragile.

## Component Layers

Most systems work better with clear layers instead of one giant component pile.

### 1. Primitives

Low-level building blocks with minimal opinion:

- `Box`
- `Text`
- `Stack`
- `Inline`
- `Button`

These should be composable and predictable.

### 2. Patterns / Composites

Composed building blocks for recurring UI:

- `FormField`
- `Card`
- `Dialog`
- `Tabs`
- `Toast`

These encode more behavior and structure.

### 3. Product-Specific Components

App- or feature-level components that belong outside the core design system:

- `InvoiceStatusCard`
- `SubscriptionUpgradeBanner`
- `TeamMemberRow`

Not every reusable component belongs in the shared system.

## API Design for Components

Component APIs should be small, composable, and difficult to misuse.

### Good API Characteristics

- predictable prop names
- clear variant model
- accessible behavior by default
- escape hatches without encouraging chaos

### Prefer Composition Over Prop Explosion

Hard-to-maintain API:

```jsx
<Button
  primary
  danger
  rounded
  fullWidth
  iconLeft="plus"
  iconOnly
  dense
  loading
/>
```

Better:

```jsx
<Button variant="danger" size="sm" loading>
  <IconPlus />
  Create
</Button>
```

If a component needs dozens of boolean props, the abstraction is probably wrong.

## Variants, Sizes, and States

A design system usually needs a clear model for:

- variants
- sizes
- states
- slots

### Variant Model

Good examples:

- `variant="primary" | "secondary" | "ghost"`
- `size="sm" | "md" | "lg"`
- `tone="success" | "warning" | "danger"`

Avoid variants that encode page-specific meaning like `dashboardPrimary` or `settingsAlt`.

### State Model

Components should define expected behavior for:

- hover
- focus
- pressed
- disabled
- loading
- invalid

These states should not be reinvented in every screen.

## Slots and Composition

Complex components often need named regions instead of giant prop bags.

Example:

```jsx
<Dialog>
  <Dialog.Header />
  <Dialog.Body />
  <Dialog.Footer />
</Dialog>
```

This is often more maintainable than a component with many “content props”.

Slots are especially helpful when layout structure should remain consistent but content should vary.

## Accessibility as a System Default

Accessibility should be built into the system, not left to each feature team.

System-level defaults should cover:

- keyboard interaction
- focus styling
- semantic markup
- color contrast
- reduced-motion handling
- error and busy states

If the design system makes the accessible path harder than the inaccessible path, teams will drift.

## Styling Strategy

A design system can work with different styling approaches:

- CSS variables
- utility classes
- CSS Modules
- CSS-in-JS
- static extracted styles

The important part is not the styling brand. It is whether the approach supports:

- tokens
- theming
- predictable override behavior
- component portability
- runtime and build performance needs

The system should not depend on “magic” that only one or two maintainers understand.

## Theming

Theming works best when it is token-driven.

Common theme axes:

- light and dark mode
- brand or tenant customization
- density
- motion preference

### Theme Pattern

```css
:root {
  --color-surface: white;
  --color-text: #111;
}

[data-theme="dark"] {
  --color-surface: #111;
  --color-text: white;
}
```

Components should consume semantic tokens like `--color-surface` rather than branching manually in each component.

## Documentation Patterns

A design system is only useful if teams can discover how to use it.

Good docs usually include:

- when to use the component
- when not to use it
- examples
- variant guidance
- accessibility notes
- content guidance when relevant

### Useful Documentation Types

- component API docs
- design rationale
- do/don’t examples
- migration notes
- layout and composition recipes

Docs should answer real usage questions, not just dump prop tables.

## Contribution and Governance

A design system needs rules for what gets added.

Good governance questions:

- Is this pattern reused across products?
- Can an existing component solve it with composition?
- Is the proposed API stable enough to support long-term?
- Does it improve or harm consistency?
- Who owns maintenance after merge?

Without governance, systems grow into inconsistent wrappers around product-specific needs.

## Versioning and Change Management

Shared UI systems need stable change practices.

Useful patterns:

- semantic versioning
- deprecation before removal
- migration guides
- codemods for common API changes
- changelogs focused on adoption impact

Breaking changes in a design system ripple into many applications quickly.

## Adoption Patterns

A design system succeeds only if product teams actually use it.

Common adoption strategies:

- start with a few strong primitives
- replace high-duplication components first
- provide examples for common flows
- make migration easier than staying custom

If the system feels slower or harder than copying local code, teams will bypass it.

## Testing a Design System

Design systems need more than unit tests.

Useful test layers:

- interaction tests for behavior
- accessibility checks for semantics and keyboard flows
- visual regression for stable appearance
- token/theme regression coverage

The goal is confidence that the shared foundation stays reliable across many consumers.

## Common Anti-Patterns

| Anti-Pattern | Why It Hurts |
|-------------|--------------|
| One component for every screen-specific pattern | Bloats the system |
| Prop explosion instead of composition | Hard to learn and maintain |
| Tokens that mirror hardcoded values with no semantics | Theming becomes painful |
| Accessibility left to consumers | Inconsistent and fragile UI |
| No ownership or review rules | The system drifts |
| Requiring custom overrides for normal use cases | API is probably under-designed |
| Treating docs as optional | Teams cannot adopt consistently |

## Decision Guide

Use this rough mapping:

- Need reusable visual values: tokens
- Need layout and text primitives: foundational components
- Need recurring interaction pattern: composed component
- Need feature-specific business UI: keep it in the product, not the core system
- Need multiple brands or modes: semantic tokens plus theming
- Need consistency across teams: docs plus governance, not just code

## Rules of Thumb

- Start with tokens and primitives, not a giant component backlog
- Make the default path accessible and visually correct
- Prefer composition over API sprawl
- Keep product-specific abstractions out of the core system
- Treat documentation and governance as part of the system itself
- Optimize for adoption, not only architecture purity

## Related Skills

- `accessibility-interaction-patterns` for system-level accessible defaults
- `developer-experience` for tooling and workflow ergonomics
- `performance-patterns` for styling/runtime tradeoffs and shipped UI cost
- `testing-strategies-patterns` for regression and confidence patterns

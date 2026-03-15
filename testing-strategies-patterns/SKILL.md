---
name: testing-strategies-patterns
description: Explains testing strategy for modern web applications including unit, integration, component, end-to-end, contract, accessibility, visual, and performance testing. Use when deciding what to test at each layer, choosing modern testing tools, designing CI quality gates, or improving confidence without creating a slow and brittle test suite.
---

# Testing Strategies Patterns

## Overview

Testing strategy is about deciding where confidence should come from. Strong teams do not try to prove everything with one kind of test. They distribute risk across fast local checks, realistic integration tests, and a smaller number of end-to-end flows.

Modern tooling has improved this significantly. Tools like Vitest, Playwright, Testing Library, Cypress, Storybook, MSW, and contract-testing frameworks make it practical to build layered test suites that are both fast and credible.

## The Goal

The goal is not "maximum coverage" in the abstract. The goal is:

- catch regressions early
- keep feedback loops fast
- test behavior at the right layer
- avoid brittle tests that slow delivery

## Test Layers

| Layer | What It Verifies | Speed | Common Tools |
|------|-------------------|-------|--------------|
| Static analysis | Types, lint, obvious mistakes | Fastest | TypeScript, ESLint, Biome |
| Unit | Pure logic in isolation | Very fast | Vitest, Jest |
| Component | UI behavior with realistic rendering | Fast | Testing Library, Vitest, Cypress Component |
| Integration | Multiple modules or network boundaries together | Medium | Vitest, Testing Library, MSW, Supertest |
| End-to-end | Real user flows in a running app | Slowest | Playwright, Cypress |
| Contract | API compatibility between systems | Medium | Pact, schema validation |
| Accessibility | Keyboard, semantics, announcements | Fast to medium | axe, Playwright, Testing Library |
| Visual | Layout and rendering regressions | Medium | Storybook, Chromatic, Playwright screenshots |
| Performance | Real-user or lab performance budgets | Medium | Lighthouse CI, Web Vitals, Playwright traces |

Do not treat these as competing options. They are complementary.

## Testing Pyramid, But Updated

The classic pyramid still matters:

```text
          Small number of E2E tests
        More integration/component tests
     Large number of unit and static checks
```

But modern frontend teams often use a modified shape:

- many unit tests for domain logic
- many component and integration tests for UI behavior
- a thin layer of E2E for critical journeys

This is more useful than a simplistic "only unit tests" or "only E2E" approach.

## What to Test Where

### Unit Tests

Use unit tests for:
- data transformations
- validation logic
- reducers and state machines
- utility functions
- pricing, permissions, or other business rules

```javascript
import { describe, it, expect } from 'vitest';
import { calculateTotal } from './cart';

describe('calculateTotal', () => {
  it('applies discount before tax', () => {
    expect(calculateTotal({ subtotal: 100, discount: 10, taxRate: 0.1 })).toBe(99);
  });
});
```

Avoid unit-testing framework internals or trivial wrappers with no meaningful behavior.

### Component Tests

Use component tests for:
- rendering states
- button and form interactions
- loading, empty, and error views
- keyboard behavior and basic accessibility

```jsx
render(<LoginForm />);
await user.type(screen.getByLabelText(/email/i), 'user@example.com');
await user.click(screen.getByRole('button', { name: /sign in/i }));
expect(await screen.findByText(/welcome/i)).toBeInTheDocument();
```

Prefer assertions that reflect user-observable behavior rather than internal implementation details.

### Integration Tests

Use integration tests when behavior spans several parts:
- component plus network request
- route plus loader/action
- API handler plus database abstraction
- cache invalidation and refetch behavior

Good integration tests catch the gaps between units, which is where many bugs actually live.

### End-to-End Tests

Use E2E for:
- sign up and login
- checkout or payment initiation
- content creation and editing
- major navigation flows
- role-based access journeys

E2E tests should prove that the system works as assembled. They should not be your only safety net.

## Modern Tooling Map

### Vitest

Good fit for:
- fast unit tests
- integration tests in Vite-based apps
- mocking, fake timers, and watch mode
- shared TypeScript-first frontend stacks

Why teams like it:
- fast startup
- good ESM support
- strong fit with modern frontend toolchains

### Jest

Still valid when:
- the repo already uses it
- the ecosystem around the project depends on it
- migration cost outweighs the benefit

Choose based on repo fit, not trend-chasing.

### Testing Library

Use it to test UI the way users experience it:
- query by role, label, text
- focus on visible behavior
- avoid shallow rendering and implementation-coupled assertions

### Playwright

Strong choice for modern E2E and browser automation:
- cross-browser coverage
- reliable auto-waiting
- tracing, screenshots, video
- component testing in some setups
- strong CI ergonomics

For many modern teams, Playwright is the default E2E recommendation.

### Cypress

Still useful for:
- mature existing suites
- interactive debugging
- teams already invested in its workflow

Choose it if it fits the stack and team habits better than Playwright.

### MSW (Mock Service Worker)

Use for:
- mocking HTTP at the network layer
- keeping tests realistic without rewriting fetch internals
- making component and integration tests less brittle

This often produces better tests than mocking every API call manually.

### Storybook

Use for:
- documenting component states
- manual review of edge cases
- visual regression baselines
- component test harnesses

Storybook is not a replacement for tests, but it amplifies component confidence when paired with automated checks.

## Mocking Strategy

Mock carefully. Over-mocking creates fake confidence.

Good uses of mocks:
- isolate nondeterministic services
- simulate network failures
- make expensive dependencies predictable

Bad uses of mocks:
- mocking the exact implementation you are trying to test
- replacing so much behavior that the test proves almost nothing

Prefer:
- real logic for units
- MSW or test servers for HTTP boundaries
- a real browser for critical user journeys

## Component vs E2E Boundaries

If a behavior can be proven reliably in a component or integration test, do it there first.

Reserve E2E for:
- assembled-system confidence
- browser-specific behavior
- navigation across pages
- integration with auth, storage, or backend infrastructure

This keeps E2E suites small, meaningful, and maintainable.

## Contract Testing

Contract tests reduce breakage between frontend and backend teams.

Use them when:
- teams ship independently
- API drift causes frequent regressions
- typed clients alone are not enough

Examples:
- validate responses against OpenAPI schemas
- verify consumer expectations with Pact-style tooling
- enforce required fields and status code behavior in CI

## Accessibility Testing

Accessibility should be part of the testing strategy, not a separate audit phase.

Use automated checks for:
- missing labels
- invalid roles
- color contrast hints
- landmark and form issues

Also include manual checks for:
- keyboard navigation
- focus management
- screen reader announcements
- modal and route-change behavior

Automated tooling catches some issues, not all of them.

## Visual Regression Testing

Use visual tests selectively:
- design system primitives
- marketing pages
- complex responsive layouts
- theming variants

Keep them stable by:
- controlling test data
- reducing animation noise
- testing only high-value surfaces

Visual diffs become noisy if applied indiscriminately.

## Performance and Reliability Checks

Modern test strategy often includes non-functional gates:

- Lighthouse CI for performance budgets
- bundle size checks
- Web Vitals monitoring
- flaky test detection
- retry policies with reporting, not silent masking

These checks elevate a repo because they guard user experience, not just correctness.

## CI Strategy

Split CI into layers:

### Fast checks on every change

- typecheck
- lint
- unit tests
- focused component or integration tests

### Heavier checks on merge or release

- E2E suite
- visual diffs
- accessibility sweeps
- performance budgets

This keeps feedback fast while still protecting critical paths.

## Flaky Test Prevention

Flakes usually come from timing, shared state, or unrealistic assertions.

Reduce flakes by:
- avoiding arbitrary sleeps
- waiting on observable conditions
- isolating test data
- making tests idempotent
- resetting storage, cookies, and mocks between cases
- keeping E2E tests independent from one another

If a suite is flaky, do not normalize reruns as the main solution. Fix the source of nondeterminism.

## Coverage Guidance

Coverage is a signal, not proof.

Useful:
- spotting untested high-risk areas
- enforcing a floor for core logic

Misleading:
- rewarding trivial tests
- pushing teams to test implementation details only to hit a number

Prefer risk-based coverage goals over vanity percentages.

## Decision Matrix

| Problem | Best Starting Layer |
|--------|----------------------|
| Pure calculation or utility logic | Unit |
| Form interaction or loading state | Component |
| Route loader, API call, cache behavior | Integration |
| Sign-in, checkout, core business journey | End-to-end |
| Backend/frontend schema drift | Contract |
| Focus, keyboard flow, labels | Accessibility |
| Layout or styling drift | Visual |
| Performance regressions | Performance budget tooling |

## Practical Starter Stack

For a modern web app, a strong default stack is:

- TypeScript + lint as baseline guards
- Vitest for unit and integration tests
- Testing Library for UI behavior
- MSW for network mocking
- Playwright for E2E
- axe-based checks for accessibility
- Lighthouse CI for performance budgets

This is not mandatory, but it is a credible modern baseline that will elevate most repos without overcomplicating them.

## Related Skills

- Read [developer-experience](../developer-experience/SKILL.md) for fast local feedback loops and tooling ergonomics.
- Read [accessibility-interaction-patterns](../accessibility-interaction-patterns/SKILL.md) for interaction details that tests should enforce.
- Read [data-fetching-patterns](../data-fetching-patterns/SKILL.md) and [caching-invalidation-patterns](../caching-invalidation-patterns/SKILL.md) for the async behaviors that commonly need integration coverage.
- Read [build-pipelines-bundling](../build-pipelines-bundling/SKILL.md) for bundle and performance implications that should feed into CI gates.

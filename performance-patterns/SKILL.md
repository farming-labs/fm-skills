---
name: performance-patterns
description: Explains web performance patterns including Core Web Vitals, network and JavaScript cost, rendering and hydration work, lazy loading, caching, and performance budgets. Use when diagnosing slow pages, improving user-perceived speed, or deciding which optimization strategy matters most.
---

# Performance Patterns

## Overview

Web performance is about how quickly users can:

- see useful content
- interact without lag
- move through the application without jank

Performance is not one problem. It is usually a combination of:

- too much network work
- too much JavaScript
- too much rendering work
- too much data fetching latency
- too much layout instability

The right optimization depends on which of those is actually dominating.

## The Main Performance Buckets

| Bucket | Typical Problem | Typical Fix |
|--------|------------------|-------------|
| Network | Large assets, slow server, too many requests | Compress, cache, preload, reduce bytes |
| JavaScript | Big bundles, expensive parsing, long tasks | Ship less JS, split code, defer work |
| Rendering | Large DOM, expensive layout/paint | Simplify UI work, reduce rerenders |
| Hydration | Too much client work before interactivity | Hydrate less, hydrate later, use islands |
| Data | Waterfalls, slow APIs, duplicate fetching | Parallelize, cache, prefetch, stream |

## Core Web Vitals

Core Web Vitals are the most useful high-level user-centered performance signals.

| Metric | Measures | Good |
|--------|----------|------|
| **LCP** | How quickly main content becomes visible | <= 2.5s |
| **INP** | How quickly the page responds to interaction | <= 200ms |
| **CLS** | How stable the layout is while loading | <= 0.1 |

### Interpreting Them

- Bad **LCP** usually means slow server response, large render-blocking assets, or slow hero image/content delivery
- Bad **INP** usually means too much JavaScript or long main-thread work
- Bad **CLS** usually means space is not reserved for content that appears later

Do not optimize blindly. Start with the metric that is actually failing.

## A Practical Performance Model

Think in this order:

```text
Can the browser get the bytes quickly?
Can it understand and execute them cheaply?
Can it render useful content early?
Can the user interact without waiting?
Can the UI stay stable while it updates?
```

This usually maps to:

```text
TTFB -> LCP -> Hydration / JS cost -> INP -> CLS
```

## Performance Budgets

Budgets prevent performance from degrading gradually over time.

Common budgets:

- JavaScript bytes per route
- image weight for above-the-fold content
- number of blocking requests
- maximum LCP
- maximum INP

Example budget mindset:

```text
Home page:
- Initial JS under 180 KB compressed
- Hero image under 120 KB
- LCP under 2.5s on mid-tier mobile
```

Without budgets, performance work often becomes reactive and late.

## Architecture Impacts Performance

| Pattern | Strength | Risk |
|---------|----------|------|
| CSR | Simple hosting, rich interactivity | Slow first render, high JS cost |
| SSR | Faster first content, better SEO | Server latency, hydration cost |
| SSG | Fastest repeatable loading for static content | Staleness, rebuild tradeoffs |
| Streaming | Early shell + progressive content | More complexity in boundaries |
| Islands / partial hydration | Less JS on page | More architecture complexity |

Architecture does not guarantee performance, but it changes where the performance risks move.

## Pattern 1: Ship Less JavaScript

Large JS bundles hurt:

- download time
- parse/compile time
- hydration cost
- interaction latency

### Common Approaches

- Remove unused dependencies
- Prefer smaller libraries or built-in APIs
- Code split by route and heavy feature
- Avoid sending server-only logic to the client
- Render more on the server when appropriate

```javascript
// Load expensive feature only when needed
const Chart = lazy(() => import('./Chart'));
```

If you can delete JavaScript entirely, that is often better than optimizing it.

## Pattern 2: Prioritize the Critical Path

The browser should receive the most important resources first.

Examples:

- preload the hero image
- inline or prioritize critical CSS
- defer non-essential scripts
- avoid blocking fonts or third-party scripts above the fold

```html
<link rel="preload" href="/hero.webp" as="image">
<script defer src="/app.js"></script>
```

Do not preload everything. Preloading too much just moves congestion around.

## Pattern 3: Lazy Load Non-Critical Work

Load expensive code and assets when users actually need them.

Good candidates:

- admin pages
- charts
- WYSIWYG editors
- below-the-fold media
- modal-only features

```javascript
const Editor = lazy(() => import('./Editor'));
```

Lazy loading is useful only when the delayed resource is truly non-critical. If the user always needs it immediately, lazy loading may just add an extra wait.

## Pattern 4: Reduce Render and Layout Work

Performance problems are not always network problems.

UI can be slow because:

- too many nodes update
- layout recalculates repeatedly
- animations trigger layout/paint work
- the app does unnecessary rerenders

### Common Fixes

- Keep DOM size reasonable
- Virtualize long lists
- Avoid layout thrashing
- Animate transform/opacity instead of layout-heavy properties when possible
- Move expensive computations out of render paths

```javascript
// Bad: expensive work during every render
const sorted = bigArray.sort(compareFn);

// Better: derive once per real input change
const sorted = deriveSortedItems(bigArray);
```

## Pattern 5: Fetch Earlier, in Parallel, and Less Often

Data latency often dominates user experience.

Common mistakes:

- fetch-on-render waterfalls
- refetching identical data repeatedly
- waiting for one request before starting the next when not necessary

Better patterns:

- start independent requests in parallel
- prefetch likely-next routes
- cache stable data
- stream slower sections separately

```javascript
const [user, notifications] = await Promise.all([
  getUser(),
  getNotifications(),
]);
```

## Pattern 6: Optimize Images and Media

Images are often the largest payload on a page and are frequent LCP candidates.

### Rules of Thumb

- Use responsive image sizes
- Compress aggressively
- Prefer modern formats when supported
- Preload only the true LCP image
- Lazy load below-the-fold images
- Always reserve dimensions

```html
<img
  src="/hero.webp"
  width="1200"
  height="800"
  alt="Product hero"
>
```

Missing dimensions are a common CLS bug.

## Pattern 7: Manage Fonts Carefully

Fonts can block rendering or cause layout shifts.

Good practices:

- self-host when practical
- subset font files
- limit the number of weights and styles
- use `font-display: swap` or equivalent strategy
- preload only critical fonts

Too many custom fonts can erase performance gains elsewhere.

## Pattern 8: Control Third-Party Cost

Analytics, ads, chat widgets, A/B testing tools, and tag managers often create disproportionate performance damage.

Third-party scripts can hurt:

- TTFB
- main-thread availability
- INP
- privacy/compliance posture

Rules of thumb:

- load them later if possible
- remove the ones you do not use
- measure them separately
- do not assume vendor code is cheap because it is external

## Pattern 9: Defer Non-Urgent Work

Not all work must happen immediately after load or interaction.

Examples of deferrable work:

- analytics initialization
- non-critical hydration
- below-the-fold widgets
- expensive derived UI that is not immediately visible

Split urgent work from non-urgent work whenever the platform or framework supports it.

## Pattern 10: Stabilize the Layout

CLS is usually preventable.

Common causes:

- images without dimensions
- injected banners or ads
- font swaps changing text metrics
- content inserted above existing content

Typical fixes:

- reserve space
- keep placeholders the same size as final content
- avoid inserting UI above the reading position without care

## Pattern 11: Cache the Right Things

Caching improves performance only when freshness is handled correctly.

Useful cache layers:

- browser HTTP cache
- CDN cache
- application/server cache
- client-side query cache

Caching helps performance when:

- data is reused often
- assets are immutable or versioned
- invalidation is understood

Caching hurts when stale data creates correctness bugs or forces wasteful refetch loops.

## Pattern 12: Measure Before and After

Performance work without measurement turns into guesswork.

### Useful Signal Types

| Type | Use |
|------|-----|
| Lab tests | Repeatable debugging in a controlled environment |
| Field data | Real-user experience in production |
| Bundle analysis | Detect JS growth |
| Profiler traces | Diagnose rendering and main-thread work |

### Practical Workflow

1. Identify the failing metric or slow user flow
2. Find the dominant bottleneck
3. Apply the narrowest meaningful fix
4. Re-measure
5. Add a budget or guardrail so it does not regress

## Common Optimization Mistakes

| Mistake | Why It Fails |
|---------|--------------|
| Micro-optimizing components before checking bundle size | Big wins are often elsewhere |
| Lazy loading critical UI | User still waits, just later |
| Prefetching too aggressively | Wastes bandwidth and device resources |
| Caching everything | Creates correctness and invalidation issues |
| Measuring only on a fast laptop | Hides real user pain |
| Chasing one benchmark while hurting UX elsewhere | Metrics are proxies, not the product |

## Decision Guide

Use this rough mapping:

- Slow first paint: look at server response, critical assets, and render-blocking resources
- Slow first interaction: look at JS size, hydration, and long tasks
- Janky typing or clicking: inspect main-thread work and rerenders
- Layout jumps: reserve space and fix dynamic insertion behavior
- Slow route changes: prefetch, split bundles, cache data, reduce client work on navigation

## Rules of Thumb

- Delete work before optimizing work
- Optimize the critical path first
- Measure on realistic devices and networks
- Prefer broad architectural wins over tiny component tweaks
- Performance budgets are often more valuable than one-off performance heroics

## Related Skills

- `build-pipelines-bundling` for bundle size and chunking strategy
- `rendering-patterns` for SSR, SSG, CSR, and streaming tradeoffs
- `hydration-patterns` for hydration cost and islands architecture
- `data-fetching-patterns` for request waterfalls and loading strategies
- `caching-invalidation-patterns` for freshness and cache correctness

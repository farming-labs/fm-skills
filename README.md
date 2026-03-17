# Framework Skills

A collection of framework-agnostic AI skills for understanding & implementing modern web development fundamentals.

## Purpose

These skills help AI assistants understand:
- How web applications are architected (SPA, MPA, hybrid)
- Build pipelines, bundling, code splitting, and optimization
- Different rendering strategies and when to use each
- Performance optimization, Core Web Vitals, and budget-driven tradeoffs
- SEO implications of architectural decisions
- Universal JavaScript and cross-platform deployment (Nitro, H3, edge runtimes)
- Middleware patterns for request/response processing (auth, CORS, rate limiting)
- Observability across logs, metrics, traces, error reporting, and alerts
- Error handling, retries, fallbacks, boundaries, and graceful degradation
- Developer experience: HMR, dev servers, error overlays, and modern tooling
- Mutation UX patterns such as optimistic updates, rollback, and reconciliation
- Cache invalidation, freshness windows, and revalidation strategies
- Accessible interaction design including focus, keyboard flow, and announcements
- Testing strategy across unit, integration, E2E, accessibility, and performance layers
- Realtime delivery models such as polling, SSE, and WebSockets
- How meta-frameworks solve common problems
- Hydration, routing, and other core concepts

Rather than framework-specific instructions, these skills provide the **conceptual foundation** needed to make informed decisions and understand any meta-framework.

## Skills Overview

| Skill | Description |
|-------|-------------|
| [web-app-architectures](./web-app-architectures/SKILL.md) | SPA vs MPA fundamentals, hybrid architectures |
| [rendering-patterns](./rendering-patterns/SKILL.md) | CSR, SSR, SSG, ISR, Streaming |
| [performance-patterns](./performance-patterns/SKILL.md) | Core Web Vitals, bundle cost, rendering, hydration, caching |
| [seo-fundamentals](./seo-fundamentals/SKILL.md) | SEO for web apps, Core Web Vitals, crawling |
| [hydration-patterns](./hydration-patterns/SKILL.md) | Hydration, islands architecture, resumability |
| [meta-frameworks-overview](./meta-frameworks-overview/SKILL.md) | Next.js, Nuxt, SvelteKit, Astro, Remix, Qwik |
| [routing-patterns](./routing-patterns/SKILL.md) | Client vs server routing, file-based routing |
| [accessibility-interaction-patterns](./accessibility-interaction-patterns/SKILL.md) | Keyboard access, focus management, ARIA, async feedback |
| [testing-strategies-patterns](./testing-strategies-patterns/SKILL.md) | Unit, integration, E2E, accessibility, visual, and CI testing |
| [error-handling-patterns](./error-handling-patterns/SKILL.md) | Error boundaries, retries, user recovery, fallbacks, observability hooks |
| [observability-patterns](./observability-patterns/SKILL.md) | Logs, metrics, traces, client errors, SLOs, and alerts |
| [state-management-patterns](./state-management-patterns/SKILL.md) | Client state, server state, URL state, caching |
| [data-fetching-patterns](./data-fetching-patterns/SKILL.md) | Fetch patterns, caching, loading states |
| [caching-invalidation-patterns](./caching-invalidation-patterns/SKILL.md) | Cache keys, stale data, revalidation, mutation invalidation |
| [optimistic-ui-patterns](./optimistic-ui-patterns/SKILL.md) | Optimistic mutations, rollback, temp IDs, reconciliation |
| [realtime-patterns](./realtime-patterns/SKILL.md) | Polling, SSE, WebSockets, presence, reconnection, reconciliation |
| [build-pipelines-bundling](./build-pipelines-bundling/SKILL.md) | Bundling, code splitting, tree shaking, build optimization |
| [universal-javascript-runtimes](./universal-javascript-runtimes/SKILL.md) | Nitro, H3, unenv, web standards, cross-platform deployment |
| [middleware-patterns](./middleware-patterns/SKILL.md) | Server/edge middleware, request pipelines, auth, CORS, rate limiting |
| [developer-experience](./developer-experience/SKILL.md) | HMR, dev servers, Vite architecture, error overlays, fast refresh |

## Learning Path

For best understanding, read the skills in this order:

```
1. web-app-architectures        (Foundation: SPA vs MPA)
        ↓
2. build-pipelines-bundling     (How code is transformed and bundled)
        ↓
3. developer-experience         (HMR, dev servers, Vite, fast tooling)
        ↓
4. rendering-patterns           (When/where HTML is generated)
        ↓
5. hydration-patterns           (How static becomes interactive)
        ↓
6. performance-patterns         (LCP, INP, CLS, and critical-path tradeoffs)
        ↓
7. routing-patterns             (How navigation works)
        ↓
8. accessibility-interaction-patterns (Keyboard, focus, announcements)
        ↓
9. testing-strategies-patterns  (How to build confidence and quality gates)
        ↓
10. error-handling-patterns     (Retries, recovery UX, boundaries, and fallbacks)
        ↓
11. observability-patterns      (Logs, metrics, traces, and production diagnosis)
        ↓
12. data-fetching-patterns      (How to load data)
        ↓
13. caching-invalidation-patterns (Freshness, staleness, revalidation)
        ↓
14. state-management-patterns   (Where to store data)
        ↓
15. optimistic-ui-patterns      (Mutations, rollback, reconciliation)
        ↓
16. realtime-patterns           (Live delivery, sync, presence)
        ↓
17. seo-fundamentals            (Search engine optimization)
        ↓
18. universal-javascript-runtimes (Deploy anywhere: edge, serverless, Node)
        ↓
19. middleware-patterns         (Request/response pipelines, auth, CORS)
        ↓
20. meta-frameworks-overview    (How frameworks implement all of the above)
```

## Installation

### Via skills.sh

You can also install using [skills.sh](https://skills.sh/farming-labs/fm-skills):

```bash
npx skills add farming-labs/fm-skills
```


### For Cursor IDE

Copy to your personal skills directory:

```bash
# Personal skills (available in all projects)
cp -r fm-skills ~/.cursor/skills/

# Or symlink
ln -s /path/to/fm-skills ~/.cursor/skills/fm-skills
```

### For Project-Specific Use

Copy to your project:

```bash
cp -r fm-skills .cursor/skills/
```

## Usage

Once installed, Cursor will automatically suggest these skills when relevant topics are discussed. For example:

- Asking "Should I use SSR or SSG?" will trigger rendering-patterns
- Asking "How do I improve LCP, INP, or bundle performance?" will trigger performance-patterns
- Asking "How do SPAs handle routing?" will trigger web-app-architectures and routing-patterns
- Asking "How do I make this dropdown, modal, or form interaction accessible?" will trigger accessibility-interaction-patterns
- Asking "What should I test with Vitest, Playwright, or Testing Library?" will trigger testing-strategies-patterns
- Asking "How should this app handle load failures, retries, or fallback UI?" will trigger error-handling-patterns
- Asking "How should I instrument logs, metrics, traces, or alerts for this app?" will trigger observability-patterns
- Asking "Should this update use polling, SSE, or WebSockets?" will trigger realtime-patterns
- Asking "How do I improve SEO for my React app?" will trigger seo-fundamentals
- Asking "How should I invalidate cache after mutations or avoid stale query data?" will trigger caching-invalidation-patterns
- Asking "How should I build optimistic updates for likes, comments, or reordering?" will trigger optimistic-ui-patterns

## Skill Structure

Each skill follows the standard format:

```
skill-name/
└── SKILL.md          # Main content with YAML frontmatter
```

## Contributing

Contributions welcome! Each skill should be:

1. **Framework-agnostic**: Explain concepts, not framework-specific APIs
2. **Concise**: Under 500 lines, link to external resources for depth
3. **Practical**: Include decision matrices and code examples
4. **Connected**: Reference related skills for deeper understanding

## License

MIT License - Feel free to use, modify, and distribute.

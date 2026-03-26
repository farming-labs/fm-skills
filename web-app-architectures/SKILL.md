---
name: web-app-architectures
description: "Guide for choosing between SPA, MPA, and hybrid web application architectures. Use when deciding app architecture, comparing SPA vs MPA tradeoffs, planning a new web application, evaluating rendering strategies, or choosing between server-rendered and client-rendered approaches."
---

# Web Application Architectures

## Architecture Quick Reference

| Pattern | Navigation | Initial Load | SEO | State | Best For |
|---------|-----------|-------------|-----|-------|----------|
| **MPA** | Full page reload | Fast (HTML only) | Excellent | Lost on nav | Content sites, blogs, docs |
| **SPA** | Instant (JS routing) | Slow (JS bundle) | Requires SSR | Persistent | Dashboards, authenticated apps |
| **Hybrid SSR** | SPA after first load | Fast (server HTML) | Excellent | Persistent | E-commerce, marketing + app |
| **Islands** | Full page reload | Fast (minimal JS) | Excellent | Per-island | Content sites with interactive widgets |

---

## Multi Page Application (MPA)

**Each navigation = full server round-trip.**

```
User clicks link → Browser requests HTML → Server renders full page → Browser loads entire page
```

### MPA Characteristics

- Each page is a complete HTML document — excellent for SEO and crawlers
- State resets on every navigation (persist via cookies, localStorage, URL params, server sessions)
- Progressive enhancement: works without JavaScript
- Lower client-side complexity, higher server rendering cost

### When to Choose MPA

- Content is the product (blogs, news, documentation)
- SEO is non-negotiable
- Users may have slow connections or JS disabled
- Team strength is server-side languages
- Simple interactivity requirements

---

## Single Page Application (SPA)

**One HTML shell + JS bundle handles all routing and rendering.**

```
Initial:    Browser loads shell HTML + JS bundle
Navigate:   JS intercepts clicks → Updates URL via History API → Renders new view
Data:       Fetch API calls return JSON only
```

### SPA Characteristics

- Navigation is 10-100x faster (no HTTP round-trip, just DOM updates)
- Large initial JS bundle increases Time to Interactive (TTI)
- State persists in memory across navigation — risk of memory leaks
- Back button and deep linking require explicit handling
- SEO requires SSR or prerendering workarounds

### SPA Tradeoffs

| Advantage | Disadvantage |
|-----------|-------------|
| App-like UX with smooth transitions | Large initial JS bundle (200KB-2MB+) |
| Persistent UI state (players, forms) | SEO requires workarounds |
| Reduced server load after initial | Memory management complexity |
| Offline support via Service Workers | Blank page if JS fails |

### Core Browser APIs Enabling SPAs

```javascript
// Client-side routing via History API
history.pushState(state, '', '/new-route');  // Change URL without reload
history.replaceState(state, '', '/new-route');  // Replace current entry
window.addEventListener('popstate', (e) => renderRoute(location.pathname));

// Data fetching — JSON instead of HTML
const data = await fetch('/api/products/123').then(r => r.json());
renderProduct(data);
```

---

## Hybrid Architectures

### SPA with Server-Side Rendering (SSR)

Server pre-renders HTML for initial load, then hydrates to full SPA.

```
First request:  Server renders complete HTML → Browser hydrates to SPA
Subsequent:     Client-side SPA navigation (no server round-trip)
```

**Frameworks:** Next.js, Nuxt, SvelteKit, Remix

### MPA with Islands Architecture

Server renders full HTML; only interactive "islands" get JavaScript hydration.

```
Server renders full HTML → JS hydrates only marked interactive components
Rest of page: zero JavaScript
```

**Frameworks:** Astro, Fresh (Deno)

**Island hydration strategies:**

| Strategy | Trigger | Use Case |
|----------|---------|----------|
| `client:load` | Immediately on page load | Critical interactive elements |
| `client:idle` | When browser is idle | Non-critical enhancements |
| `client:visible` | When element enters viewport | Below-fold components |

### Streaming / Progressive Rendering

Server streams HTML chunks as data becomes available — browser renders progressively.

```
Server starts sending HTML → Browser renders as chunks arrive → JS hydrates incrementally
```

---

## Architecture Decision Matrix

Use this to select the right architecture based on project requirements:

| Requirement | Recommended | Why |
|-------------|-------------|-----|
| Content site, SEO critical | MPA or Hybrid SSR | Complete HTML for crawlers |
| Dashboard, authenticated app | SPA | SEO irrelevant, rich interactivity needed |
| E-commerce (SEO + interactivity) | Hybrid SSR | Product pages indexed, cart is interactive |
| Minimal JS, fast initial load | MPA with Islands | Ship JS only where needed |
| Rich interactions, app-like UX | SPA or Hybrid | Persistent state, smooth transitions |
| Limited team, simple stack | MPA | Lowest complexity |
| Offline support needed | SPA + Service Workers | Client-side caching and routing |
| Mixed content + app sections | Islands or Hybrid | Different strategies per page section |

### Decision Workflow

```
1. Is SEO critical for most pages?
   YES → Is rich interactivity also needed?
         YES → Hybrid SSR (Next.js, Nuxt, SvelteKit)
         NO  → MPA (or Islands if some interactivity needed)
   NO  → Is it an app-like experience (dashboard, tool)?
         YES → SPA
         NO  → MPA with progressive enhancement

2. Validate choice against constraints:
   - Team expertise (server-side → MPA, frontend-heavy → SPA)
   - Performance budget (slow connections → MPA/Islands)
   - Offline requirements (needed → SPA + Service Workers)
   - Scale (high traffic → SPA/Hybrid with CDN-cached static assets)
```

---

## Key Implementation Patterns

### Code Splitting (SPA/Hybrid)

Break the JS bundle into route-based chunks loaded on demand:

```javascript
// Route-based code splitting
const routes = {
  '/':          () => import('./pages/Home.js'),
  '/dashboard': () => import('./pages/Dashboard.js'),
  '/settings':  () => import('./pages/Settings.js'),
};

// Only loads Dashboard.js when user navigates to /dashboard
const page = await routes[currentPath]();
```

### File-Based Routing Convention

Map filesystem structure to routes (used by Next.js, Nuxt, SvelteKit, Astro):

```
src/pages/
├── index.tsx          → /
├── about.tsx          → /about
├── blog/
│   ├── index.tsx      → /blog
│   └── [slug].tsx     → /blog/:slug (dynamic)
└── [...catchall].tsx  → /* (catch-all)
```

**Route priority:** static segments > dynamic params > catch-all.

### SPA State Persistence Across Navigation

```
SPA:  Component state survives navigation in memory
      Risk: memory leaks in long-running sessions
      Mitigation: cleanup on unmount, WeakMap/WeakRef for caches

MPA:  State must be explicitly serialized
      Options: URL params, cookies, localStorage, server sessions
      Benefit: no memory leaks (page destroyed on each nav)
```

### Scroll Restoration

```javascript
// Save scroll position before navigation
const scrollPositions = new Map();

router.beforeNavigate((from) => {
  scrollPositions.set(from.path, { x: scrollX, y: scrollY });
});

router.afterNavigate((to) => {
  const pos = scrollPositions.get(to.path);
  pos ? window.scrollTo(pos.x, pos.y) : window.scrollTo(0, 0);
});
```

---

## SEO Implications by Architecture

| Architecture | Crawler Experience | Action Required |
|-------------|-------------------|-----------------|
| MPA | Complete HTML on first request | None — works by default |
| SPA | Empty `<div id="root">` | Add SSR, prerendering, or dynamic rendering |
| Hybrid SSR | Complete HTML + hydration | None — best of both worlds |
| Islands | Complete HTML + partial JS | None — static HTML by default |

**Why SPAs struggle with SEO:** Google's crawler has two phases — fast HTML indexing (Phase 1) and expensive JS rendering (Phase 2). SPA content only appears in Phase 2, which is delayed and not guaranteed for all pages.

---

## Server Load Comparison

| Aspect | MPA | SPA | Hybrid |
|--------|-----|-----|--------|
| Per-request CPU | High (template rendering) | Low (JSON serialization) | Medium (SSR on first hit) |
| Bandwidth | High (full HTML each nav) | Low (JSON payloads) | Medium |
| CDN cacheability | Page-level | Static assets + API | Static shell + API |
| Scaling strategy | Scale servers | CDN for static, scale API | CDN + scale SSR/API |

---

## Navigation Performance Comparison

```
MPA per-navigation cost:
  DNS + TCP + TLS + Server + Transfer + Parse + Render = 400-1600ms

SPA per-navigation cost:
  JS route match + DOM update = 10-70ms (+ optional data fetch 100-500ms)

Hybrid first load:
  Server-rendered HTML (fast like MPA) → Hydrate → SPA navigation thereafter
```

---

## Validation Checklist

When recommending an architecture, verify:

- [ ] SEO requirements mapped to architecture capabilities
- [ ] Initial load performance budget considered (TTI targets)
- [ ] Team expertise aligns with chosen stack complexity
- [ ] Offline/PWA requirements addressed
- [ ] State management strategy matches architecture (client vs server)
- [ ] Deployment infrastructure supports chosen pattern (CDN, SSR server, edge)
- [ ] Accessibility: works without JavaScript if progressive enhancement required

---

## Related Skills

- [rendering-patterns](../rendering-patterns/SKILL.md) — SSR, SSG, CSR, ISR deep dive
- [seo-fundamentals](../seo-fundamentals/SKILL.md) — SEO strategies and implementation
- [hydration-patterns](../hydration-patterns/SKILL.md) — Hydration concepts and optimization
- [meta-frameworks-overview](../meta-frameworks-overview/SKILL.md) — Framework comparison and selection

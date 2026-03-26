---
name: hydration-patterns
description: "Guide for choosing and implementing hydration strategies — full hydration, progressive hydration, partial hydration, islands architecture, and resumability. Use when making server-rendered HTML interactive, reducing TTI, optimizing JS bundle size, or deciding between islands and full hydration for a project."
---

# Hydration Patterns

## Strategy Decision Matrix

Pick the hydration approach based on your project type:

| Scenario | Strategy | Why |
|----------|----------|-----|
| SPA / dashboard (most content interactive) | Full hydration | Simple, full interactivity everywhere |
| Content site with some widgets | Islands (Astro, Fresh) | Ship zero JS for static parts |
| E-commerce product page | Partial / progressive | Prioritize buy-button TTI, defer reviews |
| Marketing landing page | Islands or static (no hydration) | Minimal JS, fast paint |
| Highly interactive app | Full hydration + code splitting | Balance interactivity with bundle size |
| Performance-critical, mixed interactivity | Resumability (Qwik) | Near-zero TTI, JS on demand |

## Full Hydration

Hydrate the entire page in one pass. Use for apps where most components are interactive.

```javascript
// React
import { hydrateRoot } from 'react-dom/client';
hydrateRoot(document.getElementById('root'), <App />);
```

**Trade-offs:** Simple mental model. Large JS bundles and slow TTI on content-heavy pages because ALL component code ships, even for static content.

## Progressive Hydration

Hydrate components in priority order — critical UI first, rest deferred.

```javascript
// 1. Hydrate above-the-fold immediately
hydrateRoot(headerContainer, <Header />);

// 2. Defer non-critical with startTransition
import { startTransition } from 'react';
startTransition(() => {
  hydrateRoot(commentsContainer, <Comments />);
});

// 3. Hydrate on visibility
function LazyHydrate({ children }) {
  const ref = useRef(null);
  const [hydrated, setHydrated] = useState(false);

  useEffect(() => {
    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) {
        setHydrated(true);
        observer.disconnect();
      }
    }, { rootMargin: '200px' });
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);

  if (!hydrated) return <div ref={ref} dangerouslySetInnerHTML={{ __html: '' }} />;
  return children;
}
```

**When to use:** Pages where above-the-fold content matters most and below-fold can load lazily.

## Selective Hydration (React 18+)

Components wrapped in `<Suspense>` hydrate independently and out of order. React prioritizes subtrees the user interacts with.

```jsx
<Layout>
  <Header />
  <Suspense fallback={<Spinner />}>
    <Comments />  {/* Hydrates when data ready, independent of Sidebar */}
  </Suspense>
  <Suspense fallback={<Spinner />}>
    <Sidebar />   {/* Can hydrate before or after Comments */}
  </Suspense>
</Layout>
```

**Key behavior:** If a user clicks inside a pending Suspense boundary, React prioritizes hydrating that subtree first.

## Partial Hydration / Islands Architecture

Ship zero JS for static components. Only interactive "islands" get JavaScript.

### Identifying Islands

Audit components for interactivity signals:

| Signal | Needs JS? | Action |
|--------|-----------|--------|
| `useState`, `useSignal`, `createSignal` | Yes | Make it an island |
| `onClick`, `onSubmit`, event handlers | Yes | Make it an island |
| `useEffect`, `onMount` | Yes | Make it an island |
| Pure props-to-markup, no state/events | No | Keep static, ship no JS |

### Astro Islands Example

```astro
---
import Header from './Header.astro';    // Static — zero JS
import Counter from './Counter.jsx';    // Interactive — ships JS
import Comments from './Comments.jsx';  // Interactive — ships JS
---

<Header />                       <!-- No JS shipped -->
<Counter client:load />          <!-- Hydrate immediately on page load -->
<Counter client:visible />       <!-- Hydrate when scrolled into viewport -->
<Counter client:idle />          <!-- Hydrate during browser idle time -->
<Comments client:media="(min-width: 768px)" />  <!-- Hydrate on media match -->
```

**Result:** Only Counter and Comments JS ships. Header compiles to pure HTML.

### Frameworks with Islands Support

- **Astro** — first-class islands, multi-framework
- **Fresh (Deno)** — islands by default
- **Iles** — islands for Vite
- **Qwik** — resumability (related approach, see below)

## Resumability (Qwik)

Skip hydration entirely. Server serializes application state and handler references into HTML. Client resumes without re-executing component code.

```javascript
// Qwik component
function Counter() {
  const count = useSignal(0);
  return <button onClick$={() => count.value++}>Count: {count.value}</button>;
}

// Server output — handler reference serialized, not the function itself:
// <button on:click="counter_onclick.js#s0" q:obj="0">Count: 0</button>
// <script type="qwik/json">{"signals":{"0":0}}</script>

// Client: interactive immediately. Handler JS loads only on first click.
```

**Key advantage:** TTI equals page load time. No hydration delay regardless of page complexity.

## Hydration Anti-Patterns and Fixes

### 1. Hydration Mismatch

Server and client render different output, causing errors or flicker.

```javascript
// BAD: Date differs between server and client
function Greeting() {
  return <p>Hello at {new Date().toLocaleTimeString()}</p>;
}

// GOOD: Defer client-only values to useEffect
function Greeting() {
  const [time, setTime] = useState(null);
  useEffect(() => setTime(new Date().toLocaleTimeString()), []);
  return <p>Hello{time ? ` at ${time}` : ''}</p>;
}
```

**Rule:** Anything that differs between server and client (time, random values, browser APIs) must be set in `useEffect`, not during render.

### 2. Blocking Hydration on Data Fetches

```javascript
// BAD: Component fetches in useEffect before becoming interactive
useEffect(() => { fetchData().then(setData); }, []);

// GOOD: Use server-provided data via SSR props or streaming
// Pass initial data from getServerSideProps / loader / server component
```

### 3. Oversized Hydration Bundles

```javascript
// BAD: Static import of heavy library
import { FullCalendar } from 'massive-calendar-lib';

// GOOD: Dynamic import — loads only when needed
const Calendar = lazy(() => import('./Calendar'));
```

## Measuring Hydration Performance

```javascript
// Basic timing
const start = performance.now();
hydrateRoot(container, <App />);
console.log(`Hydration: ${performance.now() - start}ms`);

// React Profiler for component-level timing
<Profiler id="App" onRender={(id, phase, duration) => {
  if (phase === 'mount') console.log(`${id} hydration: ${duration}ms`);
}}>
  <App />
</Profiler>
```

**Key metrics to watch:**
- **FCP to TTI gap** — the "hydration gap" where the page looks ready but is not interactive
- **Total Blocking Time (TBT)** — time the main thread is blocked during hydration
- **JS bundle size** — directly impacts download + parse time before hydration starts

## Validation Checklist

Before shipping a hydration strategy:

- [ ] Confirm no hydration mismatches in dev mode console
- [ ] Measure TTI on target devices (not just desktop)
- [ ] Verify interactive elements work immediately after hydration (click handlers, forms)
- [ ] Check bundle size — are you shipping JS for static-only components?
- [ ] For islands: confirm static components produce zero client JS
- [ ] For progressive: verify priority order matches user interaction patterns
- [ ] Lighthouse: compare TBT and TTI before/after strategy change

## Related Skills

- [rendering-patterns](../rendering-patterns/SKILL.md) — SSR, SSG, ISR context for when HTML is generated
- [web-app-architectures](../web-app-architectures/SKILL.md) — SPA vs MPA trade-offs that inform hydration choice
- [meta-frameworks-overview](../meta-frameworks-overview/SKILL.md) — how Next.js, Astro, Qwik handle hydration
- [performance-patterns](../performance-patterns/SKILL.md) — broader performance optimization including bundle analysis

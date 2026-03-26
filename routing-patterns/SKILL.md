---
name: routing-patterns
description: "Implement client-side routing, server-side routing, file-based routing, and navigation guards in web apps. Use when setting up routes, configuring URL-to-component mapping, adding route protection, handling dynamic parameters, or choosing between SPA and MPA navigation strategies."
---

# Routing Patterns

## Routing Type Decision Matrix

| Need | Use | Why |
|------|-----|-----|
| SEO-critical content, no JS required | Server-side routing | Full HTML per URL, crawlable |
| Rich interactivity, fast transitions | Client-side routing (SPA) | No full reload, state preserved |
| Both SEO + interactivity | Hybrid (meta-framework) | SSR first load, client nav after |
| Convention-based route config | File-based routing | File path = URL, zero config |

## Server-Side Routing

```
Request: GET /products/123

Route table:
  /            → HomeController
  /products/:id → ProductController  ← matches

ProductController → fetch product 123 → render HTML → return response
```

| Pros | Cons |
|------|------|
| SEO-friendly (full HTML) | Full page reload on navigate |
| Works without JavaScript | State lost between pages |
| Simple mental model | Server handles every route |

## Client-Side Routing

### Core Mechanism

```javascript
// 1. Intercept link clicks
document.addEventListener('click', (e) => {
  const link = e.target.closest('a');
  if (link && link.href.startsWith(window.location.origin)) {
    e.preventDefault();
    history.pushState({}, '', link.pathname);
    renderRoute(link.pathname);
  }
});

// 2. Handle back/forward
window.addEventListener('popstate', () => {
  renderRoute(window.location.pathname);
});
```

### History API Essentials

```javascript
history.pushState(state, '', '/about');    // Navigate (adds to stack)
history.replaceState(state, '', '/home');   // Replace current entry
// pushState does NOT fire popstate — only back/forward does
```

## File-Based Routing

### Convention Map

```
File System                          URL
pages/index.tsx                  →   /
pages/about.tsx                  →   /about
pages/blog/[slug].tsx            →   /blog/:slug
pages/docs/[...path].tsx         →   /docs/* (catch-all)
pages/[[locale]]/index.tsx       →   / or /:locale (optional)
```

### Framework Variants

**Next.js (App Router):**
```
app/
├── page.tsx                    → /
├── about/page.tsx              → /about
├── blog/[slug]/page.tsx        → /blog/:slug
└── (marketing)/pricing/page.tsx → /pricing  (route group, no URL segment)
```

**SvelteKit:**
```
src/routes/
├── +page.svelte                → /
├── about/+page.svelte          → /about
└── blog/[slug]/+page.svelte    → /blog/:slug
```

**Remix:**
```
app/routes/
├── _index.tsx                  → /
├── about.tsx                   → /about
├── blog.$slug.tsx              → /blog/:slug
└── $.tsx                       → Catch-all (splat)
```

## Dynamic Route Parameters

```javascript
// Route: /products/[category]/[id]
// URL:   /products/shoes/nike-air-max
// → params = { category: 'shoes', id: 'nike-air-max' }

// Catch-all: /docs/[...path]
// URL:       /docs/getting-started/installation
// → params = { path: ['getting-started', 'installation'] }
```

## Nested Layouts

```
app/
├── layout.tsx              → Root layout (always renders)
├── (shop)/
│   ├── layout.tsx          → Shop layout (header, cart)
│   ├── products/page.tsx   → /products
│   └── cart/page.tsx       → /cart
└── (marketing)/
    ├── layout.tsx          → Marketing layout
    └── pricing/page.tsx    → /pricing

Renders as nested tree:
<RootLayout>
  <ShopLayout>        ← persists across /products ↔ /cart nav
    <ProductsPage />
  </ShopLayout>
</RootLayout>
```

### Advanced Layout Patterns

**Parallel routes** — render multiple views simultaneously:
```
app/
├── @sidebar/page.tsx     → Sidebar content
├── @main/page.tsx        → Main content
└── layout.tsx            → Composes both slots
```

**Intercepting routes** — modal overlays on internal nav:
```
app/
├── photos/[id]/page.tsx           → Full page (direct URL)
└── @modal/(.)photos/[id]/page.tsx → Modal (internal nav)
```

## Navigation Patterns

### Declarative

```jsx
// React Router / Next.js
<Link href="/about">About</Link>

// Vue Router
<router-link to="/about">About</router-link>
```

### Programmatic

```javascript
// Next.js:    router.push('/dashboard')
// React Router: navigate('/dashboard')
// Vue Router:   router.push('/dashboard')
// SvelteKit:    goto('/dashboard')

// Options:
router.replace('/login');           // No back-button entry
router.push('/search?q=term');      // With query params
router.push('/page', { scroll: true, shallow: true });
```

## Route Protection

### Authentication Guard (Middleware)

```javascript
// Next.js middleware
export function middleware(request) {
  const token = request.cookies.get('auth-token');
  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }
}

// Remix loader
export async function loader({ request }) {
  const user = await getUser(request);
  if (!user) throw redirect('/login');
  return { user };
}
```

### Authorization

```javascript
if (!user.roles.includes('admin')) redirect('/unauthorized');
if (!features.newDashboard) redirect('/legacy-dashboard');
```

## URL State Management

```javascript
// Read query params
const searchParams = useSearchParams();
const query = searchParams.get('q');

// Update query params
const params = new URLSearchParams(searchParams);
params.set('page', '2');
router.push(`/products?${params.toString()}`);
```

**URL state vs component state:**
- URL (shareable): page, filters, search query, tab selection
- Component (ephemeral): form input pre-submit, hover/focus, temp UI

## Performance Patterns

```jsx
// Prefetching — automatic on hover in Next.js
<Link href="/about" prefetch={true}>About</Link>
router.prefetch('/dashboard');  // Manual

// Code splitting by route
const Dashboard = lazy(() => import('./Dashboard'));
<Route path="/dashboard" element={
  <Suspense fallback={<Loading />}><Dashboard /></Suspense>
} />

// Navigation loading state
const navigation = useNavigation();
{navigation.state === 'loading' && <LoadingBar />}
```

## Route Matching Order

```javascript
// Static before dynamic — order matters
app.get('/users/new', newUserHandler);   // MUST be before :id
app.get('/users/:id', getUserHandler);   // Would match "new" otherwise

// Specificity scoring (frameworks auto-sort file-based routes):
// Static segment:  highest priority
// Dynamic [param]: medium
// Catch-all [...]: lowest
```

## Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Back button broken | Use `push()` not `replace()` for normal nav |
| No 404 handling | Add catch-all route (`[...not-found]/page.tsx`) |
| Hydration mismatch | Don't use `useSearchParams` in server components |
| Hash URLs for SEO content | Use `/products/123` not `/#/products/123` |
| Trailing slash inconsistency | Pick one convention, redirect the other |
| Encoded chars in params | Always `decodeURIComponent()` dynamic segments |

## Validation Checklist

1. Every route resolves to a component (no orphan paths)
2. 404 catch-all route exists
3. Auth guards cover all protected routes
4. Dynamic params are validated/sanitized in loaders
5. Prefetch configured for high-traffic navigation paths
6. Scroll restoration works on back/forward
7. Route order: static → dynamic → catch-all

## Related Skills

- See [web-app-architectures](../web-app-architectures/SKILL.md) for SPA vs MPA routing decisions
- See [meta-frameworks-overview](../meta-frameworks-overview/SKILL.md) for framework-specific routing details
- See [seo-fundamentals](../seo-fundamentals/SKILL.md) for SEO-friendly URL structure

---
name: caching-invalidation-patterns
description: Explains cache invalidation strategies in web applications including cache keys, freshness windows, mutation-driven invalidation, revalidation, and eviction tradeoffs. Use when designing server-state caching, deciding when data is stale, or fixing bugs caused by stale or inconsistent cached data.
---

# Caching Invalidation Patterns

## Overview

Caching makes applications fast by reusing previously fetched or computed data.

The problem is not storing data in a cache. The hard part is knowing:

- when cached data is still valid
- when it is stale
- what must be refreshed after a mutation
- how broad or narrow an invalidation should be

This is why cache invalidation is a core application design problem, not just a performance trick.

## What Invalidation Means

Invalidation means telling the system that cached data should no longer be treated as authoritative.

That can lead to one of several outcomes:

- Mark the cache entry stale and refetch later
- Refetch immediately
- Evict the entry entirely
- Replace the entry with known fresh data

Different systems use different APIs, but the underlying choices are the same.

## Core Questions

Before choosing an invalidation strategy, answer these:

1. What data is cached?
2. Who owns the source of truth?
3. How quickly can that data change?
4. What mutations can make it stale?
5. Is stale data acceptable for seconds, minutes, or not at all?

If those answers are unclear, invalidation bugs are usually inevitable.

## Cache Types

| Cache Type | Example | Common Invalidation Trigger |
|------------|---------|-----------------------------|
| In-memory client cache | Query library cache, store | Mutation, window focus, timer |
| Browser HTTP cache | `Cache-Control`, ETag | TTL expiry, revalidation |
| CDN / edge cache | HTML, JSON, images | Purge, tag revalidation, TTL |
| Server application cache | Redis, memory cache | Write-through, pub/sub, TTL |
| Build-time cache | Static generation output | Rebuild, webhook, revalidate |

Most real apps use several of these at once, so invalidation often has to happen at multiple layers.

## Freshness vs Correctness

Caching always trades freshness for speed.

| Strategy | Speed | Freshness | Use Case |
|----------|-------|-----------|----------|
| Long TTL, no eager revalidation | Highest | Lowest | Rarely changing public content |
| Short TTL | Good | Better | Frequently viewed but non-critical data |
| Mutation-triggered invalidation | Good | High | User-owned records, dashboards |
| Immediate replace with canonical response | Good | Highest | Mutations returning updated record |
| No cache | Lowest | Highest | High-risk data where staleness is unacceptable |

Not all stale data is a bug. The right question is whether the staleness is acceptable for that feature.

## Cache Keys

Good invalidation starts with good cache keys.

### Characteristics of Good Keys

- Stable for the same resource
- Specific enough to avoid collisions
- Structured enough to target related entries

```javascript
['products']
['products', { page: 2, sort: 'price' }]
['product', productId]
['user', userId, 'orders']
```

### Bad Keys Cause Bad Invalidation

Common failures:

- Using a single generic key for unrelated data
- Omitting filters, pagination, or locale from the key
- Using unstable objects that change identity every render
- Mixing list keys and detail keys without a clear convention

If the key design is sloppy, invalidation becomes broad, expensive, or incorrect.

## Common Invalidation Strategies

### 1. Time-Based Invalidation

Cached data expires after a freshness window.

```javascript
cache.set(key, data, { ttl: 60000 });
```

Best for:
- Public content
- Reference data
- Data where slight staleness is acceptable

Pros:
- Simple
- Predictable
- No mutation wiring required

Cons:
- Data may remain stale until TTL expires
- Very short TTLs increase refetch load

### 2. Mutation-Driven Invalidation

When a write succeeds, invalidate affected reads.

```javascript
await api.updateProduct(productId, input);
cache.invalidate(['product', productId]);
cache.invalidate(['products']);
```

Best for:
- CRUD interfaces
- User dashboards
- Settings pages

Pros:
- More accurate than pure TTLs
- Aligns with known write events

Cons:
- Requires careful mapping between writes and reads
- Easy to miss related list/detail/aggregate queries

### 3. Revalidation on Access

Serve cached data first, then verify it in the background.

```text
Read cache -> show cached value -> revalidate -> update if changed
```

Best for:
- Feeds
- Dashboards
- Pages where responsiveness matters more than perfect immediacy

This is the basic stale-while-revalidate model.

### 4. Event-Driven Invalidation

A cache is invalidated in response to external events.

Examples:
- CMS publish webhook
- WebSocket event
- Database change event
- Admin action from another device

Best for:
- Collaborative systems
- Multi-device apps
- CMS-backed sites

### 5. Tag or Group Invalidation

Invalidate all entries associated with a shared tag or group.

Examples:
- All pages tagged `products`
- All entries for tenant `acme`
- All cached responses affected by a category update

Best for:
- CDNs
- Static site revalidation
- Large systems with many derived views

## List and Detail Invalidation

A very common bug: a mutation updates the detail view but leaves lists stale.

Example:

- `/products/123` shows the updated price
- `/products?page=1` still shows the old price
- `/search?q=shoe` still shows old summary data

When a record changes, consider all derived reads:

- Detail query for that record
- Any list containing that record
- Filtered or paginated variants
- Aggregate counts or summaries
- Related views such as search results or dashboards

If invalidating all related data is too expensive, prefer replacing known entries directly and invalidating the rest lazily.

## Replace vs Invalidate

After a mutation, you often have two choices.

### Replace Cached Data Directly

Use when the mutation response contains canonical updated data.

```javascript
const updated = await api.updateTodo(id, input);
cache.set(['todo', id], updated);
```

Good for:
- Detail records
- Small targeted updates
- Mutation responses that return the full authoritative record

### Invalidate and Refetch

Use when the mutation affects more data than you can patch safely.

```javascript
await api.updateTodo(id, input);
cache.invalidate(['todos']);
```

Good for:
- Complex lists
- Aggregates
- Server-side derived data
- Permissions or sorting logic controlled by backend

### Hybrid Approach

Often best:

- Replace the precise record you know
- Invalidate broader derived queries

## Broad vs Narrow Invalidation

| Approach | Example | Tradeoff |
|----------|---------|----------|
| Narrow | Invalidate `['product', 123]` only | Efficient, but easy to miss related views |
| Medium | Invalidate detail + product lists | Balanced default for CRUD apps |
| Broad | Invalidate all product-related queries | Safe, but more network churn |

Start as narrow as correctness allows, but not narrower.

If you routinely ship stale data bugs, your invalidation is probably too narrow.

## Optimistic Updates and Invalidation

Optimistic UI does not remove the need for invalidation.

After an optimistic mutation:

1. Apply the optimistic patch
2. Send the mutation
3. Reconcile with server response
4. Invalidate or refresh dependent queries

Without step 4, adjacent views often remain stale even when the edited component looks correct.

## Pagination, Filters, and Search

Invalidation becomes harder once data appears in many shapes.

Examples:

- `['products', { page: 1 }]`
- `['products', { page: 2 }]`
- `['products', { category: 'shoes' }]`
- `['search', { q: 'running' }]`

When a product changes, you may not know every cached list that contains it.

Common approaches:

- Broadly invalidate list families such as all `products` queries
- Patch only the currently visible list and invalidate the rest
- Use tags so related cached views can be purged together

## HTTP and CDN Caching

Not all invalidation happens in JavaScript state libraries.

### HTTP Freshness Controls

Common tools:

- `Cache-Control: max-age=60`
- `ETag` and conditional requests
- `Last-Modified`
- `stale-while-revalidate`

These are useful when the browser or CDN should participate in caching decisions.

### CDN / Edge Revalidation

Useful for:
- Static pages backed by CMS content
- Product pages generated ahead of time
- API responses cached near users

Common invalidation mechanisms:

- TTL expiry
- Explicit purge by URL
- Tag-based revalidation
- Webhook-triggered regeneration

## Common Bugs

| Bug | Cause |
|-----|-------|
| Updated detail view, stale list | Related list query not invalidated |
| User sees old data after mutation | Background refetch won the race |
| Refetch storm | Invalidation is too broad or too frequent |
| Wrong tenant/user data | Cache key missing auth or tenant scope |
| Stale search results | Search and filtered variants not included in strategy |
| Flickering UI | Cache replaced repeatedly with partial or out-of-order data |

## Practical Patterns

### Mutation with Targeted Invalidation

```javascript
const updated = await api.updatePost(postId, input);

cache.set(['post', postId], updated);
cache.invalidate(['posts']);
cache.invalidate(['dashboard']);
```

### TTL Plus Mutation Invalidation

Use both when reads happen often but writes still need fast correctness.

```javascript
useQuery({
  key: ['notifications'],
  ttl: 30000,
});

await api.markAllRead();
cache.invalidate(['notifications']);
```

### Webhook-Based Static Revalidation

```text
CMS publish -> webhook -> purge or regenerate affected pages -> next request gets fresh HTML
```

## Choosing a Strategy

Use this decision guide:

- Rarely changing public data: TTL or static regeneration
- User-edited CRUD data: mutation-driven invalidation
- Collaborative or multi-device data: event-driven invalidation
- Cheap reads with high correctness needs: invalidate and refetch
- Canonical mutation response available: replace exact cache entry, then invalidate broader derived views if needed

## Rules of Thumb

- Design cache keys before designing invalidation
- Invalidate based on data relationships, not component boundaries
- Prefer canonical server responses over guessed local patches when possible
- Broad invalidation is often safer than subtly wrong narrow invalidation
- Measure refetch cost, but do not optimize at the expense of correctness too early

## Related Skills

- `data-fetching-patterns` for loading and caching flows
- `state-management-patterns` for server-state boundaries
- `optimistic-ui-patterns` for mutation reconciliation and pending UI

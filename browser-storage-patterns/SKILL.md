---
name: browser-storage-patterns
description: Explains browser storage patterns including cookies, localStorage, sessionStorage, and when to use each. Use when deciding how to persist client-side data, handle auth/session concerns, or choose safe storage patterns for preferences, drafts, and browser state.
---

# Browser Storage Patterns

## Overview

Browser storage is for data that must survive beyond a single render or request.

Common questions:

- Should this live in a cookie or in `localStorage`?
- Should it persist across tabs or only this tab?
- Does the server need to read it?
- Is the data sensitive?
- What happens during SSR when browser APIs do not exist yet?

Choosing the wrong storage mechanism creates security, UX, and correctness problems quickly.

## Storage Options

| Storage | Best For | Scope | Sent to Server | Persistence |
|---------|----------|-------|----------------|-------------|
| Cookies | Sessions, server-readable flags | Per browser/profile | Yes | Configurable |
| `localStorage` | Preferences, low-sensitivity persistence | Per origin | No | Until cleared |
| `sessionStorage` | Tab-scoped drafts, flow state | Per tab | No | Until tab closes |
| IndexedDB | Larger structured offline data | Per origin | No | Until cleared |

For most app decisions, the key choice is between cookies, `localStorage`, and `sessionStorage`.

## Decision Guide

Use this rough mapping:

- Server needs the value on every request: cookie
- Value is a non-sensitive browser preference: `localStorage`
- Value should disappear when the tab closes: `sessionStorage`
- Value is large, structured, or offline-heavy: IndexedDB

If the data is sensitive, do not default to `localStorage`.

## Cookies

Cookies are small key-value pairs stored by the browser and attached to matching HTTP requests.

They are useful when:

- the server needs session or preference data
- auth/session state must be checked during SSR
- middleware or backend routing depends on the value

### Good Cookie Use Cases

- session IDs
- refresh/session tokens in `HttpOnly` cookies
- locale preference needed during server render
- A/B assignment flags that server must read

### Important Cookie Attributes

- `HttpOnly`: JavaScript cannot read it
- `Secure`: sent only over HTTPS
- `SameSite`: limits cross-site sending behavior
- `Max-Age` / `Expires`: controls lifetime
- `Path` / `Domain`: controls scope

### Security Rule

For auth sessions, `HttpOnly` cookies are usually safer than storing tokens in `localStorage`.

## localStorage

`localStorage` is a synchronous browser key-value store that persists until cleared.

It is useful when:

- the value is only needed in the browser
- persistence across reloads matters
- the data is small and not sensitive

### Good localStorage Use Cases

- theme preference
- dismissed banner state
- recently used UI settings
- last selected workspace or view

### Example

```javascript
const theme = localStorage.getItem('theme') || 'light';
localStorage.setItem('theme', 'dark');
```

### Caveats

- Values are strings only
- Access is synchronous
- Not available during server render
- Users can clear it
- It should not hold secrets you would not expose to client JavaScript

## sessionStorage

`sessionStorage` is similar to `localStorage`, but scoped to one browser tab.

It is useful when:

- data should not survive closing the tab
- a multi-step flow should be isolated to one tab
- a draft or wizard should not leak between windows

### Good sessionStorage Use Cases

- checkout step state
- unsaved form draft for the current tab
- temporary redirect return state
- one-tab-only workflow context

### Example

```javascript
sessionStorage.setItem('draft-id', draftId);
const currentDraft = sessionStorage.getItem('draft-id');
```

## IndexedDB

IndexedDB is more powerful than `localStorage`, but heavier to work with.

Use it when:

- data is large
- records need indexing or structured queries
- offline-first behavior matters
- you need more than a few simple key-value entries

Do not reach for IndexedDB if a small preference key is all you need.

## Common Usage Patterns

### Pattern 1: Theme and UI Preferences

Store low-risk, user-owned preferences in `localStorage`.

```javascript
const savedTheme = localStorage.getItem('theme') || 'system';
```

Good examples:

- theme
- density mode
- sidebar collapsed state
- language preference when server does not need it

### Pattern 2: Session Auth with Cookies

Store auth session state in secure cookies when the server must participate in auth decisions.

Good examples:

- server-rendered dashboard auth
- protected API routes
- middleware-based redirects

This works well because the server can check auth before sending protected content.

### Pattern 3: Tab-Scoped Draft State

Use `sessionStorage` for workflows that should survive refresh but not be shared across tabs.

Good examples:

- multi-step form progress
- payment flow return state
- partially completed draft in one tab

### Pattern 4: Client-Only Dismissal State

Use `localStorage` for UI that should stay dismissed across visits.

```javascript
const dismissed = localStorage.getItem('promo-dismissed') === 'true';
```

Examples:

- “What’s new” banners
- helper tooltips
- onboarding prompts

### Pattern 5: Hybrid Session + Preference Model

A common production setup:

- auth session in `HttpOnly` cookie
- UI preferences in `localStorage`
- flow-specific temporary state in `sessionStorage`

This keeps sensitive state server-readable while leaving browser-only UI preferences in client storage.

## Security Patterns

Storage decisions are often security decisions.

### Safer Defaults

- Keep auth sessions in `HttpOnly` cookies
- Keep secrets and tokens out of `localStorage` when possible
- Treat all client-readable storage as potentially exposed to XSS
- Store only the minimum necessary browser data

### Why localStorage Is Risky for Sensitive Tokens

If malicious JavaScript runs in the page, it can usually read `localStorage`.

That means storing highly sensitive tokens there increases the blast radius of XSS.

## SSR and Hydration Caveats

Browser storage APIs do not exist during server render.

This means code like this will break in SSR environments:

```javascript
const theme = localStorage.getItem('theme');
```

Safer pattern:

```javascript
let theme = 'light';

if (typeof window !== 'undefined') {
  theme = localStorage.getItem('theme') || 'light';
}
```

For values needed before paint, consider whether the server should read a cookie instead.

## Expiry and Staleness

`localStorage` does not expire by itself.

If data should go stale, add your own timestamp.

```javascript
localStorage.setItem('notice', JSON.stringify({
  value: true,
  expiresAt: Date.now() + 86400000,
}));
```

This is useful for:

- banner dismissals
- temporary experiments
- recently viewed items

## Cross-Tab Behavior

`localStorage` is shared across tabs for the same origin.

That is useful for:

- syncing theme changes
- logging out all tabs after a state change

But it can also be surprising if a workflow is meant to stay isolated.

Use `sessionStorage` when tab isolation matters more than shared persistence.

## Serialization Patterns

Browser storage stores strings.

Use JSON carefully:

```javascript
localStorage.setItem('prefs', JSON.stringify({ theme: 'dark' }));
const prefs = JSON.parse(localStorage.getItem('prefs') || '{}');
```

Guard against:

- invalid JSON
- missing keys
- schema changes across releases

If the schema evolves, add versioning or migration logic for important stored data.

## Failure Cases

Storage access is not always guaranteed.

Possible issues:

- storage disabled
- private/incognito restrictions
- quota exceeded
- parse errors from old data

Good pattern:

- wrap reads and writes in safe helpers
- fall back to in-memory defaults if needed

## Common Anti-Patterns

| Anti-Pattern | Why It Hurts |
|-------------|--------------|
| Storing auth secrets in `localStorage` by default | Increases XSS risk |
| Using cookies for large client-only state | Bloats requests |
| Using `localStorage` for tab-scoped flows | State leaks across tabs |
| Reading browser storage during SSR without guards | Breaks server render |
| Storing complex app data with no versioning | Upgrade and parse issues |

## Rules of Thumb

- Use cookies when the server must read the value
- Use `localStorage` for small, non-sensitive persistent browser preferences
- Use `sessionStorage` for tab-scoped temporary flows
- Prefer `HttpOnly` cookies for auth session state
- Add your own expiry logic when persistent client data should age out
- Treat all client-readable storage as untrusted and mutable

## Related Skills

- `state-management-patterns` for browser state versus server state boundaries
- `routing-patterns` for flow state and navigation context
- `error-handling-patterns` for storage fallbacks and degraded behavior
- `universal-javascript-runtimes` for SSR and runtime constraints

---
name: optimistic-ui-patterns
description: Explains optimistic UI updates, mutation lifecycles, rollback strategies, temporary IDs, and reconciliation patterns. Use when implementing fast-feeling mutations, pending states, undo flows, or conflict handling for user actions before the server responds.
---

# Optimistic UI Patterns

## Overview

Optimistic UI updates the interface **before** the server confirms a mutation.

Instead of waiting for the round trip:

```text
Click -> Request -> Response -> UI changes
```

the application behaves like this:

```text
Click -> UI changes immediately -> Request -> Confirm or rollback
```

This makes products feel much faster, but it shifts complexity into rollback, reconciliation, and failure handling.

## When Optimistic UI Works Well

| Good Fit | Why |
|----------|-----|
| Likes, favorites, toggles | Small, reversible state changes |
| Adding a comment or message | User expects immediate feedback |
| Reordering items | Local intent is clear before server responds |
| Updating a label or preference | Easy to patch and easy to revert |
| Mark as read / complete | Often low-risk and easy to reconcile |

## When to Avoid or Limit It

| Risky Case | Why |
|------------|-----|
| Payments and financial actions | Failure cost is high |
| Destructive actions without undo | Rollback may be confusing or impossible |
| Inventory-limited operations | Server truth may differ quickly |
| Permission-sensitive changes | Client may assume access it does not have |
| Complex server-side validation | Failure may be common or multi-field |

For high-risk mutations, prefer one of these:

- Show an immediate pending state without changing final UI yet
- Use optimistic UI only for the local shell, not the actual data commit
- Offer undo instead of silent rollback

## Mutation Lifecycle

Most optimistic flows follow the same lifecycle:

1. Capture the user's intent
2. Save enough information to undo the change
3. Apply an optimistic patch locally
4. Mark the record or action as pending
5. Send the mutation to the server
6. Reconcile with the server response
7. Clear pending state or rollback on failure

## Core Building Blocks

### 1. Optimistic Patch

Apply the exact state change the user expects to see immediately.

```javascript
setLiked(true);
```

### 2. Rollback Information

Store either the previous snapshot or an inverse patch.

```javascript
const previousLiked = liked;
setLiked(true);

try {
  await api.likePost(postId);
} catch {
  setLiked(previousLiked);
}
```

### 3. Pending Metadata

Track whether the change is still in flight.

```javascript
setPost({
  ...post,
  liked: true,
  _optimistic: true,
});
```

Pending metadata is useful for:

- Disabling repeat submissions
- Showing spinners or subtle "syncing" states
- Styling temporary items differently
- Preventing stale server responses from overwriting newer intent

### 4. Server Reconciliation

The server response is the canonical truth.

Even if the optimistic value matched, the response may include:

- Server-generated IDs
- Sanitized text
- Updated timestamps
- Derived counters
- Permission or validation adjustments

Always merge or replace local optimistic state with the canonical response.

## Common Patterns

### Toggle Mutation

Good for likes, bookmarks, or completion state.

```jsx
function LikeButton({ postId, initialLiked }) {
  const [liked, setLiked] = useState(initialLiked);
  const [pending, setPending] = useState(false);

  async function onClick() {
    const previous = liked;
    const next = !liked;

    setLiked(next);
    setPending(true);

    try {
      await api.updateLike(postId, next);
    } catch {
      setLiked(previous);
    } finally {
      setPending(false);
    }
  }

  return (
    <button onClick={onClick} aria-busy={pending}>
      {liked ? 'Liked' : 'Like'}
    </button>
  );
}
```

### Create with Temporary IDs

Useful when inserting a new item into a list before the backend returns the real ID.

```javascript
const tempId = `temp-${Date.now()}`;
const optimisticComment = {
  id: tempId,
  body: text,
  status: 'sending',
};

setComments((items) => [optimisticComment, ...items]);

try {
  const saved = await api.createComment({ body: text });
  setComments((items) =>
    items.map((item) => (item.id === tempId ? saved : item))
  );
} catch {
  setComments((items) => items.filter((item) => item.id !== tempId));
}
```

### Reorder with Rollback

Reordering usually feels broken if it waits for the network.

```javascript
const previousOrder = items;
const nextOrder = reorder(items, sourceIndex, destinationIndex);

setItems(nextOrder);

try {
  await api.reorderItems(nextOrder.map((item) => item.id));
} catch {
  setItems(previousOrder);
}
```

### Counter Updates

For counts such as likes or cart quantity, patch both the record and any displayed aggregate carefully.

```javascript
setPost((post) => ({
  ...post,
  liked: true,
  likeCount: post.likeCount + 1,
}));
```

This breaks down when concurrent updates are common. In that case, prefer reconciling with the server's returned count instead of trusting the local increment long-term.

## Cache-Based Optimistic Updates

When using a server-state library, optimistic UI is often applied directly to the cache.

Conceptually:

1. Cancel or pause competing refetches
2. Snapshot the previous cached value
3. Write the optimistic value into cache
4. Roll back if the mutation fails
5. Invalidate or replace cache with server truth

```javascript
const previous = cache.get(['todos']);

cache.set(['todos'], (items) => [
  { id: tempId, text, status: 'sending' },
  ...items,
]);

try {
  const saved = await api.createTodo({ text });
  cache.set(['todos'], (items) =>
    items.map((item) => (item.id === tempId ? saved : item))
  );
} catch {
  cache.set(['todos'], previous);
} finally {
  cache.invalidate(['todos']);
}
```

The exact APIs vary across TanStack Query, SWR, Apollo, Relay, and framework-specific data layers, but the lifecycle is the same.

## Concurrency and Race Conditions

Optimistic UI becomes difficult when multiple mutations affect the same record.

### Common Failure Modes

| Problem | Example |
|---------|---------|
| Out-of-order responses | Request A finishes after Request B and overwrites newer state |
| Duplicate submissions | User clicks twice and sends same mutation twice |
| Conflicting local edits | User changes a field again before prior save settles |
| Background refetch overwrite | Fresh fetch removes optimistic patch too early |

### Practical Defenses

- Attach a client request ID to each mutation
- Ignore stale responses older than the latest local intent
- Queue mutations for the same record when order matters
- Use version numbers or `updatedAt` timestamps if the backend supports them
- Pause or coordinate refetches while optimistic patches are active

## Rollback Strategies

### Full Rollback

Restore the previous snapshot exactly.

Best for:
- Simple toggles
- Single-record edits
- Reorder operations

### Partial Rollback

Keep some local UI state while correcting only server-owned fields.

Best for:
- Draft forms
- Complex editors
- Mutations with derived server fields

### Undo Instead of Rollback

The UI accepts the change immediately and offers a short-lived undo action.

Best for:
- Archive
- Delete to trash
- Dismiss actions

This often feels better than a silent rollback because the user understands what happened.

## UX Guidelines

### Show Pending State Without Killing Momentum

Good optimistic UI is immediate, but it should not lie.

Use subtle indicators such as:

- Reduced opacity for unsynced items
- A "Sending..." label for temporary records
- Disabled destructive follow-up actions while pending
- A retry affordance on failure

### Make Failure Visible

Silent rollback is confusing when the user already saw the successful state.

Prefer:

- Inline error near the changed item
- Toast plus retry when the action is important
- Undo or conflict messaging when state may have changed elsewhere

### Preserve User Trust

If optimistic failures are frequent, the feature feels broken.

A good rule:

- High success rate -> optimistic UI works well
- Frequent validation or permission failures -> use conservative pending UI instead

## Backend Requirements That Help

Optimistic UI is easier when the backend supports:

- Idempotent mutation endpoints
- Stable resource identifiers
- Server-returned canonical records
- Version fields or timestamps
- Clear error codes for validation vs authorization vs conflict

Frontend optimism is much safer when backend mutation semantics are predictable.

## Decision Checklist

Use optimistic UI when most answers below are "yes":

- Is the user intent immediately clear?
- Is the change reversible or easy to correct?
- Is failure relatively rare?
- Can the previous state be restored safely?
- Can the server return canonical data for reconciliation?

If several answers are "no", prefer a fast pending state over a true optimistic commit.

## Related Skills

- `state-management-patterns` for client state vs server state boundaries
- `data-fetching-patterns` for cache coordination and invalidation
- `routing-patterns` when optimistic state is reflected in the URL

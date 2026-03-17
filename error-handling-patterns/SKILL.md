---
name: error-handling-patterns
description: Explains error handling patterns in web applications including error boundaries, retries, recovery UX, normalized error shapes, graceful degradation, and observability hooks. Use when designing how applications should fail, recover, and communicate problems to users.
---

# Error Handling Patterns

## Overview

Error handling is not only about preventing crashes. It is about deciding:

- what the user should see
- what can be retried
- what state should be preserved
- what should be logged or reported
- when the system should fail fast vs degrade gracefully

Good error handling makes failures understandable, containable, and recoverable.

## Types of Errors

Different failures need different handling.

| Error Type | Example | Typical Response |
|-----------|---------|------------------|
| Validation | Invalid form input | Inline correction guidance |
| Network | Timeout, offline, DNS failure | Retry, offline fallback, preserve user intent |
| Authorization | 401, 403 | Re-auth, permission message, redirect |
| Server | 500, dependency outage | Retry or graceful fallback |
| Not found | Missing route or deleted record | Not-found UI, navigation recovery |
| Conflict | Stale write, version mismatch | Refresh, merge, or ask user to resolve |
| Rendering | Component crash | Error boundary, fallback UI |
| Background failure | Polling or refresh failure | Non-blocking notification, degraded data state |

Treating all failures the same leads to confusing UI and bad recovery behavior.

## Error Handling Goals

Good systems try to:

- avoid unnecessary crashes
- preserve user work when possible
- explain the problem in actionable language
- offer the next best recovery path
- capture enough telemetry to diagnose the issue later

## Layered Error Handling

Handle errors at the right layer.

### Component / Inline Layer

Good for:

- form validation
- failed mutation on one record
- retrying a single action

### Page / Route Layer

Good for:

- failed page data load
- access-denied or not-found states
- route-specific crash recovery

### Application / Global Layer

Good for:

- uncaught rendering crashes
- catastrophic boot failures
- unexpected fatal conditions

### Service / Backend Layer

Good for:

- standardizing error responses
- mapping dependency errors
- adding request IDs and diagnostics

Not every error should bubble to the top of the app. Most should be handled close to the failing task.

## Normalize Error Shapes

Applications become easier to handle when errors are mapped into a consistent shape.

```javascript
{
  code: 'NETWORK_TIMEOUT',
  message: 'Request timed out',
  retryable: true,
  statusCode: 504,
  requestId: 'req_123'
}
```

Useful normalized fields:

- machine-readable code
- human-readable message
- retryability
- status or category
- request or trace ID

Without normalization, each component has to guess how to interpret failures.

## User-Facing Error UX

The UI should communicate what happened and what the user can do next.

### Good Error Messages

- Say what failed
- Say whether the user can retry or fix something
- Avoid internal jargon
- Preserve technical detail for logs, not for primary UI

Bad:

```text
Unknown exception occurred
```

Better:

```text
We couldn’t save your changes. Check your connection and try again.
```

## Inline vs Global Errors

| UI Placement | Best For |
|-------------|----------|
| Inline near field | Validation and input-specific errors |
| Inline near component | Failed widget or card action |
| Page-level message | Page data load failure |
| Toast | Secondary status updates, non-blocking failures |
| Full-screen fallback | Fatal route or app crash |

If the failure affects only one action, keep the error near that action.

## Preserve User Intent

Failure handling is better when users do not lose work.

Examples:

- keep form values after failed submit
- keep draft comment text after network failure
- keep optimistic state recoverable when server rejects it
- preserve navigation context when auth expires

Losing work turns a recoverable failure into a frustrating experience.

## Retry Patterns

Retries are useful for transient failures, not permanent ones.

Good retry candidates:

- network timeouts
- temporary 5xx errors
- flaky third-party dependency calls

Bad retry candidates:

- validation errors
- permission errors
- malformed requests

### Backoff Pattern

```javascript
for (let attempt = 0; attempt < 3; attempt++) {
  try {
    return await request();
  } catch (error) {
    if (!isRetryable(error)) throw error;
    await wait(2 ** attempt * 500);
  }
}
```

Retries should usually use backoff and a cap. Blind aggressive retries can amplify outages.

## Idempotency and Safe Recovery

Retries are much safer when the backend supports idempotent operations.

Examples:

- retrying a GET is usually safe
- retrying “create payment” may not be safe without an idempotency key
- retrying “save draft” is safer when updates replace state rather than append state

Frontend recovery design is stronger when backend semantics are predictable.

## Error Boundaries

Rendering crashes should not take down the whole application if they can be contained.

Error boundaries are useful for:

- route-level fallback UI
- isolating risky widgets
- keeping the shell usable when one region fails

### Conceptual Example

```jsx
<ErrorBoundary fallback={<WidgetError />}>
  <RevenueChart />
</ErrorBoundary>
```

Important limitation:

- rendering boundaries do not catch every async error automatically
- async failures in effects, requests, or event handlers still need explicit handling

## Async Error States

Many application failures happen in async flows:

- page load
- search
- mutation submit
- background refresh
- realtime reconnect

These need explicit UI states:

- loading
- success
- empty
- error
- retrying
- stale or degraded

If those states are not modeled directly, async errors usually leak into inconsistent UI.

## Graceful Degradation

Sometimes the best response is not full failure but reduced capability.

Examples:

- show cached data while refresh fails
- disable one advanced widget while core page remains usable
- fall back from realtime updates to polling
- serve static or stale content during partial backend outage

Graceful degradation is valuable when:

- the core task can still continue
- the degraded behavior is honest and understandable

## Page Load Error Patterns

### Full-Page Failure

Use when the entire route depends on one failed resource.

Typical options:

- retry button
- back navigation
- support link or status page

### Partial Failure

Use when one page section failed but the rest still works.

Typical options:

- failed widget fallback
- per-section retry
- degraded view with timestamp or stale indicator

Partial failures are often better than blank screens.

## Mutation Error Patterns

Mutations need careful handling because user intent is active and recent.

Common rules:

- keep input or draft data
- show the error close to the failed action
- allow retry when appropriate
- roll back optimistic state if needed
- do not silently fail

Example:

```javascript
try {
  await saveProfile(input);
  setStatus('saved');
} catch (error) {
  setStatus('error');
  setErrorMessage(toUserMessage(error));
}
```

## Auth and Permission Errors

Auth failures need different handling from generic server errors.

Examples:

- `401`: session expired, needs login refresh
- `403`: user authenticated but not allowed

Recommended responses:

- preserve current location when redirecting to login
- show permission errors clearly
- avoid looping redirects on expired sessions

## Accessibility in Error Handling

Errors should be perceivable and navigable.

Important patterns:

- move focus to error summary or first invalid field after submit
- announce dynamic errors with appropriate live regions when focus does not move
- do not rely on color alone
- keep messages visible long enough to read

Accessible error handling is part of functional correctness, not just polish.

## Observability Hooks

Not every user-facing error should become a noisy alert, but important failures should be observable.

Useful data to capture:

- error code
- route or feature
- release version
- request or trace ID
- retry count
- whether the error reached the user

This helps distinguish:

- noisy internal events
- actual user-impacting failures

## Common Anti-Patterns

| Anti-Pattern | Why It Hurts |
|-------------|--------------|
| Generic “Something went wrong” everywhere | Gives no recovery path |
| Swallowing errors silently | Fails without user or developer visibility |
| Retrying non-retryable failures | Wastes time and increases confusion |
| Clearing user input on failed submit | Loses work |
| Treating empty state as error | Produces wrong UX |
| Full-page crash for one failed widget | Over-scopes the failure |

## Decision Guide

Use this rough mapping:

- Invalid user input: inline validation and correction
- Failed data load: route fallback or section fallback with retry
- Failed mutation: preserve intent, show local error, retry if safe
- Expired session: re-auth flow with context preservation
- Uncaught render crash: error boundary
- Dependency degradation: graceful fallback plus telemetry

## Rules of Thumb

- Handle failures as close to the action as possible
- Preserve user work whenever you can
- Retry only when the failure is likely transient
- Standardize error shapes early
- Distinguish user-fixable from system-fixable problems
- Connect user-visible failures to observability data

## Related Skills

- `observability-patterns` for telemetry, alerts, and production diagnosis
- `accessibility-interaction-patterns` for perceivable and navigable error feedback
- `optimistic-ui-patterns` for rollback and mutation recovery
- `data-fetching-patterns` for load, stale, and retry flows

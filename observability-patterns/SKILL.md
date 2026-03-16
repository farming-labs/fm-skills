---
name: observability-patterns
description: Explains observability patterns for web applications including logs, metrics, traces, client error reporting, correlation IDs, SLOs, and alert design. Use when debugging production issues, designing telemetry, or deciding what signals to collect from frontend and backend systems.
---

# Observability Patterns

## Overview

Observability is how you understand what a system is doing in production.

It answers questions like:

- Why is this page slow for some users?
- Which request path is failing?
- Did this deploy increase error rate?
- Which backend dependency is causing latency?
- Are users affected, or is this only an internal warning?

Observability is not just logging. It is the combined use of:

- logs
- metrics
- traces
- error reporting
- dashboards and alerts

## The Core Signals

| Signal | Best For | Weakness |
|--------|----------|----------|
| Logs | Detailed event context | High volume, harder to aggregate |
| Metrics | Trends, rates, latency, alerting | Lose per-event detail |
| Traces | Following one request across services | More setup and sampling tradeoffs |
| Error reporting | Grouping crashes and exceptions | Misses non-crash degradation |

Good observability uses all of them together instead of expecting one signal to answer every question.

## Logs

Logs record discrete events with contextual fields.

Useful examples:

- HTTP request completed
- API validation failed
- Background job retried
- Client mutation failed
- User-facing error boundary activated

### Prefer Structured Logs

```json
{
  "level": "error",
  "message": "Payment authorization failed",
  "requestId": "req_123",
  "userId": "user_42",
  "route": "/checkout",
  "statusCode": 502
}
```

Structured logs are better than free-form text because they can be filtered, grouped, and correlated reliably.

### Logging Rules of Thumb

- Log meaningful events, not every line of code
- Include identifiers needed for correlation
- Avoid leaking secrets, tokens, and sensitive personal data
- Keep log volume intentional or costs become unmanageable

## Metrics

Metrics show how the system behaves over time.

Common metric families:

- request rate
- error rate
- latency
- queue depth
- cache hit rate
- active connections
- Core Web Vitals

### Good Metric Questions

- Is traffic increasing or dropping?
- Is p95 latency regressing?
- Did error rate spike after a deployment?
- Are retries masking a deeper outage?

### Percentiles Matter

Averages often hide user pain.

Example:

- average latency: 180ms
- p95 latency: 1.8s

That usually means many users are fine while a meaningful minority are having a bad experience.

## Traces

Tracing follows one request or user action across system boundaries.

This is especially useful when:

- a frontend action calls multiple APIs
- one API fans out to several services
- latency is coming from an unknown dependency
- the error is intermittent and hard to reproduce

### Trace Shape

```text
User click
  -> browser request
    -> edge middleware
      -> app server
        -> database query
        -> cache lookup
        -> third-party API
```

Traces help answer not just "what failed?" but "where in the chain did time or failure accumulate?"

## Error Reporting

Error reporting is a specialized signal for exceptions, crashes, and grouped failures.

For web applications, useful sources include:

- uncaught client exceptions
- rejected promises
- server exceptions
- failed background jobs
- hydration mismatches that affect users

Error reporting is strongest when each event includes:

- release version
- environment
- route
- request or trace ID
- user or tenant scope when safe

## Frontend Observability

Frontend systems need observability too, not just backend services.

Important client-side signals:

- JavaScript exceptions
- failed network requests
- Core Web Vitals
- route transition latency
- hydration failures
- feature-specific failure rates

### Useful Frontend Questions

- Is this error isolated to one browser?
- Did a new deployment hurt INP or LCP?
- Which route has the highest client error rate?
- Are users retrying a broken mutation repeatedly?

## Backend Observability

Important backend signals:

- request throughput
- response codes
- latency percentiles
- dependency latency
- database errors and query timing
- queue and worker health
- cache hit/miss rates

Backend telemetry should make it easy to distinguish:

- application bugs
- infrastructure failures
- dependency failures
- bad input or abuse

## Correlation IDs

Correlation IDs connect signals across layers.

Examples:

- request ID for a single HTTP request
- trace ID across multiple services
- session or flow ID for a user journey

### Why They Matter

Without correlation IDs:

- logs are hard to connect
- frontend and backend failures look unrelated
- one bad request disappears in noisy event streams

With them, you can move from:

- client error event
- to failing network request
- to backend logs
- to a slow database query

## What to Instrument

Instrument user flows and system boundaries first.

Good starting points:

- route requests
- route transitions
- API calls
- cache lookups
- database queries
- queue jobs
- external service calls
- form submission failures
- auth and permission failures

Avoid starting with hyper-granular internal events that generate noise without helping diagnosis.

## Golden Signals

A practical baseline is to watch these:

- latency
- traffic
- errors
- saturation

Examples:

- API p95 latency
- requests per minute
- error rate by route
- worker queue backlog
- CPU or database connection pool saturation

These are often the fastest path to spotting active incidents.

## SLOs and User Impact

Observability is most useful when tied to user experience, not just system internals.

Examples:

- 99% of page loads render primary content within target time
- 99.9% of API requests succeed
- checkout flow error rate stays below threshold

This helps separate:

- noisy but acceptable background issues
- user-visible incidents that deserve paging

## Alerting Patterns

Alerts should be actionable, not just noisy.

### Good Alerts

- sustained error-rate increase
- p95 latency regression beyond threshold
- job backlog growing for a meaningful duration
- deploy-correlated crash spike

### Bad Alerts

- single isolated exception
- every retry attempt
- metrics with no known owner or response playbook

If an alert does not tell someone what to investigate next, it is probably not a good alert.

## Dashboards

Dashboards are for fast situational awareness.

Useful dashboard slices:

- traffic, error rate, and latency by service
- route-level frontend health
- release/version comparison
- dependency health
- queue and worker state

Dashboards should help answer:

- Is something broken right now?
- Where is the damage concentrated?
- Did this start after a release?

## Sampling and Cost

Telemetry is not free.

Common tradeoffs:

- log retention cost
- trace sampling rate
- high-cardinality metric explosion
- client telemetry bandwidth

### High Cardinality Warning

Avoid unbounded metric labels like:

- raw user ID
- raw request path with dynamic IDs
- free-form error text

Those are usually better as log fields, not metric dimensions.

## Common Anti-Patterns

| Anti-Pattern | Why It Hurts |
|--------------|--------------|
| Logging everything | Expensive and hard to query |
| Metrics without labels or ownership | Hard to act on |
| Alerts on raw noise | Pager fatigue |
| Error events without release/version | Hard to connect to deployments |
| Frontend blind spots | Backend looks healthy while users are failing |
| No correlation IDs | Investigation becomes guesswork |

## Incident Debugging Workflow

A practical workflow:

1. Check alerts or user reports
2. Look at top-level metrics to confirm scope
3. Identify affected route, service, or release
4. Use traces or request IDs to narrow the failing path
5. Inspect structured logs for exact failure context
6. Confirm user impact and whether rollback or mitigation is needed

Good observability shortens this loop.

## Decision Guide

Use this rough mapping:

- Need trend and alerting: metrics
- Need exact event details: logs
- Need cross-service latency analysis: traces
- Need grouped exceptions and deploy regression visibility: error reporting
- Need product-health visibility: frontend telemetry and user-flow metrics

## Rules of Thumb

- Instrument boundaries and critical paths first
- Prefer structured logs over unstructured strings
- Tie telemetry to releases and user impact
- Use correlation IDs everywhere possible
- Alert on symptoms that require action, not on every anomaly
- Observe frontend and backend together

## Related Skills

- `performance-patterns` for user-perceived speed and Core Web Vitals
- `middleware-patterns` for request logging and cross-cutting server instrumentation
- `testing-strategies-patterns` for catching failures before production
- `realtime-patterns` for connection health and delivery guarantees

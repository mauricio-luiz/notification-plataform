# ADR-002: Initial Processing Model (Synchronous, Evolving to Asynchronous)

- **Status:** Accepted
- **Date:** 2026-11-19
- **Related:** [ADR-001](./ADR-001-database.md), [project-foundation requirements](../../specs/project-foundation/requirements.md)

## Context

When an order is completed, the sales system sends a notification request to this API. The API validates it, persists the notification, delivers it through a simulated email service, updates its status and answers over HTTP.

We need to deliver a working system quickly, without distributed infrastructure, but we know the system must later scale horizontally and process notifications independently of incoming HTTP requests (`CLAUDE.md`).

## Decision

### Phase 1 — Synchronous processing (current)

The whole flow runs inside the HTTP request lifecycle:

1. Validate the request at the system boundary.
2. Persist the notification.
3. Deliver it through the simulated email service.
4. Update the notification status.
5. Return the HTTP response.

Rules for this phase:

- Business rules live in a Service, never in controllers or route handlers.
- Delivery sits behind one clearly named service method, so it can later be moved to a background job without changing the controller contract. The method name and signature, and the HTTP response when the simulated email fails, are defined in the notifications specification, not here.
- Laravel runs with `QUEUE_CONNECTION=sync`.
- No Redis, message queue, worker or scheduler is introduced.
- Application instances stay stateless; all state is in PostgreSQL.

### Phase 2 — Asynchronous processing (future)

Delivery will be delegated to background workers through an asynchronous message queue, so notifications are processed independently of HTTP requests and workers scale separately from the API.

This ADR does **not** select the queue technology. That requires its own specification, approval and ADR.

## Alternatives Considered

| Option | Why not chosen now |
|---|---|
| Asynchronous from the start | Adds a queue, workers and failure-handling before the core flow is validated; violates "avoid distributed components before their specification". |
| Synchronous forever | Request latency and failures are coupled to the email service; limits throughput and horizontal scaling. |

## Consequences

- **Positive:** simple to build, test and debug; the HTTP response carries the final delivery result.
- **Negative:** request latency includes delivery time; a slow or failing email service directly affects the API response.
- **Migration:** moving to Phase 2 should change how delivery is triggered (dispatch a job), not the business rules. The HTTP contract will likely change (e.g. a notification accepted as pending instead of already delivered); that change must be specified at that time.
- **Open for the future spec:** queue technology, retry and failure policy, idempotency, and the status model for pending/failed deliveries.

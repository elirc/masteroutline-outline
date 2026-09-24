# ADR-006: Events Table + Redis/Bull Queues as the Async Integration Spine

**Status:** Active-with-Scar-Tissue

## Context

Every mutation has a fan-out of side effects: websocket broadcasts to other
viewers, notifications, webhook deliveries, revision snapshots, search
reindexing, backlink maintenance. Executing these inline would couple API
latency to the slowest integration and make every new side effect a change to
existing handlers. The product also needs an audit trail as a feature
(`server/routes/api/events/`). Constraint: self-hosters will not run Kafka;
Redis is already required.

## Decision

A two-stage spine:

1. **Durable record:** mutations write an `Event` row
   (`server/models/Event.ts`) inside the same transaction as the mutation
   (ADR-005 makes this atomic).
2. **Async fan-out:** after commit, the event is published to Bull queues on
   Redis (`server/queues/`). **Processors**
   (`server/queues/processors/`, extending `BaseProcessor`) subscribe by
   event name and either act directly or enqueue **tasks**
   (`server/queues/tasks/`) for retryable units of work. The `worker` service
   (ADR-001) consumes them. `WebsocketsProcessor.ts` alone maps dozens of
   event names to socket.io room broadcasts; `DebounceProcessor.ts` collapses
   event storms (e.g. rapid edits → one revision via `RevisionsProcessor.ts`).

Delivery semantics: **at-least-once from the queue onward** (Bull retries),
with idempotency left to each consumer.

## Alternatives Considered

1. **Inline side effects.** Rejected: API latency couples to integration
   latency; a Slack outage breaks document saves; every feature edits
   existing handlers.
2. **A real broker (Kafka/NATS/RabbitMQ).** Rejected: replay and
   consumer-group semantics would be genuinely better, but it's a
   disqualifying operational demand for self-hosters. Redis was already in
   the stack for caching and websockets.
3. **Transactional outbox with a relay** (poll the events table, publish,
   mark published). Considered-by-hindsight: the events table *is* an outbox
   in shape, but there is no relay/acknowledgement column — publish happens
   in-process after commit.

## Consequences

**Positive:**
- New side effects are new processors/tasks; existing handlers don't change.
  Adding "notify Discord on publish" touches zero core files (see plugins
  registering `Hook.Processor` / `Hook.Task`).
- The audit log is a free product feature, transactionally correct.
- Retries, backoff, and queue health (`server/queues/HealthMonitor.ts`) come
  from Bull rather than bespoke code.

**Negative:**
- **The commit/publish gap:** the event row commits, then the process
  publishes to Redis. A crash between the two yields an event no processor
  ever sees — at-least-once from the queue, but *at-most-once into* it. No
  relay reconciles the gap. Rare; unhandled; real.
- Unbounded growth: `events` grows with all activity forever —
  `CleanupOldEventsTask.ts` is visible remediation, not original design.
- Blurred failure domains: a poisoned event or slow processor delays
  unrelated consumers sharing worker capacity (websocket lag caused by a
  notifications bug is the canonical war story).
- At-least-once + non-idempotent consumer = duplicate side effects; each
  processor must defend itself, and discipline varies.

## Evidence

- `server/models/Event.ts` — the `globalEventQueue` import; event schema.
- `server/queues/processors/index.ts`, `BaseProcessor.ts` — subscription
  and dispatch.
- `server/queues/processors/WebsocketsProcessor.ts` — the largest consumer.
- `server/queues/processors/DebounceProcessor.ts` +
  `RevisionsProcessor.ts` — event-storm collapsing.
- `server/queues/tasks/CleanupOldEventsTask.ts` — the growth scar.
- `server/queues/HealthMonitor.ts` — queue self-surveillance added because
  queue depth is a production-critical signal.

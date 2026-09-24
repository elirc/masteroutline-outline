# Observability and Operations

**Observability** = can you answer "what is the system doing and why?" from the outside, without adding code. The senior question this page trains: *"how would I know this broke?"* — asked per flow, before shipping.

## What exists here

| Signal | Mechanism | Anchors |
| --- | --- | --- |
| Logs | winston via a central [Logger](../../server/logging/Logger.ts) with categories (`lifecycle`, `http`, `worker`…); `DEBUG` env filters | worker logs include event + attempt ([worker.ts L45–48](../../server/services/worker.ts#L44-L49)) |
| Metrics | statsd via hot-shots ([Metrics](../../server/logging/Metrics.ts)); flushed on shutdown ([server/index.ts L243](../../server/index.ts#L243)) | gauges in websockets service |
| Traces | dd-trace (Datadog APM), initialized *before all instrumented imports* ([server/index.ts L6](../../server/index.ts#L1-L7)); `traceFunction` wrappers name spans per presenter/worker/processor | [tracing.ts](../../server/logging/tracing.ts); auth adds user/team tags to root spans ([authentication.ts L56–65](../../server/middlewares/authentication.ts#L56-L65)) |
| Errors | Sentry client + server (`@sentry/react`, `@sentry/node`); websocket breadcrumbs client-side ([WebsocketProvider L115–122](../../app/components/WebsocketProvider.tsx#L110-L125)) | [app/utils/sentry.ts](../../app/utils/sentry.ts) |
| Health | `/_health` checks Postgres + Redis ([server/index.ts L159–177](../../server/index.ts#L159-L177)); queue [HealthMonitor](../../server/queues/HealthMonitor.ts) | |
| Queue introspection | bull-board UI via the `admin` service | [server/services/admin.ts](../../server/services/admin.ts) |
| Product telemetry that doubles as ops data | SearchQuery durations, View rows, DocumentInsight rollups | [SearchQuery](../../server/models/SearchQuery.ts) |
| Audit | the events table itself — who did what, when, from which IP | [Event.ts](../../server/models/Event.ts) |

Deploy/rollback story (self-hosted reality): Docker image ([Dockerfile](../../Dockerfile)); migrations gate boot ([checkPendingMigrations](../../server/index.ts#L51-L60)); graceful shutdown drains connections in order ([ShutdownHelper usage L223–243](../../server/index.ts#L223-L243)); rollback = previous image + expand/contract discipline so old code tolerates new schema ([data page](../03-architecture-and-patterns/02-data-model-and-persistence.md)).

## "How would I know this broke?" per key flow

| Flow | Breakage | Detection today | Gap (judgment) |
| --- | --- | --- | --- |
| Document update (Flow 1) | API errors | Sentry + 5xx rates | fine |
| Realtime propagation | silent staleness | ...largely user reports (*possible gap*: no end-to-end "event emitted → client received" metric) | a synthetic canary (emit + subscribed probe) would close it |
| Revisions | processor failures | Bull failed-job counts, worker error logs | alert threshold needed on failed jobs |
| Notifications/email | silent non-delivery | queue depth + failed jobs; SMTP errors in logs | dead-letter visibility; "emails sent" counter to trend |
| Search | slow/no results | SearchQuery duration rows (query them!) | no alerting on p95 unless wired externally |
| Imports/exports | stuck operations | FileOperation states + ErrorTimedOut* watchdogs surface failure *to the user* | good — the strongest pattern here |
| Collaboration | lost edits/split brain | Hocuspocus logs, connection metrics | hardest to observe; user-visible symptoms first (*investigate*: MetricsExtension in [server/collaboration](../../server/collaboration/MetricsExtension.ts)) |

The FileOperation pattern deserves special attention: async work whose *status is a first-class row the user can see* (state machine + timeout watchdog + failure email — [ExportFailureEmail](../../server/emails/templates/ExportFailureEmail.tsx)). That's observability designed into the product, not bolted on — the strongest transferable idea on this page.

## Operational habits to practice here

1. Run with `DEBUG=http` and watch request logs; then `SERVICES=web` only and watch a worker-dependent feature *not* happen — feel the service split.
2. Open bull-board (admin service) during a burst of edits; watch queues fill and drain.
3. Kill Redis mid-session: what breaks loudly (health check, queues) vs silently (event enqueue post-commit — the [risk register](../09-reference/risk-register.md) entry made visceral)?
4. Read one `traceFunction` wrapper and find its span in local APM output (or just its name in code) — knowing *span naming* is half of using APM well.

## Interview angle

1. *"How do you monitor background jobs?"* — queue depth + failed counts + watchdog sweepers + user-visible state rows; the FileOperation story.
2. *"A user says the app is slow — what do you look at first?"* — traces by endpoint → DB vs CPU split → the [performance hunt](04-performance-thinking.md).
3. *"How do you make deploys safe?"* — health checks, graceful drain, migration gating, version-skew tolerance ([debugging Scenario 5](03-systematic-debugging.md)).
4. *"Logs vs metrics vs traces?"* — answer with one concrete example of each *from this repo* rather than definitions.

## Drill

Pick the realtime-propagation gap: design (one paragraph) the canary — what emits, what listens, what metric, what alert threshold, what false-positive risk. Compare with how you'd detect the same thing purely from client-side Sentry breadcrumbs. Both designs fit on an index card; being able to produce them on demand is an on-call interview superpower.

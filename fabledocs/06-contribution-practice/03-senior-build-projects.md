# Senior Build Projects

Six projects, 2 days–4 weeks each. The senior bar isn't code volume — it's the **plan**: problem framing, alternatives rejected with reasons, migration/rollout/rollback, test and security strategy, and open questions surfaced *before* they become incidents. Write the full plan (1–2 pages) before any code; the plan is gradeable on its own.

Template per project: Problem → Product value → Design checklist → Likely files → Migration plan → Test plan → Security plan → Performance plan → Rollout/rollback → Open questions → Stretch.

---

## SP-1: True outbox for the event bus (1 week)
Problem: post-commit enqueue can silently drop events ([Event.ts L82–97](../../server/models/Event.ts#L82-L97); [critique risk 1](../03-architecture-and-patterns/06-architecture-critique.md)).
Product value: webhooks/notifications/search stop being best-effort; self-hosters with flaky Redis get correctness.
Design checklist: `publishedAt` column on events; enqueue marks published; relay CronTask sweeps unpublished older than N sec; dedupe on the consumer side (jobId = event id); metrics on relay pickups (each pickup = a previously-lost event!).
Likely files: [Event.ts](../../server/models/Event.ts), new task in [server/queues/tasks](../../server/queues/tasks), migration, [worker.ts](../../server/services/worker.ts) (jobId), tests.
Migration: additive column, backfill `publishedAt = createdAt` for history.
Test plan: kill-between-commit-and-add simulation (mock queue.add to throw); relay picks it up; double-delivery tolerated by consumers (prove with the run-twice suite).
Security: none new. Performance: relay query needs a partial index (`WHERE "publishedAt" IS NULL`).
Rollout: relay behind env flag; watch pickup metric — should be ~0 in healthy systems.
Rollback: disable flag; column stays (harmless).
Open questions: `Event.schedule` events (never persisted) — accept their losability explicitly, or persist them too?
Interview story potential: **the best one in this curriculum** — a classic distributed-systems pattern applied surgically, with a metric proving the bug existed.

## SP-2: Presenter→client contract types (3–5 days)
Problem: no compile-time link between presenter outputs and client models ([critique risk 5](../03-architecture-and-patterns/06-architecture-critique.md)).
Design checklist: shared `shared/apiTypes.ts` (or per-presenter return types exported); presenters annotated with explicit return types; client models consume the types for `updateData` inputs; drift = `tsc` failure.
Likely files: [server/presenters/*](../../server/presenters), [shared/types.ts](../../shared/types.ts), [app/models/*](../../app/models).
Migration: one presenter per PR, document first.
Test plan: `yarn tsc` is the test; keep snapshots as the runtime witness.
Open questions: apiVersion-dependent shapes (`data` vs `text` — [presenter L38–68](../../server/presenters/document.ts#L38-L68)) need union types per version — does the team want that complexity?
Interview story potential: "made an implicit cross-runtime contract compiler-checked, incrementally."

## SP-3: End-to-end realtime canary + resync (1–2 weeks, pairs with MT-10)
Problem: silent websocket staleness is undetectable today ([observability gaps](../05-quality-engineering/06-observability-and-operations.md)).
Design checklist: server-side canary emitter (synthetic event every N sec per shard); client ack sampling; metric + alert; reconnect resync with jitter.
Likely files: [websockets service](../../server/services/websockets.ts), [WebsocketProvider](../../app/components/WebsocketProvider.tsx), Metrics.
Test plan: integration with a dropped/blocked socket; chaos-test the deploy scenario (mass reconnect).
Performance plan: canary volume vs teams count; sample, don't blanket.
Rollout: metric-only first (observe), alert later, resync last.
Interview story potential: "designed detection before mitigation for a distributed staleness bug."

## SP-4: Search provider: implement a second backend (2–4 weeks)
Problem/value: prove (and pressure-test) the provider seam ([SearchProviderManager](../../server/utils/SearchProviderManager.ts)) with e.g. Meilisearch (self-hosting-friendly); the [SearchIndexProcessor](../../server/queues/processors/SearchIndexProcessor.ts) events are already emitted.
Design checklist: provider interface conformance; index schema with permission data (collectionId, teamId as filterable); backfill task with checkpointing; dual-read comparison mode (log result diffs, serve Postgres).
Migration: backfill → shadow-read → cutover per team → keep tsvector as fallback.
Security plan: **permission filtering must move into the index query** — the provider contract's sharpest edge; a bug here is cross-tenant leakage. Adversarial tests mandatory.
Rollback: provider selection is env-config; instant.
Open questions: index freshness SLO; deletes (tombstones vs sync sweeps).
Interview story potential: full system-design-in-practice — backfill/shadow/cutover is *the* migration vocabulary interviews want.

## SP-5: Granular API key scopes UI + enforcement audit (1 week)
Problem: scopes exist ([ApiKey.canAccess](../../server/middlewares/authentication.ts#L222-L224)) — audit their granularity, enforcement coverage, and management UX; close gaps.
Design checklist: enumerate scope semantics; verify every route class enforces (write the matrix); settings UI for scoped key creation ([app/scenes/Settings](../../app/scenes/Settings)); docs of record.
Test plan: matrix-driven route tests (scope × endpoint grid, generated).
Security plan: this *is* one. Least-privilege by default for new keys.
Interview story potential: "audited and hardened an auth surface with a coverage matrix" — security depth signal.

## SP-6: Import pipeline resilience (2 weeks)
Problem: imports are long multi-stage jobs ([ImportsProcessor](../../server/queues/processors/ImportsProcessor.ts), [APIImportTask](../../server/queues/tasks/APIImportTask.ts), ImportTask chunk rows); harden against partial failure: resumability, per-chunk retry isolation, poison-chunk quarantine, user-visible per-stage progress.
Design checklist: chunk state machine review; idempotent chunk processing (safe re-run proof per stage); quarantine + skip-and-report; progress presenter.
Test plan: inject failures at each stage; assert resume-not-restart; run-twice suite per stage.
Performance plan: chunk size tuning with measurements.
Rollback: pipeline versioning — in-flight imports finish on old path (job payloads carry version).
Open questions: partially-imported trees — delete-on-abandon after how long?
Interview story potential: batch-processing reliability — rich STAR material for "tell me about a hard bug/robustness project."

---

Grading a project plan (before code): does it name the *riskiest assumption* and test it first? does rollout have an observe-only phase? does rollback actually restore invariants (not just code)? does one open question get escalated rather than decided? Senior work is distinguished by what it *refuses to decide alone*.

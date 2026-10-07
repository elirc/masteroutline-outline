# Performance Thinking

Rule zero: **measure first**. This repo already collects the measurements — knowing where they are is half the skill: search durations in SearchQuery rows, statsd metrics via [Metrics](../../server/logging/Metrics.ts) (hot-shots), dd-trace spans (`traceFunction` wrappers on presenters/workers — [presenters/document.ts L134–136](../../server/presenters/document.ts#L134-L136)), Bull queue depths ([HealthMonitor](../../server/queues/HealthMonitor.ts)), and the relative-ci bundle tracking ([relativeci.config.js](../../relativeci.config.js)).

## The performance domains here, with likely hotspots

### 1. Database (usually the real one)
- **N+1s:** the batch presenter exists because a per-document FileOperation lookup was one ([presentDocuments L147–176](../../server/presenters/document.ts#L147-L176)). Hunt technique: enable Sequelize logging locally, hit a list endpoint, count queries — anything O(rows) is a finding. The `separate: true` includes in [Document scopes](../../server/models/Document.ts#L179-L228) are *deliberate* extra queries (one per association, not per row) — know the difference.
- **Heavy columns:** document `state` (Yjs binary) — excluded by default via `withoutState` scope ([Document.ts L129–148](../../server/models/Document.ts#L129-L148)). The transferable pattern: exclude blob-ish columns unless requested; a "SELECT *" on documents is a perf bug here.
- **Missing/unused indexes:** migrations show the history of adding them; verify with `EXPLAIN`. The multi-tenant shape means almost every index should lead with `teamId` or an id column.
- **Unbounded queries:** everything user-facing goes through [pagination()](../../server/routes/api/middlewares/pagination.ts) — a route without it returning arrays is a review flag. Note the count-skipping optimization for client requests (L21–47): COUNT on big tables is itself a cost.

### 2. Server CPU (Node's special vulnerability)
Single thread ⇒ sync hotspots are outages, not slowdowns. Known heavy paths: ProseMirror JSON↔Markdown/HTML conversions ([DocumentHelper](../../server/models/helpers/DocumentHelper.tsx)), export zips, email rendering. The repo's mitigation: push them to workers (ExportTasks, EmailsProcessor). Review rule: new sync transforms on request paths need a size bound or a queue.

### 3. Realtime fan-out
Cost scales with (edits × subscribers × open tabs). The debounce processor and the per-room emission model are the mitigations; the metric to watch is websockets queue depth and delivery latency.

### 4. Client render
- MobX gives fine-grained re-rendering nearly for free ([framework models](../02-stack-and-language-mastery/02-framework-mental-models.md)) — the classic Reacty "everything re-renders" problems are rarer here; the bugs are the *opposite* (missing `observer`).
- Long lists: virtualization deps are present (`react-window`, `@tanstack/react-virtual` — [package.json](../../package.json)); any non-virtualized unbounded list is a candidate finding.
- Editor: ProseMirror keystroke path is performance-sacred; plugins run per-transaction. Never put non-memoized heavy work in editor plugin `apply`/decoration code ([shared/editor/plugins](../../shared/editor/plugins)).

### 5. Bundle
Route-level code splitting via lazy routes ([app/routes](../../app/routes/index.tsx)); heavyweights like mermaid/katex should load on demand (verify: grep how mermaid is imported — static import in a shared chunk would be a finding). CI tracks bundle deltas (relative-ci) — perf as a *reviewed contract*.

### 6. Caching
Redis-backed [CacheHelper](../../server/utils/CacheHelper.ts) (e.g. unfurl caching, cleared in dev at boot — [server/index.ts L67–70](../../server/index.ts#L63-L71)). Cache rules to interview with: every cache entry needs a key-derivation story and an invalidation story; this repo prefixes keys via [RedisPrefixHelper](../../server/utils/RedisPrefixHelper.ts).

## A worked hunt: "documents.list feels slow"

1. Reproduce with realistic data volume (factories in a script — build 5k docs).
2. Sequelize logging on; hit endpoint; capture query list. Expected: 1 main + a few `separate` includes; finding = any per-row query.
3. `EXPLAIN ANALYZE` the main query with real params; check index use on the `teamId + collectionId + deletedAt/archivedAt` shape.
4. Time the presenter separately (wrap in `console.time` locally) — JSON conversion cost shows here.
5. Only now change code; re-measure; put both numbers in the PR.

Steps 1–4 produce the *evidence sentence* every perf PR needs: "p50 X→Y ms because Z, measured by W."

## Interview angle

1. *"How do you find an N+1?"* — logging + count, then the batch-map fix (point at presentDocuments).
2. *"A page got slow — walk me through."* — the worked hunt, in order, measurement before code.
3. *"How do you keep bundles small?"* — route splitting + CI budget tracking.
4. *"Caching strategy?"* — Redis helper + key/invalidation discipline + "the client entity cache is also a cache with an invalidation story (websockets)".

## Drill

Actually run the worked hunt's steps 1–3 locally against `documents.list` (any data volume). Write the evidence sentence even if the verdict is "fine as-is" — practicing the *negative* result honestly is part of the skill.

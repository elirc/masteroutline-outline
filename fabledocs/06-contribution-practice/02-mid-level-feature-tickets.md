# Mid-Level Feature Tickets

Ten cross-layer tickets. The mid-level bar: **design notes before code** (5–10 sentences: approach, alternatives, risk, rollback), changes landing safely across schema/API/client/tests, and honest scoping. Each ticket lists the layer path and its riskiest step. Times assume familiarity from modules 01–05.

Required per ticket before implementation: (1) design note, (2) list of files you *expect* to touch, (3) the test list. After: compare expectation vs reality — the delta is your calibration score.

---

## MT-1: Per-user "recently deleted by me" view in Trash
Layers: API (extend `documents.deleted` with a `userId` filter) → client store → Trash scene → tests.
Anchors: [documents.deleted L412+](../../server/routes/api/documents/documents.ts#L411-L468), [app/scenes/Trash](../../app/scenes/Trash).
Risk: filter must not widen visibility (only *your* deletions of docs you can still see). Rollback: params are additive; revert = ignore param.
Interview story potential: "extended a public API additively with permission-safe filtering."

## MT-2: Document word/reading-time count, live in the UI
Layers: schema (column or computed-on-read decision — *make the argument*) → update pipeline ([DocumentUpdateTextTask](../../server/queues/tasks/DocumentUpdateTextTask.ts) is the natural writer) → presenter → client model → Document scene header.
Risk: the dual write path (REST + collaborative) must both maintain it — this ticket *is* [critique risk 2](../03-architecture-and-patterns/06-architecture-critique.md) in miniature; the event-driven task placement sidesteps it (both paths emit events).
Rollback: additive column; feature-flag the UI.
Interview story potential: strong — "I chose event-driven derivation over save-hooks because two write paths existed; here's the failure I avoided."

## MT-3: `collections.duplicate` endpoint
Layers: new route + schema → command (compose existing [documentDuplicator](../../server/commands/documentDuplicator.ts) recursively) → policies (`duplicate` on Collection?) → client action → tests.
Risk: large collections = long transaction; consider a FileOperation-style async path above a size threshold — *the design note must address this*.
Rejection realism: maintainers may want it async-only; propose both with the cutoff.
Interview story potential: "designed a sync/async hybrid for an expensive operation."

## MT-4: Rate-limit audit + gap fixes
Layers: inventory every mutating route's limiter (script it: grep route definitions, join with `RateLimiterStrategy` uses) → classify by cost → add limiters to the top 3 gaps → tests ([rateLimiter.test.ts](../../server/middlewares/rateLimiter.test.ts) patterns).
Risk: over-limiting real users (bulk operations from the UI); pick strategies matching existing analogues.
Interview story potential: "systematically audited an API surface for abuse protection" — excellent security story.

## MT-5: Notification preference: "digest instead of immediate" (design-heavy)
Layers: schema (user notification settings) → [NotificationsProcessor](../../server/queues/processors/NotificationsProcessor.ts) honoring the preference → a digest CronTask batching pending notifications → email template → settings UI.
Risk: the batching task's idempotency (double-send on retry is the failure users scream about); dedupe via a `digestedAt` marker.
Interview story potential: strong async-design story with an idempotency centerpiece.

## MT-6: Export a single document *tree* as markdown zip from the UI
Layers: verify server support ([ExportDocumentTreeTask](../../server/queues/tasks/ExportDocumentTreeTask.ts) exists!) → wire an API path if missing → client action in the document menu → FileOperation progress UI reuse.
Risk: mostly integration; the lesson is *discovering how much already exists* before writing anything — a mid-level superpower.
Interview story potential: "shipped a feature mostly by composing existing infrastructure."

## MT-7: Comment edit history ("edited" indicator with previous versions)
Layers: schema (comment revisions table or JSONB history — argue it) → comments.update writing history → presenter → UI indicator + popover.
Risk: unbounded history growth (retention rule needed); privacy (deleted text lingering in history — policy decision, escalate don't assume).
Interview story potential: schema-design tradeoff story (table vs JSONB) with retention thinking.

## MT-8: Slow-query telemetry for search zero-result queries
Layers: SearchQuery model already stores queries/durations → add a `resultCount` usage view for admins (route + simple scene or just an API) → tests.
Risk: PII in queries shown to admins — is that acceptable? (Team-scoped admins seeing team queries: probably yes, but *write the reasoning*.)
Interview story potential: "turned existing telemetry into an actionable product surface."

## MT-9: Harden `documents.import` limits
Layers: read [documentImporter](../../server/commands/documentImporter.ts) + multipart middleware → enforce file-size/type limits consistently → friendly errors → tests with crafted files.
Risk: breaking legitimate large imports; make limits env-configurable ([env.ts](../../server/env.ts) pattern).
Interview story potential: security-hardening story pairing with the [security checklist drill](../05-quality-engineering/05-security-checklist.md).

## MT-10: Websocket reconnect resync (the critique's investigate-item, made real)
Layers: client — on reconnect, refetch entities changed since disconnect (needs a "changed since" API or ETag-ish strategy) → [WebsocketProvider](../../app/components/WebsocketProvider.tsx) reconnect hook → tests.
Risk: thundering herd on mass reconnects (deploy!) — jitter + scope the refetch to visible entities.
Interview story potential: excellent — distributed-consistency reasoning on a real client cache; pairs with [critique](../03-architecture-and-patterns/06-architecture-critique.md) risk 4. **Do the investigation first: the mechanism may partially exist.**

---

Grading (per ticket): design note predicted the hard part? expected-vs-actual file list within ±3? tests written at the right layer? one sentence on "what I'd do differently"? Full rubric: [09-reference/learning-rubrics.md](../09-reference/learning-rubrics.md).

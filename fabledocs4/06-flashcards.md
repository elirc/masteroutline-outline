# 06 — Flashcards: Retention for fabledocs2 + fabledocs3

Format: **Q** front / **A** back. Import into Anki (one card per Q/A pair)
or self-quiz. Cards are grouped by topic; shuffle when reviewing. Answers are
deliberately short — if you can't expand any answer to 60 spoken seconds,
that's a signal to reread the source doc (cited per group).

Cadence that works: all cards once, then daily misses only, then every few
days. ~50 cards ≈ 25 minutes cold, <10 warm.

---

## Architecture (fabledocs2:01, ADR-001/002)

**Q:** How does one build artifact serve both self-hosters and cloud?
**A:** `server/index.ts` parses `--services=` / `SERVICES`; each service in `server/services/` attaches to a shared Koa app. Self-host runs all in one process; cloud runs specialized pods from the same image.

**Q:** Why does collaboration without `REDIS_COLLABORATION_URL` clamp to one web process?
**A:** Y.Doc state is process-local; two processes editing one doc without Redis sync would fork the document. The clamp guards a data-integrity invariant — quietly, which the ADR flags as a design regret.

**Q:** What's the top-level layering split, and what's it split *by*?
**A:** `app` / `server` / `shared` / `plugins` — split by **runtime**, not feature. `shared` runs in browser and Node; the editor schema lives there because both sides must agree exactly.

**Q:** How is `shared/`'s dual-runtime property enforced?
**A:** Vitest runs the shared project twice — `shared-node` and `shared-jsdom`. A server-only import fails the jsdom run (where a test imports it — coverage gaps are the escape route).

**Q:** Why are plugins compile-time and in-tree rather than runtime-loadable?
**A:** A plugin that can touch models can corrupt data; sandboxing Node wasn't feature-sized. So plugins are trusted code registered via typed hooks (`PluginManager.ts`, `Hook` enum), and core never imports one by name.

**Q:** Name the three service-boundary consumers of the event spine.
**A:** WebsocketsProcessor (broadcasts), NotificationsProcessor→Emails (notifications), SearchIndexProcessor (indexing) — plus revisions, webhooks, backlinks. All consume events; none are called inline from handlers.

## API design (fabledocs2:02, ADR-003/004)

**Q:** Why RPC-over-POST instead of REST?
**A:** The domain is verb-heavy (move, restore, duplicate) and maps badly to resource semantics. `noun.verb` names operations directly — trivially rate-limitable, auditable, and greppable. Cost: no HTTP caching, over-fetching presenters.

**Q:** What's the fixed middleware pipeline order for an API mutation?
**A:** Rate limit → authenticate → validate (Zod → `ctx.input`) → transaction → handler → presenter. Response: `{ data, policies?, pagination? }`.

**Q:** What's the known wart in `validate.ts`'s error behavior?
**A:** It throws only the *first* Zod issue per request — API consumers fix payloads one error at a time. Unfixed because changing it changes the error contract.

**Q:** Why did GraphQL lose?
**A:** Per-object conditional authz makes resolver field-leaks too easy; "POST this JSON" is a better integration contract than a query language; per-operation caching/limits are simpler with named RPCs.

**Q:** What's the "shipped policies" trick and what bug class does it kill?
**A:** The same CanCan ability evaluation that gates the API is serialized into responses (`policies` key) and drives UI affordances (`app/models/Policy.ts`). Frontend/backend permission drift becomes structurally impossible for covered actions.

**Q:** Why was Postgres RLS rejected for authorization?
**A:** Debugging RLS through ORM-generated SQL is brutal, and self-hosters would blame their own DB config for permission bugs. Residual risk of the choice: list-endpoint WHERE clauses must stay manually aligned with policies.

**Q:** In CanCan, what can a condition return besides a boolean, and why is it confusing?
**A:** A string — a denial *reason*. The `boolean | string` union in `cancan.ts` reads as truthy-string = allowed if you don't know the convention; the ADR calls it the layer's chief newcomer trap.

## Data access (fabledocs2:03, ADR-005/007)

**Q:** Name the document's three representations and their authority order.
**A:** `state` (binary Yjs CRDT — authoritative), `content` (Prosemirror JSON projection), `text` (markdown projection). Projections refresh async (`DocumentUpdateTextTask`) and may be stale.

**Q:** How do legacy documents get Yjs state?
**A:** Lazily: first collaborative open, `PersistenceExtension.onLoadDocument` builds the Y.Doc from content/text under `SELECT … FOR UPDATE`. Rejected big-bang conversion: a bug would corrupt rarely-opened docs invisibly for months.

**Q:** Why explicit transaction-passing instead of AsyncLocalStorage?
**A:** Lost async context makes CLS fail *silently into autocommit* — worst failure mode for a correctness mechanism. Explicit passing is ugly but fails visibly in review. Cost: `transaction` on every command signature, forgettable without type-level enforcement.

**Q:** Why must the Event insert share the mutation's transaction?
**A:** The event *is* downstream's worldview — websockets, search, webhooks, revisions. A half-commit (mutation without event) starves them silently; nothing looks wrong in the DB.

**Q:** What is `SET LOCAL lock_timeout = '15s'` in `documentCollaborativeUpdater` and why LOCAL?
**A:** A scar from a lock-convoy/pool-exhaustion incident: bound each waiter so one doc's slow persist fails that save, not the server. LOCAL scopes it to the transaction so it can't leak onto the pooled connection's next borrower.

**Q:** What does the `withoutState` scope prevent?
**A:** Accidentally loading multi-megabyte CRDT blobs when you only need metadata — a memory-incident-in-waiting encoded as a data-layer default.

**Q:** Why is debounced persistence a durability gap, and what mitigates it?
**A:** Hard crash loses unflushed Y.Doc changes since last persist. Mitigation: clients hold their own CRDT copy and resend on reconnect — safe by CRDT merge math, but dependent on a client surviving to reconnect.

## Distributed systems (ADR-006/008)

**Q:** Describe the event spine's delivery semantics precisely.
**A:** Transactional event row (exactly-once record) → post-commit in-process publish (**at-most-once into** the queue — the gap) → Bull retries (**at-least-once from** the queue). Consumers must be idempotent; the gap has no relay.

**Q:** What visible remediation proves the events table's unbounded growth was unplanned?
**A:** `CleanupOldEventsTask.ts` — cleanup-by-cron added later, not original design. (General reconstruction method: decisions leave scars; scars are evidence.)

**Q:** List Redis's five hats.
**A:** socket.io fan-out adapter, Bull queues, distributed locks + cache (`CacheHelper`), rate-limit counters, optional collaboration state sync. One `REDIS_URL`, five coupled failure modes.

**Q:** Why keys the rate limiter on user id when possible, IP otherwise?
**A:** Offices behind one NAT share an IP — per-user keying stops one office from consuming a shared bucket. Token→userId resolution is cached in Redis to keep the middleware cheap.

**Q:** Why not Postgres LISTEN/NOTIFY for fan-out?
**A:** Payload limits, and every web process holding listener connections adds pressure on Postgres — already the most-contended resource given the CRDT row-lock workload. Protect Postgres; spend Redis.

**Q:** Why CRDTs over OT?
**A:** OT needs an always-authoritative sequencer ordering ops; CRDTs merge without one — tolerating offline edits, multi-tab, reconnect-resend. Mature Yjs + y-prosemirror bindings sealed it.

## Error handling (fabledocs2:05, ADR-009)

**Q:** What three things does every error in `server/errors.ts` carry, and who consumes each?
**A:** HTTP status (transport), stable string `id` (clients branch on it; messages stay free to change/translate), `isReportable` (the onerror funnel routes true → Sentry, false → just traffic).

**Q:** Why is `isReportable` set at the construction site?
**A:** The author knows whether the error is traffic or a bug; a generic catch handler guessing from status codes knows the least. A 500 from flaky S3 and a 500 from a null deref need different routing.

**Q:** What's the crying-wolf failure mode of the reportable bit?
**A:** One frequent error mislabeled reportable floods Sentry into quota rate-limiting — blinding you exactly when a real regression fires. One boolean, huge blast radius; hence the rule "reportable = an engineer should *act*."

**Q:** Why did Result types lose to exceptions?
**A:** Koa's middleware model *is* exception propagation to `onerror`; Result-typed internals in an exception-shaped shell means lossy conversion at every boundary. Accepted cost: TS can't express what a function throws.

**Q:** What's the graceful-shutdown ordering and what dictates it?
**A:** Stop intake (stoppable) → flush collaboration state → drain queues → close Redis/Postgres (`ShutdownHelper`, `ShutdownOrder`). Dictated by ADR-007's debounce gap: kill the DB before the flush and a deploy eats edits.

## Testing (fabledocs2:04, ADR-010)

**Q:** State the server testing doctrine in one sentence.
**A:** Integration tests through the real HTTP stack (`TestServer` boots actual Koa) against real Postgres, factories for data, MSW-faked outbound HTTP as the only mocked seam.

**Q:** Why real Postgres instead of SQLite or a mocked ORM?
**A:** The code leans on Postgres-only behavior (FOR UPDATE, SET LOCAL, tsvector, paranoid scopes) — SQLite-green proves nothing; ORM mocks encode *beliefs* about the ORM, which the paranoid-scope upgrade regression showed can silently diverge.

**Q:** What's the mandatory convention in authorization tests?
**A:** Build a second team and assert the cross-team actor gets 403. Cross-tenant leakage is the least-affordable bug class in a multi-tenant-by-column schema.

**Q:** Why not rollback-per-test isolation?
**A:** Code under test manages its own transactions (ADR-005); wrapping them turns them into savepoints with different semantics. Isolation is by cleanup convention — occasionally leaky, accepted cost.

**Q:** What's the testing strategy's standing IOU?
**A:** No automated multi-client end-to-end harness for collaborative editing — the highest-risk subsystem has the thinnest system-level coverage; pieces tested, composition rests on CRDT guarantees + staged rollout.

## Observability (fabledocs2:06)

**Q:** Four instruments, four questions — match them.
**A:** Category-tagged logs (what happened, where), StatsD metrics (how much, trending), dd-trace spans (where did time go), Sentry (what must a human fix — reportable errors only).

**Q:** Why is `import "./logging/tracer"` first in `server/index.ts`?
**A:** dd-trace instruments by patching module internals at require time — it only sees modules loaded *after* it. Reordering imports silently degrades tracing; the comment is load-bearing.

**Q:** Why does `Metrics.ts` swallow StatsD errors, and what mitigates the cost?
**A:** Observability must never be load-bearing — a metrics outage can't become an app outage (self-hosters first). Cost: can't distinguish zero from broken; mitigated by a heartbeat metric whose flatline means "pipe dead, not app quiet."

**Q:** Why do Hocuspocus extensions duplicate logging/metrics/auth?
**A:** Websocket sessions never pass through Koa middleware — the collaboration path needs parallel seams (`AuthenticationExtension`, `LoggerExtension`, `MetricsExtension`, `ConnectionLimitExtension`).

**Q:** Why a closed `LogCategory` union instead of free-form logger names?
**A:** ~15 deliberate categories keep logs queryable ("filter `multiplayer` at 14:32") and runbooks stable, vs. hundreds of ad hoc names. Cost: uneven granularity, and rebalancing a closed union touches every call site.

## Meta / method

**Q:** How do you reverse-engineer ADRs from a codebase with no docs?
**A:** Hunt **paid costs** — code that exists only because a decision made it necessary. Cleanup tasks imply unbounded growth; lock timeouts imply convoys; runtime clamps imply in-process state. Decisions leave scars; scars are evidence.

**Q:** What four-beat structure makes an architecture answer sound senior?
**A:** Constraint → decision → a rejected alternative *and why* → the cost knowingly accepted. Volunteering the cost is the seniority signal.

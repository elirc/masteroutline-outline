# Pattern Catalog

Sixteen patterns this codebase actually uses, as recognition cards. The goal is not to name-drop — it's to recognize the *shape* in any repo and know when it's the wrong shape. Each card ends with the interview question it answers.

---

## Pattern 1: RPC-over-POST API

Problem it solves: REST's resource/verb mapping gets awkward for operations like "move", "restore", "empty trash". Naming the operation directly keeps the API self-describing.
General shape: `POST /api/<noun>.<verb>` with a JSON body; GET is not used for reads either.
Real example: the ~30 endpoints in [documents.ts](../../server/routes/api/documents/documents.ts) (`documents.move` L1355, `documents.empty_trash` L2047).
Second example: [comments.ts](../../server/routes/api/comments/comments.ts#L28) (`comments.create` … `comments.unresolve`).
Why this implementation works: uniform middleware chain per route; verbs never need to be shoehorned into PUT/PATCH semantics; the whole API is greppable by string.
Failure modes: loses HTTP caching and semantic status-code conventions; POST-for-read means CSRF protection must cover reads too (it does — [csrf.ts](../../server/middlewares/csrf.ts)).
Use it when: internal/product APIs with many operations. Avoid it when: you need third-party cacheable REST semantics or strict hypermedia.
Interview angle: "REST vs RPC — when would you choose each?" Answer with this repo as the RPC case and its tradeoffs.
Drill: find one endpoint whose verb would be genuinely painful in REST (candidate: `documents.templatize` L1196).

## Pattern 2: Middleware chain as execution contract

Problem it solves: every endpoint needs the same cross-cutting steps in the same order.
General shape: `router.post(name, auth(), pagination(), rateLimiter(...), validate(schema), transaction(), handler)`.
Real example: [documents.search L1034–1040](../../server/routes/api/documents/documents.ts#L1034-L1040) — all five in one signature.
Second example: [documents.update L1251–1255](../../server/routes/api/documents/documents.ts#L1251-L1255).
Why this implementation works: the route signature *is* the documentation: you can read authentication requirements, rate limits, and transactional behavior without opening the handler.
Failure modes: order mistakes (validate before auth leaks validation info to unauthenticated callers; transaction around slow external calls holds DB connections). Missing `transaction()` on a multi-write handler = partial writes.
Use it when: any Koa/Express app. Avoid when: a step isn't truly cross-cutting.
Interview angle: "Where do validation/authz/transactions live in your API?"
Drill: grep three routes with `transaction()` and one mutating route without it; decide whether the omission is safe (single write?) or a latent bug.

## Pattern 3: Colocated Zod request contracts

Problem it solves: request shape drift and hand-rolled validation.
General shape: per-route-dir `schema.ts` exporting `XxxSchema` + inferred `XxxReq` types, enforced by one middleware.
Real example: [documents/schema.ts](../../server/routes/api/documents/schema.ts) consumed via `validate(T.DocumentsUpdateSchema)` at [documents.ts L1253](../../server/routes/api/documents/documents.ts#L1251-L1255); handler reads typed `ctx.input.body`.
Second example: [comments/schema.ts](../../server/routes/api/comments/schema.ts).
Why this implementation works: one source of truth produces runtime validation *and* compile-time types (`z.infer`) — the **contract** is executable.
Failure modes: schemas that only check types, not semantics (lengths, enums, cross-field rules) give false confidence; the first-issue-only error message in [validate.ts L18–24](../../server/middlewares/validate.ts#L17-L25) hides multi-field errors.
Use it when: always, at every trust boundary. Avoid when: never — but don't duplicate schemas per layer.
Interview angle: "How do you validate input in TypeScript so the types don't lie?"
Drill: find a schema using `.refine()` for a cross-field rule (grep `refine` in the schema files).

## Pattern 4: CanCan policies + serialized abilities

Problem it solves: authorization rules scattered through handlers become unauditable.
General shape: declarative `allow(Actor, action, Target, condition)` registry; `authorize()` throws; `serialize()` exports the decision table to the client.
Real example: engine [cancan.ts](../../server/policies/cancan.ts#L36-L120); rules [document.ts](../../server/policies/document.ts); client consumption via [PoliciesStore](../../app/stores/PoliciesStore.ts) + [usePolicy](../../app/hooks/usePolicy.ts).
Second example: [collection.ts policy](../../server/policies/collection.ts).
Why this implementation works: policies compose (`update` calls `can(actor,"read")`); the UI never re-implements rules, it renders `policy.abilities.update ? <EditButton/> : null` — one brain, two consumers.
Failure modes: client policies are *advisory only* — forgetting the server-side `authorize()` while trusting the hidden button is an IDOR. Policies expect preloaded associations and blow up (by design) otherwise — see the invariant at [document.ts L283–290](../../server/policies/document.ts#L275-L308).
Use it when: >2 roles or per-resource permissions. Avoid when: trivial single-role apps (overhead).
Interview angle: "How do you keep frontend and backend permission checks in sync?" This is the strongest architecture answer in the repo.
Drill: pick `archive`; list every condition path that grants it, from [document.ts L217–228](../../server/policies/document.ts#L217-L228) down through `update` and `read`.

## Pattern 5: Scope-preloaded authorization data

Problem it solves: policies need memberships, but per-check queries would be N+1 and async.
General shape: the data layer preloads permission-relevant associations (`withMembership` scope); policies are then synchronous and pure.
Real example: [Document scopes L179–228](../../server/models/Document.ts#L179-L228) + [findByPk override](../../server/models/Document.ts#L758-L833) that auto-applies them when `userId` is passed.
Second example: `Collection.findByPk(collectionId, { userId })` at [documents.ts L150–155](../../server/routes/api/documents/documents.ts#L148-L156).
Why this implementation works: splits "fetch what authz needs" (async, batched, one query) from "decide" (sync, testable). The `invariant` crash for missing preloads converts silent permission bugs into loud dev-time errors.
Failure modes: forgetting `userId` in a new call site → runtime invariant failure (good) or, worse, a policy path that treats missing memberships as "no access" silently (check which invariants exist per model).
Use it when: policy checks need relational data. Avoid when: rules are attribute-only.
Interview angle: "How do you avoid N+1 queries in permission checks?"
Drill: trace what SQL `Document.findByPk(id, {userId})` actually issues (turn on Sequelize logging or reason from the scope) — count the queries.

## Pattern 6: Command layer for multi-model mutations

Problem it solves: fat routes and fat models; operations spanning several models need one named home.
General shape: `server/commands/<thing><Verb>er.ts` — a function taking `ctx` + props, using `ctx.state.transaction`, returning the mutated aggregate.
Real example: [documentMover.ts](../../server/commands/documentMover.ts) (collection structure + document + memberships in one txn).
Second example: [documentUpdater.ts](../../server/commands/documentUpdater.ts), [accountProvisioner.ts](../../server/commands/accountProvisioner.ts).
Why this implementation works: routes stay thin (parse → authorize → command → present); commands are unit-testable without HTTP (their `.test.ts` files prove it).
Failure modes: commands calling commands can double-emit events; unclear ownership between model methods and commands ("where does `publish` live?" — answer here: on the model, called by commands).
Use it when: mutation touches >1 model or has branching business rules. Avoid when: a single `model.update` suffices — indirection tax.
Interview angle: "Where does business logic live in your codebase?" — layered answer: model invariants on models, orchestration in commands, HTTP in routes.
Drill: read [documentMover.ts](../../server/commands/documentMover.ts) and list every table it can write.

## Pattern 7: Presenter layer (response DTOs)

Problem it solves: leaking ORM objects to the wire couples clients to the schema and leaks fields.
General shape: `present<Model>(ctx, model, options) → plain object`; composed in handlers.
Real example: [presenters/document.ts](../../server/presenters/document.ts) — including its security role (public field withholding L97–129).
Second example: [presenters/user.ts](../../server/presenters/user.ts) (has a test + snapshot).
Why this implementation works: schema changes stay invisible until deliberately exposed; batch variants ([presentDocuments L147–176](../../server/presenters/document.ts#L147-L176)) fix N+1s at the layer that caused them.
Failure modes: presenters drifting from client model expectations (no shared type enforcement between presenter output and [app/models](../../app/models) — *possible risk*, see critique); "just spread the model" shortcuts.
Use it when: any API with external consumers. Avoid when: internal admin tooling where speed > stability.
Interview angle: "How do you evolve an API without breaking clients?"
Drill: diff the fields of [presenters/document.ts](../../server/presenters/document.ts) against [app/models/Document.ts](../../app/models/Document.ts) observables — find one field present in only one side and explain why.

## Pattern 8: Transactional event emission (outbox-flavored)

Problem it solves: "write DB then publish message" is a two-phase problem; publishing before commit announces data that may roll back.
General shape: event creation participates in the row's transaction; queue publish happens in `afterCommit`.
Real example: [Event.ts `@AfterSave` L82–97](../../server/models/Event.ts#L82-L97) — note the explicit `transaction.parent` handling for nested transactions.
Second example: every `saveWithCtx`-created event (see [documentUpdater L110–118](../../server/commands/documentUpdater.ts#L110-L125)).
Why this implementation works: consumers never observe uncommitted state; the audit row and the domain row are atomic.
Failure modes: this is *not* a full outbox — if the process dies after commit but before the Redis `add`, the event is lost (no relay re-reads the events table). Low probability, real consequence: missed notifications/webhooks. **Possible risk**, logged in the [risk register](../09-reference/risk-register.md).
Use it when: any DB-plus-message write. Avoid when: you need guaranteed delivery — then use a true outbox with a relay.
Interview angle: "How do you keep your queue consistent with your database?" — being able to say "afterCommit hook, and here's the gap a real outbox would close" is a senior-level answer.
Drill: write the 5-line pseudocode of a true outbox relay for this schema (the `events` table already exists — what's missing is a `publishedAt` column and a poller).

## Pattern 9: Event-bus fan-out with per-consumer jobs

Problem it solves: one event, many independent reactions, isolated failures.
General shape: global queue → for each processor with matching `applicableEvents`, enqueue a *separate* job; each retries independently.
Real example: [worker.ts L53–88](../../server/services/worker.ts#L53-L88); subscriptions like [NotificationsProcessor L28–40](../../server/queues/processors/NotificationsProcessor.ts#L28-L41).
Second example: the websockets special case (dedicated queue consumed only by the websockets service, [worker.ts L63–66](../../server/services/worker.ts#L63-L66)).
Why this implementation works: a failing webhook delivery retries without re-running notifications; `shouldQueue` ([BaseProcessor L11–20](../../server/queues/processors/BaseProcessor.ts#L11-L20)) lets processors cheaply skip irrelevant events.
Failure modes: no ordering guarantees across processors; retries require **idempotent** `perform`; wildcard subscribers (`"*"`) pay for every event.
Use it when: side effects multiply. Avoid when: strict ordering or exactly-once is required.
Interview angle: "Design the 'after someone publishes a doc' side effects" — this is the reference architecture.
Drill: pick `documents.publish` and enumerate every processor that reacts (grep `applicableEvents` for it).

## Pattern 10: Processor/Task split

Problem it solves: "react to events" and "run a job with args" are different contracts.
General shape: processors subscribe; tasks are invoked (`new XTask().schedule(props)`) with priority/retry options.
Real example: [BaseTask](../../server/queues/tasks/base/BaseTask.ts) (default: 5 attempts, exponential backoff, priorities); [NotificationsProcessor](../../server/queues/processors/NotificationsProcessor.ts) scheduling per-notification tasks.
Second example: cron-style tasks extend [CronTask](../../server/queues/tasks/base/CronTask.ts) (cleanup family: [CleanupDeletedDocumentsTask](../../server/queues/tasks/CleanupDeletedDocumentsTask.ts) etc.).
Why this implementation works: tasks are the retry-able unit of work; processors stay thin routers. Priorities keep interactive-ish work (emails) ahead of housekeeping.
Failure modes: heavy logic in processors makes retries expensive and non-granular; task props must be JSON-serializable and *small* (ids, not models).
Interview angle: "How do you structure background jobs?"
Drill: find a task that overrides `options` for higher priority or fewer attempts (grep `get options` under server/queues/tasks).

## Pattern 11: Soft delete + scheduled hard delete

Problem it solves: users delete things by accident; compliance wants real deletion eventually.
General shape: paranoid models set `deletedAt`; cron tasks purge after a retention window.
Real example: Document trash → [CleanupDeletedDocumentsTask](../../server/queues/tasks/CleanupDeletedDocumentsTask.ts) + [documentPermanentDeleter](../../server/commands/documentPermanentDeleter.ts).
Second example: teams — [CleanupDeletedTeamsTask](../../server/queues/tasks/CleanupDeletedTeamsTask.ts).
Why this implementation works: undo becomes an update (`documents.restore`, [documents.ts L956](../../server/routes/api/documents/documents.ts#L955-L980)); purge is centralized where cascade rules (attachments, memberships) can be handled deliberately.
Failure modes: forgetting `paranoid: false` when you *need* deleted rows (restore flows — note `paranoid` threading through [Document scopes](../../server/models/Document.ts#L734-L748)); unique constraints colliding with soft-deleted rows; queries that forget the default scope excludes deleted.
Interview angle: "How do you implement trash/undo?" plus GDPR-flavored follow-ups.
Drill: find where `documents.restore` authorization differs from `update` ([policy L192–206](../../server/policies/document.ts#L192-L206)) and why (`deletedAt` docs fail `isActive`).

## Pattern 12: Fractional indexing for user-ordered lists

Problem it solves: reordering items without renumbering the whole list (O(1) writes, concurrent-insert friendly).
General shape: order keys are strings; inserting between "a" and "b" generates a key between them.
Real example: `fractionalIndex` import in [documents.ts L2](../../server/routes/api/documents/documents.ts#L1-L11), used for ordering (grep `fractionalIndex(` there and in [starCreator.ts](../../server/commands/starCreator.ts)).
Second example: [pinCreator.ts](../../server/commands/pinCreator.ts) (*inferred from the pattern's use for pins/stars; verify with grep*).
Why this implementation works: no transactionally-renumbered siblings; two clients inserting concurrently rarely conflict.
Failure modes: key growth after many midpoint insertions (needs occasional rebalancing); string collation assumptions.
Interview angle: "How would you implement drag-to-reorder persistently?" — the expected modern answer is exactly this.
Drill: compute by hand what key falls between "a1" and "a2" using the library's README rules, then find a rebalancing story in this repo (is there one? that's the interesting part — *investigate*).

## Pattern 13: Observable domain cache (client)

Problem it solves: SPA needs a single source of truth for entities across scenes, updated by both fetches and pushes.
General shape: per-entity store holding `Map<id, Model>` observables; components observe; all writes funnel through `store.add`.
Real example: [Store base class](../../app/stores/base/Store.ts#L56-L172) with `add`/`remove`; [RootStore](../../app/stores/RootStore.ts) wiring.
Second example: websocket events writing into the same cache ([WebsocketProvider L253–262](../../app/components/WebsocketProvider.tsx#L253-L262)).
Why this implementation works: fetch, mutation response, and websocket push all converge on `add` → the UI can't disagree with itself about an entity.
Failure modes: cache staleness policy is implicit (no TTL); memory growth in long sessions; `remove`'s client-side cascade ([L174–221](../../app/stores/base/Store.ts#L174-L221)) must mirror server FK behavior or ghosts appear.
Interview angle: "MobX/Redux/React-Query — how do you keep server state consistent client-side?" Compare this hand-rolled normalized cache with React Query's request-keyed cache: normalized entity cache wins when websocket pushes update entities used by many views.
Drill: find how a deleted collection removes its documents client-side (the `Relation` decorator's `onDelete` — [app/models/decorators/Relation.ts](../../app/models/decorators/Relation.ts)).

## Pattern 14: Server-computed UI permissions

Problem it solves: duplicated permission logic drifting between client and server.
General shape: API responses carry `policies: [{id, abilities}]`; UI checks abilities, server re-checks on action.
Real example: `presentPolicies(user, [document])` in every documents response ([documents.ts L1289–1292](../../server/routes/api/documents/documents.ts#L1289-L1293)); [usePolicy hook](../../app/hooks/usePolicy.ts).
Second example: [cancan serialize L127–155](../../server/policies/cancan.ts#L127-L155) generating the ability map.
Why this implementation works: one rule engine; the client is a *renderer* of decisions, not a decider.
Failure modes: policies staleness after state changes (mitigated: responses re-ship policies after mutations); treating client policy as security (it's UX only).
Interview angle: "How does your frontend know what buttons to show?"
Drill: in the UI, find one component branching on `can.update` (grep `usePolicy` under app/) and trace the ability back to its `allow()` rule.

## Pattern 15: Plugin architecture with hook registration

Problem it solves: integrations (auth providers, storage, webhooks, embeds) shouldn't touch core files.
General shape: each plugin self-registers against typed hooks; core iterates registered hooks (`PluginManager.getHooks(Hook.X)`).
Real example: [server/utils/PluginManager.ts](../../server/utils/PluginManager.ts), loaded at boot ([server/index.ts L65](../../server/index.ts#L63-L66)); API routes mounted from plugins in [routes/api/index.ts](../../server/routes/api/index.ts) (see the `PluginManager`/`Hook` imports); auth providers enumerated via `AuthenticationHelper.providers` in [routes/auth/index.ts](../../server/routes/auth/index.ts#L19-L35).
Second example: [plugins/webhooks/server](../../plugins/webhooks/server) contributing processors/tasks without core edits.
Why this implementation works: adding Discord auth = adding a directory; core surface stays closed to modification, open to extension.
Failure modes: hidden coupling if plugins reach into core internals; load order assumptions; plugin config validation happening late.
Interview angle: "How would you make your app extensible for integrations?"
Drill: pick [plugins/slack](../../plugins/slack) and list which hooks it registers (grep `PluginManager.add`).

## Pattern 16: Strategy-based rate limiting

Problem it solves: uniform global limits either throttle real users or under-protect expensive endpoints.
General shape: a default limiter app-wide + named per-route strategies.
Real example: [defaultRateLimiter](../../server/middlewares/rateLimiter.ts#L57) applied globally at [server/index.ts L98](../../server/index.ts#L97-L99); `rateLimiter(RateLimiterStrategy.OneHundredPerMinute)` on search ([documents.ts L1038](../../server/routes/api/documents/documents.ts#L1034-L1040)), `TwentyFivePerMinute` on comment creation.
Second example: auth endpoints (grep `RateLimiterStrategy` under server/routes/auth and plugins).
Why this implementation works: cost-proportional limits, Redis-backed so they hold across processes.
Failure modes: keying (per-IP vs per-user) mismatches NAT/corporate networks; forgetting limits on new expensive endpoints (review checklist item).
Interview angle: "How do you protect expensive endpoints?"
Drill: grep all `RateLimiterStrategy.` uses; bucket them by endpoint cost and check the mapping makes sense — file any mismatch as a candidate contribution.

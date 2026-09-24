# File Reading Order

Thirty files, sequenced so each one makes the next easier. Per file: why it matters, what to look for, what to skip. Don't read linearly top-to-bottom inside big files — read the parts named.

**Junior path:** files 1–14. **Mid path:** 1–24. **Senior path:** all, plus the "senior noticing" notes.

## Foundation (everyone)

1. [docs/ARCHITECTURE.md](../../docs/ARCHITECTURE.md) — the official map. Look for: the directory-to-responsibility table. Skip: nothing, it's short.
2. [package.json](../../package.json#L6-L39) — scripts are the repo's verbs. Look for: `dev:watch`, `test`, `db:migrate`, `build`. Skip: the dependency list on first pass.
3. [server/index.ts](../../server/index.ts) — bootstrap. Look for: `master()` vs `start()`, the services loop (L182–190), graceful shutdown via `ShutdownHelper` (L223–243). Skip: the `redirectOnClient` HTML helper.
4. [server/routes/api/index.ts](../../server/routes/api/index.ts) — how every API router is mounted, and the shared middleware stack (body parsing, request context, CSRF verification, API response shaping). Look for: middleware order. *Senior noticing:* order is a contract — CSRF must run after body parsing, auth after context.
5. [server/routes/api/documents/documents.ts](../../server/routes/api/documents/documents.ts) — the biggest route file (2106 lines). Read only: `documents.list` (L102), `documents.info` (L574), `documents.update` (L1251). Look for: the identical middleware chain shape on every route. Skip: export/import endpoints for now.
6. [server/routes/api/documents/schema.ts](../../server/routes/api/documents/schema.ts) — Zod request contracts. Look for: how schemas compose (`BaseSchema`, refinements). *Interview vocabulary:* this file is where **validation** lives — shape checking — as opposed to **authorization**.
7. [server/middlewares/validate.ts](../../server/middlewares/validate.ts) + [transaction.ts](../../server/middlewares/transaction.ts) — 50 lines total that define the API's execution model.
8. [server/middlewares/authentication.ts](../../server/middlewares/authentication.ts) — three token types (OAuth access token, API key, JWT cookie), one middleware. Look for: `parseAuthentication` transport rules (L84–133) — API keys may not come via cookie (L200–203), OAuth tokens only via header (L156–161). *Senior noticing:* those transport restrictions are CSRF defense.
9. [server/policies/cancan.ts](../../server/policies/cancan.ts) — the authorization engine (~300 lines, vendored & adapted). Look for: `allow`, `can`, `authorize`, and `serialize` (L127–155) which ships permissions to the client. Then [server/policies/document.ts](../../server/policies/document.ts) — real rules. Look for: how `update` builds on `read` (L86–102).
10. [server/models/Document.ts](../../server/models/Document.ts) — the central model (1408 lines). Read only: the `@Scopes` block (L129+) and the `findByPk` override (L758–833). Look for: `withMembership` — permission data is *preloaded by scope*, then *checked by policy*. This two-step is the repo's most important idiom.
11. [server/commands/documentUpdater.ts](../../server/commands/documentUpdater.ts) — a complete command: mutate, save with context, emit event. Look for: the `changed()` check (L100) and the three save paths (publish / changed / done-only).
12. [server/models/Event.ts](../../server/models/Event.ts) — Look for: `@AfterSave enqueue` (L82–97) with its `afterCommit` handling, and `Event.schedule` (L140–148) for events that skip the audit table.
13. [server/services/worker.ts](../../server/services/worker.ts) — the async backbone. Look for: three `process()` blocks (global events → per-processor jobs → tasks) and the websockets special case (L63–66).
14. [server/presenters/document.ts](../../server/presenters/document.ts) — the response contract. Look for: the `isPublic` branches (fields withheld from anonymous viewers, L97–129). *Senior noticing:* presenters are also a security layer.

## Client (mid path continues)

15. [app/index.tsx](../../app/index.tsx) + [app/routes/index.tsx](../../app/routes/index.tsx) — SPA entry and lazy-loaded route chunks.
16. [app/stores/RootStore.ts](../../app/stores/RootStore.ts) — every store registered in one graph. Look for: how stores get a back-reference for cross-store updates.
17. [app/stores/base/Store.ts](../../app/stores/base/Store.ts) — generic CRUD store. Read: `add` (L152–172), `remove` and its cascade logic (L174–221). *Senior noticing:* `remove` re-implements FK `ON DELETE` semantics client-side via the `Relation` decorator — the client cache mirrors database concepts.
18. [app/models/base/Model.ts](../../app/models/base/Model.ts) — client model with `save`/`updateData` and dirty tracking.
19. [app/stores/DocumentsStore.ts](../../app/stores/DocumentsStore.ts) — the largest store; skim the fetch methods and note they all end in `this.add(...)` → the single cache write path.
20. [app/utils/ApiClient.ts](../../app/utils/ApiClient.ts) — Look for: in-flight request dedupe (L62–64), CSRF cookie handling, error mapping to typed errors.
21. [app/components/WebsocketProvider.tsx](../../app/components/WebsocketProvider.tsx) — Look for: the `join` re-emit handshake (L126–128) and entity handlers merging events into stores (L253+).
22. [app/scenes/Document/index.tsx](../../app/scenes/Document/index.tsx) and its `components/` — how a scene composes stores + editor.
23. [app/hooks/usePolicy.ts](../../app/hooks/usePolicy.ts) + [app/models/Policy.ts](../../app/models/Policy.ts) — server-computed abilities driving UI affordances. Pair with `serialize` from file 9.
24. [shared/editor/nodes](../../shared/editor/nodes) — pick one node (e.g. a heading or checkbox) and read its `toDOM`/`parseDOM`/command. Skip the rest; the editor is a subsystem you learn lazily.

## Senior path extras

25. [server/collaboration/PersistenceExtension.ts](../../server/collaboration/PersistenceExtension.ts) — where Yjs state meets Postgres. *Senior noticing:* two write paths exist for document content (REST `documentUpdater` vs collaborative updates) — reconciling dual write paths is a classic source of subtle bugs; see the risk register.
26. [server/queues/processors/RevisionsProcessor.ts](../../server/queues/processors/RevisionsProcessor.ts) — debounced revision creation; note the Redis collaborator-set handoff (L26–29) and the idempotency guard comparing to the previous revision (L36–44).
27. [server/queues/processors/NotificationsProcessor.ts](../../server/queues/processors/NotificationsProcessor.ts) — event → per-notification tasks; the fan-out-of-fan-out pattern.
28. [server/middlewares/rateLimiter.ts](../../server/middlewares/rateLimiter.ts) + [csrf.ts](../../server/middlewares/csrf.ts) — cross-cutting defenses. Look for: which strategies exist and which routes opt in (grep `rateLimiter(RateLimiterStrategy`).
29. [server/test/factories.ts](../../server/test/factories.ts) + [server/test/TestServer.ts](../../server/test/TestServer.ts) + [vitest.config.ts](../../vitest.config.ts) — the testing machinery: four vitest projects, real Postgres for server tests.
30. [server/env.ts](../../server/env.ts) + [shared/env.ts](../../shared/env.ts) — validated configuration. *Senior noticing:* config validation at boot converts runtime mysteries into startup failures — cheap reliability.

## Pause-and-predict exercises

Before opening each numbered file, predict: (a) what it imports, (b) what exports it exposes, (c) one invariant it maintains. Write your predictions down; score yourself after reading. Being wrong is the point — mispredictions reveal your model's gaps.

Self-grade — **Basic:** completed 1–14 and can state each file's job in one sentence. **Solid:** 1–24 and can answer "where would I add X?" for a new field, a new endpoint, a new background job. **Strong:** all 30, plus you disagree with something (e.g. route file size, dual write paths) and can argue it.

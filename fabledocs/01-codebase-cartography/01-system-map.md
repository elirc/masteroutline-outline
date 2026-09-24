# System Map

## What kind of project is this?

A single-repo, multi-service TypeScript application (not a workspace monorepo — one `package.json`, one dependency graph, path aliases instead of packages). One build produces one artifact that can run any subset of six services, selected by the `SERVICES` env var and started in a loop in [server/index.ts](../../server/index.ts#L182-L190). This "modular monolith" shape is worth naming in interviews: it gets most of the operational flexibility of microservices (scale workers separately from web) with none of the distributed-systems tax (one schema, one deploy, in-process calls).

```
                        ┌────────────────────────────────────────────────┐
                        │                 one Node build                 │
   browser ──HTTP──────▶│  web (Koa API + static)                        │
   browser ──WS /realtime──▶ websockets (socket.io, Redis adapter)       │──▶ PostgreSQL
   browser ──WS /collaboration─▶ collaboration (Hocuspocus/Yjs)          │──▶ Redis (cache,
                        │  worker (Bull queues: events, tasks)           │     queues, pub/sub)
                        │  cron (scheduled tasks)   admin (bull-board)   │──▶ S3-compatible storage
                        └────────────────────────────────────────────────┘
```

Service entry points: [server/services/web.ts](../../server/services/web.ts), [worker.ts](../../server/services/worker.ts), [websockets.ts](../../server/services/websockets.ts), [collaboration.ts](../../server/services/collaboration.ts), [cron.ts](../../server/services/cron.ts), [admin.ts](../../server/services/admin.ts). Process management is [throng](../../server/index.ts#L265-L269) (cluster forking); note the constraint at [L39–48](../../server/index.ts#L39-L48): the collaboration service forces a single process unless a dedicated Redis is configured — a real example of a scaling constraint encoded in bootstrap code.

## Ownership map

| Area | Directory | Owns | Must NOT own |
| --- | --- | --- | --- |
| UI | [app/](../../app) | React components, MobX stores/models, routing, client cache | Authorization decisions (display-only via policies), persistence |
| API | [server/routes/api/](../../server/routes/api) | HTTP surface: auth middleware, Zod validation, calling commands/models, presenting | Business logic beyond orchestration (belongs in commands/models) |
| Domain | [server/commands/](../../server/commands) + [server/models/](../../server/models) | Multi-model operations, model invariants, scopes | HTTP concerns (ctx beyond `APIContext`), rendering |
| Authorization | [server/policies/](../../server/policies) | Every can/cannot decision | Data fetching (expects preloaded associations) |
| Data | [server/migrations/](../../server/migrations) (295 files) + Sequelize models | Schema, indexes | — |
| Async | [server/queues/](../../server/queues) | Processors (event-driven) and tasks (arbitrary jobs) | Anything that must be transactionally consistent with the request |
| Contracts | [server/routes/api/*/schema.ts](../../server/routes/api/documents/schema.ts) (request), [server/presenters/](../../server/presenters) (response), [shared/types.ts](../../shared/types.ts) | The client↔server wire shape | — |
| Editor | [shared/editor/](../../shared/editor) | ProseMirror schema, nodes/marks, commands, embeds | App state |
| Realtime | [server/services/websockets.ts](../../server/services/websockets.ts), [server/collaboration/](../../server/collaboration) | Fan-out + collaborative editing sessions | Business rules |
| Plugins | [plugins/](../../plugins) | Auth providers (google, oidc, azure…), integrations (slack, github, webhooks…), storage | Core flow changes |
| Emails | [server/emails/templates/](../../server/emails/templates) | Transactional email as React components (oy-vey) | Delivery decisions (processors decide) |

## Public interfaces vs. private internals

**Public (changing these is a contract change — high blast radius):**

- The RPC API: `POST /api/<noun>.<verb>` routes, e.g. the ~30 endpoints in [documents.ts](../../server/routes/api/documents/documents.ts). Used by the web app *and* third parties (it's Outline's documented public API) — you cannot break it casually.
- Presenter output shapes ([server/presenters/](../../server/presenters)) — the actual response contract.
- Auth endpoints under `/auth/` ([server/routes/auth/index.ts](../../server/routes/auth/index.ts)), OAuth provider surface ([server/routes/oauth/](../../server/routes/oauth)), MCP ([server/routes/mcp/](../../server/routes/mcp)).
- Webhook payloads ([plugins/webhooks/](../../plugins/webhooks)) — consumed by external systems.
- Websocket event names — consumed by every open browser tab, including old ones during a deploy.

**Private (refactor freely, respecting tests):** commands, processors/tasks, MobX store internals, editor internals, middleware order *within* reason.

A **boundary** in the senior sense is a line where the two sides can change independently as long as the contract holds. The presenter layer is Outline's clearest boundary: models can gain/rename columns without clients noticing, because presenters translate. Find the mirror image client-side: [app/models/Document.ts](../../app/models/Document.ts) declares observable fields matching the presenter output.

## Where things you'll search for actually live

| "Where is…" | Answer |
| --- | --- |
| Request validation | Zod schemas colocated per route dir, e.g. [documents/schema.ts](../../server/routes/api/documents/schema.ts), applied by [validate.ts](../../server/middlewares/validate.ts) |
| Authorization | [server/policies/](../../server/policies), enforced by `authorize()` from [cancan.ts](../../server/policies/cancan.ts#L182-L191) |
| Transactions | [transaction middleware](../../server/middlewares/transaction.ts) on mutating routes; commands receive `ctx.state.transaction` |
| The event bus | [Event model `@AfterSave`](../../server/models/Event.ts#L82-L97) → [queues](../../server/queues/index.ts) → [worker fan-out](../../server/services/worker.ts#L53-L88) |
| Client cache | [app/stores/base/Store.ts](../../app/stores/base/Store.ts) — observable `Map<id, Model>` per entity type, composed in [RootStore](../../app/stores/RootStore.ts) |
| Client↔server sync | [app/utils/ApiClient.ts](../../app/utils/ApiClient.ts) (fetch, CSRF, retries, dedupe) + [WebsocketProvider](../../app/components/WebsocketProvider.tsx) (server push) |
| Errors | [server/errors.ts](../../server/errors.ts) (custom classes) / [app/utils/errors.ts](../../app/utils/errors.ts) |
| Feature flags/prefs | Team preferences (see `TeamPreference` in [shared/types.ts](../../shared/types.ts)), checked inside policies like [comment](../../server/policies/document.ts#L53-L64) |
| Generated / vendored | [shared/i18n/locales](../../shared/i18n) (extracted strings — never hand-edit), [server/migrations](../../server/migrations) (append-only history — don't use old ones as style examples), `patches/` (patch-package) |

## Drill

Without opening this file again, draw the system map from memory (boxes: browser, web, websockets, collaboration, worker, cron; stores: Postgres, Redis, S3) and label which arrow carries (a) RPC calls, (b) entity push events, (c) Yjs sync, (d) queued jobs. Then verify against the diagram above.

Self-grade — **Basic:** 4+ boxes right. **Solid:** all arrows right, and you can say which service a slow "document edit isn't showing for my teammate" bug lives in. **Strong:** you can explain why websockets jobs get their own queue (only the websockets service may consume them — [worker.ts L63–66](../../server/services/worker.ts#L63-L66)) and what would happen if a worker consumed them instead.

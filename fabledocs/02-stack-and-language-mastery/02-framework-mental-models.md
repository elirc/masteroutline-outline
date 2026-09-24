# Framework Mental Models

One paragraph of "how it thinks" per framework, then where this repo uses it well and where the sharp edges are.

## React 17 + MobX 4 (the UI pair)

**How it thinks:** React re-renders components whose inputs changed; MobX makes "inputs" automatic — components wrapped in `observer` re-render when any observable they *dereferenced during render* changes. State lives in stores (observable Maps of models); components are projections. There's no reducer/dispatch ceremony: you mutate observables inside `@action`s and the dependency graph handles the rest.

Where used well: the normalized entity cache ([Store.ts](../../app/stores/base/Store.ts#L56-L60)) means a websocket update to one document re-renders exactly the components showing that document — surgical updates with zero memo bookkeeping. `updateData` merging into the *same instance* ([add, L152–172](../../app/stores/base/Store.ts#L152-L172)) preserves identity so observers stay attached.

Sharp edges to check in this repo:
- **Dereference timing:** destructuring an observable early (`const { title } = doc`) copies the value — later changes won't re-render. Access properties *in render*.
- **`observer` coverage:** a parent passing `doc.title` down as a prop makes the *parent* the observer; missing `observer` on the component that dereferences = stale UI (Scenario 1 in [systematic debugging](../05-quality-engineering/03-systematic-debugging.md)).
- **MobX 4 (not 6):** decorator syntax, no Proxies — new properties added to plain objects aren't tracked; models declare all observables upfront ([app/models](../../app/models)).
- **Hooks + MobX:** `useEffect` deps don't understand observables; effects reacting to observable changes need `autorun`/`reaction` or explicit dereference in deps. Stale-closure bugs live here.
- React 17 = no concurrent features, no `useSyncExternalStore`; and `render`-phase behavior differs from 18 docs you'll read online. Know your version's reality.

Contrast to articulate in interviews: MobX's *push-based fine-grained reactivity* vs Redux's *pull-based snapshot + selector memoization* vs React Query's *server-cache-first*. This repo is the strongest possible argument for MobX's model: server-pushed entity updates fan out to precisely the right components.

## Koa (the HTTP spine)

**How it thinks:** an onion of `async (ctx, next)` functions; everything is composition. There's no DI container, no controllers — a route *is* its middleware chain. `ctx.state` is the per-request scratchpad (`auth`, `transaction`, `pagination`), typed via [server/types.ts](../../server/types.ts)'s `APIContext`.

Used well: the uniform pipeline ([Pattern 2](../03-architecture-and-patterns/05-pattern-catalog.md)); errors as exceptions caught centrally ([onerror.ts](../../server/onerror.ts)).
Sharp edges: middleware *order* is unchecked by types (auth-after-validate compiles fine); anything forgetting `await next()` silently truncates the chain; `ctx.state` is a typed-by-assertion bag — the types promise what middleware must deliver.

## Sequelize + sequelize-typescript (the ORM)

**How it thinks:** classes as tables via decorators; instances are stateful (dirty-tracking via `changed()` — used to decide save paths in [documentUpdater L100](../../server/commands/documentUpdater.ts#L100-L118)); **scopes** are named, composable query fragments; **hooks** attach behavior to lifecycle (the event system hangs off `@AfterSave` — [Event.ts](../../server/models/Event.ts#L82-L97)).

Used well: scopes as authorization-data loaders ([withMembership](../../server/models/Document.ts#L179-L228)); custom `findByPk` accepting slugs ([L758–833](../../server/models/Document.ts#L758-L833)).
Sharp edges: forgetting to pass `transaction` (silent auto-commit — the repo's biggest footgun); paranoid-mode surprises (default scope hides deleted rows — flows that need them pass `paranoid: false`, [RevisionsProcessor L31](../../server/queues/processors/RevisionsProcessor.ts#L25-L33)); `separate: true` needed to dodge Sequelize include bugs (the comment at [Document.ts L207–210](../../server/models/Document.ts#L204-L215) cites the issue) — when the ORM's abstraction leaks, this repo documents the leak inline; instance vs plain-object confusion (presenters exist partly to force the conversion).

## socket.io + Yjs/Hocuspocus (the realtime pair)

**How they think:** socket.io = rooms + events over a websocket with reconnection sugar; the Redis adapter ([websockets.ts](../../server/services/websockets.ts#L1-L22)) makes rooms span processes. Yjs = a CRDT — every client merges every edit deterministically regardless of order; the server ([server/collaboration](../../server/collaboration)) relays updates and persists snapshots; "conflict" stops being a concept, at the cost of an opaque binary state and memory-resident docs.

Sharp edges: room membership is the whole ballgame for entity events (join handshake — [WebsocketProvider L126–128](../../app/components/WebsocketProvider.tsx#L126-L128)); auth on connect ([websockets.ts](../../server/services/websockets.ts) uses the same JWT machinery); CRDT state vs JSON `content` duality (see [architecture critique](../03-architecture-and-patterns/06-architecture-critique.md), risk 2).

## Interview angle

Full cards in [08-interview-prep/02-frontend-framework-questions.md](../08-interview-prep/02-frontend-framework-questions.md) and [03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md):
1. "How does MobX know what to re-render?" — dereference tracking, with the store/observer pair here.
2. "Compare your state management to Redux/React Query."
3. "Explain Koa middleware vs Express" — onion composition, error flow.
4. "What has your ORM done to hurt you?" — transaction threading, include bugs, paranoid scoping (have the `separate: true` comment ready).
5. "OT vs CRDT for collaborative editing?" — Yjs choice + persistence model.

## Drill

Find one `observer` component under [app/components](../../app/components) and one non-observer presentational component. Explain for each *why* its wrapping status is correct. Then find one Sequelize call site inside a transactional route and verify it threads `transaction` — if you find one that doesn't, you may have found a real bug: file it per [maintainer communication](../07-career-and-collaboration/03-maintainer-communication.md).

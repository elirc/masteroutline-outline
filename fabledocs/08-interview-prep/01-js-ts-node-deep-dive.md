# JS / TS / Node Deep-Dive Question Cards

Fourteen cards. Format: what interviewers ask → what they're testing → the repo code you can cite → junior/mid/senior answer contrast. Practice: answer aloud in ≤90 seconds *citing the anchor*, then read the follow-ups.

---

## Q1: "Explain the event loop. When does an `afterCommit` callback actually run?"
Round: JS/TS deep-dive
What it's really testing: do you understand that async ordering is about *queues of callbacks*, not threads.
Repo anchor: [server/models/Event.ts#L82-L97](../../server/models/Event.ts#L82-L97) — `@AfterSave` registers `afterCommit`; the Bull `add` is `void`-ed (fire-and-forget promise).
Junior answer sounds like: "JavaScript is single-threaded with a callback queue."
Mid-level answer adds: microtasks (promises) vs macrotasks (timers, I/O); `afterCommit` runs when Sequelize resolves the COMMIT's I/O, i.e., in a promise continuation — after the current stack, potentially interleaved with other requests' work.
Senior answer includes: the `void` means enqueue failures are unobserved by the request (invariant: HTTP success ≠ event delivered); single-threadedness means a slow synchronous presenter blocks *every* concurrent request — concurrency in Node is cooperative.
Likely follow-ups: "What happens to an unhandled rejection here?" (see the process-level handler in [server/index.ts#L245-L250](../../server/index.ts#L245-L250) — logged, not crashed).
Practice drill: trace on paper the ordering: response sent vs event enqueued vs websocket received.

## Q2: "Serial vs parallel async — show me a real case where parallelizing matters."
Round: JS/TS deep-dive
Repo anchor: [server/middlewares/authentication.ts#L43-L46](../../server/middlewares/authentication.ts#L43-L46) — `Promise.all([user.updateActiveAt(ctx), user.team?.updateActiveAt()])`; also [pagination's parallel find+count](../../server/routes/api/middlewares/pagination.ts#L50-L55).
What it's really testing: whether you reach for `Promise.all` when awaits are independent, and know when you must not (dependent data, transaction ordering, connection pool pressure).
Junior: "await both."  Mid: independent awaits → `Promise.all`, saves a round trip per request on a hot path (auth runs on *every* request). Senior: failure semantics — `Promise.all` rejects fast and can leave the other promise running; `allSettled` when partial success is acceptable; in transactions, parallel queries on one connection serialize anyway.
Follow-ups: "`Promise.all` vs `allSettled` vs `race`?" "Why might parallel DB calls in one transaction not speed anything up?"
Drill: find one more `Promise.all` in server/ (grep) and judge whether it's safe.

## Q3: "What are closures, and show me a closure bug you could imagine in this codebase."
Round: JS/TS deep-dive
Repo anchor (concept exercised): the memoized queue factories in [server/queues/index.ts](../../server/queues/index.ts) — module-level `let cached…` captured by the exported functions (a deliberate, correct closure/singleton).
Junior: definition ("function remembers its scope").
Mid: the singleton-by-closure pattern here, and the classic bug: capturing a loop variable or stale `props` in a React effect (cross-link [02-frontend-framework-questions.md](02-frontend-framework-questions.md) Q on stale closures).
Senior: closures as memory ownership — a long-lived closure (queue, socket handler) capturing a large object keeps it from GC; why the lazy factory (defer Redis connection until first use, testability).
Drill: explain why `globalEventQueue()` is a function call everywhere instead of a top-level `export const queue = createQueue(...)` (answer: import-time side effects — connecting to Redis on *import* breaks tests and non-worker services).

## Q4: "Walk me through error handling in a production Node API."
Round: JS/TS deep-dive
Repo anchor: [server/errors.ts](../../server/errors.ts) (typed error constructors like `ValidationError`, `AuthorizationError`), centralized [onerror](../../server/onerror.ts) + the API error middleware ([apiErrorHandler](../../server/routes/api/middlewares/apiErrorHandler.ts)); client mirror: [app/utils/errors.ts](../../app/utils/errors.ts) mapped in [ApiClient](../../app/utils/ApiClient.ts).
Junior: try/catch everywhere.
Mid: throw typed errors deep, translate once at the boundary to status codes; validation errors get 400 with a safe message ([validate.ts L17–25](../../server/middlewares/validate.ts#L17-L25)); never leak internals in messages.
Senior: error taxonomy = **contract** (clients switch on it — ApiClient maps statuses to typed client errors and routes 401s to a session handler, [ApiClient L43–47](../../app/utils/ApiClient.ts#L43-L47)); crash-vs-continue policy (unhandledRejection logged not fatal, [server/index.ts L245–250](../../server/index.ts#L245-L250)) and why workers differ from web (a poisoned job should fail the job, not the process).
Drill: follow one `AuthorizationError` from `authorize()` ([cancan.ts L182–191](../../server/policies/cancan.ts#L182-L191)) to the HTTP status and to the client error class.

## Q5: "How does TypeScript's inference help you here — and where does it stop helping?"
Round: JS/TS deep-dive
Repo anchor: `z.infer`-based request types (`T.DocumentsUpdateReq` used at [documents.ts L1255](../../server/routes/api/documents/documents.ts#L1251-L1255)); `authorize` as an assertion function — `asserts target` at [cancan.ts L182–187](../../server/policies/cancan.ts#L182-L191) narrows `Document | null` to `Document` after the call.
Junior: "TS catches typos."
Mid: runtime validators generating static types = the types can't drift from reality; assertion signatures (`asserts`) encode "this throws or narrows" so post-`authorize` code doesn't need null checks.
Senior: the boundaries where types *lie*: Sequelize scopes return loosely-typed models (the preloaded-membership invariant is enforced by runtime `invariant`, not types — [document policy L283–290](../../server/policies/document.ts#L275-L308)); `@ts-expect-error` sites mark known gaps ([cancan.ts L248](../../server/policies/cancan.ts#L243-L254)); generics variance forcing `any` in the `Constructor` type ([cancan.ts L5–6](../../server/policies/cancan.ts#L1-L10)).
Follow-ups: "`unknown` vs `any`?" "When is a type assertion acceptable?"
Drill: delete `asserts target` mentally — enumerate every call site that would now need a null check.

## Q6: "Unions and narrowing — give a production example."
Round: JS/TS deep-dive
Repo anchor: [authentication.ts](../../server/middlewares/authentication.ts#L135-L247) — the token branches to three auth types with different transports; `AuthTransport` union at L28.
Junior: `typeof`/`in` checks.
Mid: discriminated unions; the auth flow narrows by matcher functions (`OAuthAuthentication.match(token)`, `ApiKey.match(token)`) — narrowing by runtime predicate, encoded per branch.
Senior: unions as *exhaustiveness contracts* — adding a fourth token type should fail compilation somewhere; where it wouldn't here (if/else chains don't force exhaustiveness — a `switch` with `never` default would); tradeoff of stringly-typed event names (`Event["name"]` unions in [BaseProcessor](../../server/queues/processors/BaseProcessor.ts#L1-L9)) vs class-based events.
Drill: find the `Event` type in [server/types.ts](../../server/types.ts) and see how event payloads are discriminated by `name`.

## Q7: "What do decorators do, and what are the sharp edges?"
Round: JS/TS deep-dive
Repo anchor: sequelize-typescript models ([Event.ts](../../server/models/Event.ts) — `@Table`, `@Column`, `@AfterSave`); MobX 4 (`@observable`, `@action` in [Store.ts](../../app/stores/base/Store.ts#L56-L100)); the build implications documented in [vitest.config.ts](../../vitest.config.ts#L20-L41) comments (legacy decorators, `useDefineForClassFields: false`).
Junior: "annotations that add behavior."
Mid: decorators run at class-definition time and register metadata (columns, hooks) — the model *class* is a schema DSL; hooks like `@AfterSave` are global side effects attached to the ORM lifecycle.
Senior: the toolchain coupling — legacy vs TC39 decorators, class-fields semantics silently breaking MobX observables (that vitest comment is a war story in comment form), circular-model-graph issues with decorator metadata. Being able to explain *why the test config comment exists* is a genuinely senior flex.
Drill: read the comment block at [vitest.config.ts L20–25](../../vitest.config.ts#L20-L41) and restate it in your own words.

## Q8: "How do you prevent the same expensive request running twice?"
Round: JS/TS deep-dive
Repo anchor: in-flight request dedupe map in [ApiClient](../../app/utils/ApiClient.ts#L62-L64); server-side analog: rate-limit strategies ([Pattern 16](../03-architecture-and-patterns/05-pattern-catalog.md)); distributed analog: `redlock` dependency for cross-process locks.
Junior: disable the button.
Mid: promise-keyed dedupe — store the pending promise by request key, return it to all callers; explain why the *promise*, not the result, is cached.
Senior: idempotency keys server-side for true safety (client dedupe dies with the tab); the layered defense: UI state → client dedupe → rate limit → DB constraints as the final invariant.
Drill: describe what happens with two rapid identical `documents.info` calls vs two identical `documents.update` calls under this client.

## Q9: "Node clustering — how does this app use multiple cores?"
Round: JS/TS deep-dive / systems
Repo anchor: throng fork loop [server/index.ts#L265-L269](../../server/index.ts#L265-L269); `WEB_CONCURRENCY`; the collaboration single-process constraint [L39–48](../../server/index.ts#L39-L48).
Junior: "Node is single-threaded so you run more processes."
Mid: master/worker fork model; sticky state (websocket connections) is why socket.io needs the Redis adapter ([websockets service](../../server/services/websockets.ts)) so events reach sockets on other processes.
Senior: which state may live in-process (none, ideally) vs Redis/Postgres; the collaboration service is the exception — in-memory Yjs docs — hence 1 process unless a shared Redis coordinates; graceful shutdown draining (stoppable + ShutdownHelper, [L223–241](../../server/index.ts#L223-L243)).
Drill: explain what breaks if you set `WEB_CONCURRENCY=4` with collaboration enabled and no `REDIS_COLLABORATION_URL` (the code prevents it — say why).

## Q10: "JWT vs session cookies vs API keys — how would you support all three?"
Round: JS/TS deep-dive / API
Repo anchor: [authentication.ts](../../server/middlewares/authentication.ts) — one middleware, three token classes, transport restrictions (API key never via cookie L200–203; OAuth only via header L156–161).
Junior: defines each.
Mid: explains the transport rules as CSRF defense (cookies are auto-attached, so cookie-transported credentials need CSRF tokens; header-only tokens don't).
Senior: expiry/rotation stories per type; `updateActiveAt` on each use for auditability; scope enforcement (`canAccess`) for least privilege; the UserSuspended path returning *who suspended you* — product care in an auth path.
Drill: for each token type, answer: where stored, how revoked, CSRF-relevant?

## Q11: "What's a memory leak you'd actually look for in a Node service like this?"
Round: JS/TS deep-dive / debugging
Repo anchor: conceptual, with candidates: socket rooms and per-connection state in [websockets.ts](../../server/services/websockets.ts); the client-side unbounded `Store.data` maps ([Store.ts L57–58](../../app/stores/base/Store.ts#L56-L60)) for the browser variant.
Junior: "forgotten timers."
Mid: unremoved listeners on long-lived emitters (sockets), caches without eviction, closures capturing request-scoped data in module-level maps.
Senior: how you'd confirm — heap snapshots across load, `--inspect` (the dev script already passes it — [package.json L13](../../package.json#L13)), watching RSS per worker; and the mitigation pattern: worker recycling.
Drill: find `setInterval` uses in server/ (grep) and check each has `.unref()` or a shutdown hook (start with [server/index.ts L58](../../server/index.ts#L56-L59)).

## Q12: "Explain streams/backpressure with a real use."
Round: JS/TS deep-dive
Repo anchor: export zips streamed to the response — `streamZipResponse` ([server/utils/koa.ts](../../server/utils/koa.ts), used by [documents.ts](../../server/routes/api/documents/documents.ts#L88-L92) imports); S3 multipart via `@aws-sdk/lib-storage`.
Junior: "streams process data in chunks."
Mid: why: exporting a large collection can't buffer in memory; backpressure = the producer pauses when the consumer (socket) is slow.
Senior: failure modes — half-written zip on error (client sees truncated file; need trailer/error signaling), timeouts on slow consumers, and why exports beyond a size should move to background FileOperations + S3 + signed URL (which this repo *also* implements — the two-tier export design is a great interview story).
Drill: contrast the inline `documents.export` path with the FileOperation-based collection export ([collectionExporter.ts](../../server/commands/collectionExporter.ts), [ExportTask family](../../server/queues/tasks/ExportMarkdownZipTask.ts)).

## Q13: "How do you version and share types between client and server?"
Round: JS/TS deep-dive
Repo anchor: [shared/types.ts](../../shared/types.ts) (enums like `UserRole`, `DocumentPermission` used by both policy code and UI); path aliases in [tsconfig.json](../../tsconfig.json)/[vitest.config.ts](../../vitest.config.ts#L11-L16); editor version handshake — client sends `x-editor-version` ([documents.update L1259](../../server/routes/api/documents/documents.ts#L1255-L1260)), server can demand upgrades (`UpdateRequiredError` exists in [app/utils/errors.ts](../../app/utils/errors.ts) imports).
Junior: "put types in a shared folder."
Mid: enums shared so the wire values are typo-proof; isomorphism constraint (no Node APIs in shared/).
Senior: what's *not* shared — presenter output isn't type-checked against client models (drift risk, see [architecture critique](../03-architecture-and-patterns/06-architecture-critique.md)); version skew during deploys handled by explicit version headers, not hope.
Drill: pick `DocumentPermission` in shared/types.ts and find one server and one client consumer.

## Q14: "What happens between `yarn dev` and code actually running?"
Round: JS/TS deep-dive / tooling
Repo anchor: [package.json scripts L11–15](../../package.json#L11-L15) — server is *compiled to build/ and run* (nodemon rebuild loop), frontend is Vite dev server; SWC everywhere decorators are involved ([vitest.config.ts](../../vitest.config.ts#L26-L41)).
Junior: "the dev server runs my code."
Mid: two different pipelines (bundled SPA vs transpiled Node), why HMR applies to one and process-restart to the other.
Senior: build-tool constraints as architecture: legacy decorators forcing SWC over esbuild, `resolutions` pinning transitive deps ([package.json L372-L385](../../package.json#L372-L385)) as supply-chain/compat control.
Drill: explain why editing a server file takes seconds to reflect but an app file is instant.

---

Scoring any card: **Basic** = correct definition. **Solid** = definition + this repo's example + one failure mode. **Strong** = adds the alternative design and when the standard answer is wrong. Aim to deliver Solid consistently under time pressure; Strong on your three favorite cards.

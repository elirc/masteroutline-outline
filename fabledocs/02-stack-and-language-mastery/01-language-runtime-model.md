# Language and Runtime Model

## The event loop, using this server as the diagram

One Node process = one JS thread + libuv's I/O machinery. Every concurrent request in Outline's web service is interleaved callbacks on that one thread. Consequences you can point at:

- **Cooperative concurrency:** a synchronous hot spot (big ProseMirror JSON → Markdown conversion in a presenter, [DocumentHelper.toMarkdown](../../server/presenters/document.ts#L52-L55)) stalls *all* in-flight requests. This is why heavy transforms (exports) move to workers ([ExportTask family](../../server/queues/tasks/ExportMarkdownZipTask.ts)).
- **Microtasks vs macrotasks:** promise continuations (`afterCommit` callbacks, `.then` chains) run before timers/I/O callbacks queued later. You rarely need the fine detail — you always need "await points are where other requests' code runs," which is why shared mutable module state is dangerous and why this repo keeps per-request state on `ctx.state`.
- **Fire-and-forget with `void`:** [Event.ts L92–96](../../server/models/Event.ts#L82-L97) `void queue.add(...)` — deliberately unawaited; rejections vanish into the `unhandledRejection` logger ([server/index.ts L245–250](../../server/index.ts#L245-L250)). Every `void someAsync()` is a decision about failure visibility. Grep `void ` in server/ and judge three of them.
- **Parallelism where it counts:** independent awaits get `Promise.all` — auth's activity updates ([authentication.ts L43–46](../../server/middlewares/authentication.ts#L43-L46)), pagination's find+count ([pagination.ts L50–55](../../server/routes/api/middlewares/pagination.ts#L50-L55)). Serial-awaiting independent queries is the most common junior perf bug; this repo shows the correction.
- **Real parallelism = processes:** throng forks ([server/index.ts L265–269](../../server/index.ts#L265-L269)); shared state therefore lives in Redis/Postgres, never module scope. The one in-memory exception (collab sessions) visibly constrains process count ([L39–48](../../server/index.ts#L39-L48)).

## Closures as an engineering tool

The repo's queue accessors ([server/queues/index.ts](../../server/queues/index.ts)) are lazy singletons via module-scope `let` + closure — deferring Redis connections until first use, keeping imports side-effect-free (crucial for tests and for services that don't use queues). Contrast with the anti-pattern: `export const queue = createQueue(...)` connects at import time in every process that transitively imports it. "Imports must be side-effect-free" is a senior code-review rule you can now justify concretely.

## Error propagation model

Async errors travel the promise chain to the nearest `await`; Koa turns thrown errors in middleware into responses (central [onerror](../../server/onerror.ts) + [apiErrorHandler](../../server/routes/api/middlewares/apiErrorHandler.ts)). The repo's discipline: *throw typed errors anywhere* ([server/errors.ts](../../server/errors.ts)), *translate once* at the boundary. In workers there is no request to fail, so errors fail the *job* → retry machinery ([worker.ts L140–155](../../server/services/worker.ts#L140-L155)). Same language feature, two failure domains, two policies — that contrast is a strong interview riff.

## Node-specifics worth owning

- Graceful shutdown: SIGTERM → ordered hooks (stop accepting, drain, flush metrics) via [ShutdownHelper](../../server/utils/ShutdownHelper.ts) usage in [server/index.ts L223–254](../../server/index.ts#L223-L254). Kubernetes-era table stakes.
- `setInterval(...).unref()` ([L58](../../server/index.ts#L56-L59)) — timers that shouldn't keep the process alive. Tiny API, real ops consequence.
- Env-driven behavior: `NODE_ENV` splits appear as `env.isDevelopment/isProduction` guards (e.g. dev-only cache clearing [L67–70](../../server/index.ts#L63-L71)).

## Pitfall checklist (this stack)

- [ ] Unawaited promise where you needed the result or the error
- [ ] Independent awaits in series on a hot path
- [ ] Module-level mutable state touched by requests
- [ ] Long synchronous work in web-service code paths
- [ ] Import-time side effects (connections, timers)
- [ ] Missing `.unref()`/shutdown hook on intervals

## Interview angle

Questions this page arms you for — full cards in [08-interview-prep/01-js-ts-node-deep-dive.md](../08-interview-prep/01-js-ts-node-deep-dive.md):
1. "Explain the event loop / when do promise callbacks run?" (Q1)
2. "Serial vs parallel async — real example?" (Q2)
3. "How does Node use multiple cores, and what breaks?" (Q9)
4. "Show me a closure doing real work" (Q3)
5. "Production error-handling strategy?" (Q4)

## Drill

Take [worker.ts's global-event handler](../../server/services/worker.ts#L29-L93) and annotate every `await`: what runs concurrently while it waits, and what state could change under it. Self-grade: Strong = you identify that between fan-out awaits, *other jobs on the same worker* run, and explain why that's safe here (no shared mutable state; everything by value on the job).

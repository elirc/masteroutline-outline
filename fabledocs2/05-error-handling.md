# 05 — Error Handling: One Funnel, a Reportable Bit, and Dying Gracefully

## What it is and why it exists

Errors have exactly one vocabulary and one exit. The vocabulary is
`server/errors.ts`: factory functions (`AuthenticationError`,
`AuthorizationError`, `ValidationError`, `RateLimitExceededError`,
`InternalError`, ...) built on `http-errors`, each carrying three things — an
HTTP status, a stable machine-readable `id` (`"authorization_error"`,
`"rate_limit_exceeded"`), and an `isReportable` boolean. The exit is
`server/onerror.ts`, the Koa-level handler that catches everything thrown
anywhere in the request pipeline, shapes the JSON error response, and decides
— based on `isReportable` — whether this error is *telemetry* or *traffic*.

That last distinction is the whole design. A 403 from a policy check is
traffic: expected, frequent, boring, and if we paged on it we'd throw the
pager into the sea. A `TypeError` from a presenter is telemetry: a bug, ship
it to Sentry with context. Encoding that decision *at the error's
construction site* — where the author knows what it means — instead of at the
catch site — where a generic handler is guessing from status codes — is the
best small decision in this codebase.

Handlers, therefore, mostly don't catch anything. They throw early and let
the funnel do its job. `authorize()` throws `AuthorizationError` for you;
`validate()` throws `ValidationError`; `rejectOnEmpty` on Sequelize queries
throws instead of returning null-you'll-forget-to-check. If you find yourself
writing `try/catch` in a route handler, stop and ask what you know that the
funnel doesn't. The answer is usually "nothing."

## The hard decisions

### 1. Exceptions vs. Result types

We had the Result/Either debate when the TypeScript ecosystem got excited
about it. The pitch is real: typed errors in signatures, no invisible control
flow. We rejected it anyway, for a Koa-shaped reason: the framework's *entire*
middleware model is built on thrown exceptions propagating up the `await`
chain to `onerror`. A Result-typed core inside an exception-based shell means
converting at every boundary, and the conversion code is where bugs live.
Monoculture beat purity.

What still bothers me: TypeScript can't tell you what a function throws, so
the error contract of a command like `documentMover` is documentation and
test coverage, nothing more. Result types would have fixed exactly that. I
made the right pragmatic call and I still feel the loss.

### 2. Machine-readable `id` as the API error contract

Every error response carries the stable `id` string. Clients — our own
frontend first among them — branch on `error.id`, never on prose messages.
That means message text is free to change (and be translated) without
breaking integrations, and support can grep logs for `invalid_authentication`
across years of versions. The rejected alternative was numeric error codes
(opaque, needs a lookup table) and bare HTTP statuses (a 403 has six
different causes in this system; clients need to know which).

The wart: the `id` namespace is flat, conventions are informal, and nothing
prevents two errors from claiming the same id except grep and vigilance.
It's held for years. It will not hold forever.

### 3. Graceful shutdown is error handling

`server/utils/ShutdownHelper.ts` runs teardown in explicit phases
(`ShutdownOrder`), and `stoppable` wraps the HTTP server in
`server/index.ts`. Order matters enormously here because of what this app
*is*: at SIGTERM there are live websocket connections holding Y.Doc state
that has been debounced but not persisted (see 03). Shut down in the wrong
order — kill the DB pool before the collaboration extension flushes — and a
deploy quietly eats the last few seconds of someone's edits. So: stop
accepting new work, drain and persist collaboration state, drain queues,
*then* close Redis and Postgres. Every deploy is a small disaster drill.

The rejected alternative was the classic "SIGTERM = exit(0), let clients
retry." Most stateless apps get away with it. We are not stateless at the
edges, and pretending otherwise is how collaborative editors get a reputation
for eating text.

## Evidence to go read

- `server/errors.ts` — read the whole file; note which errors set
  `isReportable: true` (few) versus `false` (most).
- `server/onerror.ts` and `server/onerror.test.ts` — the funnel and its spec.
- `server/routes/errors.ts` — error presentation for non-API (HTML) routes.
- `server/utils/ShutdownHelper.ts` — the phase enum; map each phase to the
  subsystem it protects.
- `server/middlewares/timeout.ts` — requests that never end are errors too.

## An incident this design caused

The `isReportable` bit has a failure mode: mislabeling. For a while,
`documents.info` lookups of *deleted* documents threw a not-found variant
that someone had marked reportable, reasoning "callers holding a stale id is
suspicious." Then a large customer integrated a crawler that walked years-old
links. Sentry ingested tens of thousands of identical non-bugs, hit quota,
and — this is the actual damage — rate-limited ingestion right as a *real*
regression in a release started throwing. We were blind at exactly the wrong
moment because we'd taught the system to cry wolf. The fix was one boolean.
The lesson was a policy: an error is reportable only if an engineer should
*act* on it, and "seems suspicious" is not an action. When you add an error
type, that one bit deserves more thought than the message text.

## Before you move on

1. What are the three pieces of data every error in `server/errors.ts`
   carries, and who consumes each one?
2. Why does a policy-check failure return `isReportable: false` while a
   presenter crash is reportable — and what breaks operationally when
   someone gets this backwards?
3. During a deploy, in what order must websockets, the collaboration
   persistence layer, Bull queues, and the Postgres pool shut down? Justify
   each ordering edge with a concrete data-loss scenario.

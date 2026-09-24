# ADR-009: Typed Error Factories, Stable IDs, `isReportable`, One Funnel

**Status:** Active

## Context

A public API needs machine-consumable errors that survive message rewording
and translation. Operationally, the system must distinguish *traffic* (a 403
from a policy check — expected, frequent, boring) from *telemetry* (a
TypeError in a presenter — a bug someone must fix), or error tracking drowns
in noise exactly when it's needed. And because the app holds unpersisted
collaborative state in memory (ADR-007), process death is itself an error
path that must be handled, not just suffered.

## Decision

- **Vocabulary:** `server/errors.ts` — factory functions over `http-errors`
  (`AuthenticationError`, `AuthorizationError`, `ValidationError`,
  `RateLimitExceededError`, `InternalError`, ...), each carrying an HTTP
  status, a stable machine-readable `id` (`"authorization_error"`), and an
  `isReportable` boolean set *at the construction site*, where the author
  knows what the error means.
- **Funnel:** handlers don't catch; they throw early (`authorize()` throws,
  `validate()` throws, Sequelize `rejectOnEmpty` throws) and Koa's `onerror`
  (`server/onerror.ts`) shapes the JSON response and routes
  `isReportable: true` errors — and only those — to Sentry.
- **Contract:** clients (the SPA first among them — `app/utils/errors.ts`
  mirrors the taxonomy) branch on `error.id`, never on message prose.
- **Shutdown as error handling:** `server/utils/ShutdownHelper.ts` runs
  teardown in explicit `ShutdownOrder` phases with `stoppable` wrapping the
  HTTP server: stop intake → flush collaboration state → drain queues → close
  Redis/Postgres. Ordering is dictated by ADR-007's durability gap.

## Alternatives Considered

1. **Result/Either types.** Real benefits (throws visible in signatures),
   rejected because Koa's middleware model *is* exception propagation;
   Result-typed internals inside an exception-shaped shell means lossy
   conversion at every boundary.
2. **Catch-site classification** (generic handler guesses severity from
   status codes). Rejected: the catch site knows the least; a 500 from a
   flaky S3 and a 500 from a null deref need different routing, and only the
   construction site can say which.
3. **Numeric error codes / bare HTTP statuses** as the client contract.
   Rejected: opaque (codes) or ambiguous (a 403 has half a dozen causes
   here); string ids are greppable across years of logs.

## Consequences

**Positive:**
- Message text is free to change and translate; integrations keyed on `id`
  don't break.
- Sentry contains only what an engineer should act on — the signal-to-noise
  property the whole design exists for.
- Route handlers are nearly catch-free; error behavior is uniform because it
  is centralized (`server/onerror.test.ts` is the spec).

**Negative:**
- TypeScript cannot express what a function throws; the error contract of any
  command is documentation plus tests. This is the real price of rejecting
  Result types.
- The `id` namespace is flat and convention-governed; nothing but grep
  prevents collisions.
- `isReportable` is a single bit with high blast radius: one mislabeled
  frequent error floods Sentry into quota-limiting — muting real regressions
  (the crying-wolf failure mode is the design's own inverse).

## Evidence

- `server/errors.ts` — the full vocabulary; note how few errors set
  `isReportable: true`.
- `server/onerror.ts` + `onerror.test.ts` — the funnel and its spec.
- `server/middlewares/validate.ts` — throw-early at the boundary.
- `server/utils/ShutdownHelper.ts`, `ShutdownOrder` usage in
  `server/services/websockets.ts` and `server/index.ts` (`stoppable`).
- `server/routes/errors.ts` — the parallel presentation for HTML routes.

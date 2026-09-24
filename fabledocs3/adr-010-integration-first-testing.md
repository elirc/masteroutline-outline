# ADR-010: Integration-First Testing on Real Infrastructure

**Status:** Active

## Context

The system's risk concentrates in exactly the places mocks are blind:
Postgres-specific behavior (`FOR UPDATE`, `SET LOCAL`, tsvector search,
paranoid-scope queries), the authorization layer's composition, and
transactional atomicity of mutation + event. Code runs in three runtimes
(browser, Node, and `shared/` in both). And the worst plausible bug class is
cross-tenant data leakage in a multi-tenant-by-column schema (ADR-005).

## Decision

- **Runner:** Vitest with per-runtime **projects** (`vitest.config.ts`):
  `server` (Node + real Postgres), `app` (jsdom), and `shared` executed
  **twice** (`shared-node`, `shared-jsdom`) to enforce ADR-002's dual-runtime
  contract. All scripts pin `TZ=UTC`.
- **Server doctrine:** integration tests through the real stack.
  `server/test/TestServer.ts` boots the actual Koa app; route tests issue
  real HTTP to `/api/*` and assert status + JSON — middleware, validation,
  policies, presenters all in the loop. The database is real Postgres;
  outbound HTTP is the only faked seam (MSW, `server/test/msw.ts`).
- **Data:** programmatic factories (`server/test/factories.ts` — `buildUser`,
  `buildDocument`, ...) instead of fixture files; tests state only the
  details under test. Standing convention: authz tests always include an
  actor from a second team.
- **Placement:** tests are colocated siblings (`documentMover.ts` /
  `documentMover.test.ts`) — the test file is the de facto spec.

## Alternatives Considered

1. **Mock the ORM / repository layer.** Rejected: asserts that mocks were
   called — a tautology — and encodes *beliefs* about Sequelize that silently
   diverge from Sequelize (paranoid-scope behavior across a major upgrade is
   the canonical catch that justified the cost).
2. **SQLite for speed.** Rejected: the code leans on Postgres-only features;
   SQLite-green is not evidence.
3. **Rollback-per-test isolation.** Rejected: code under test manages its
   own transactions (ADR-005); wrapping them in an outer rollback turns them
   into savepoints with different semantics. Isolation is by cleanup
   convention (`server/test/setup.ts`, `globalTeardown.ts`) instead.
4. **Fixture files.** Rejected as rot-prone: every migration breaks them,
   and no test declares which of forty fields it depends on.

## Consequences

**Positive:**
- Framework- and database-behavior regressions surface in CI, not
  production; the tests exercise the same SQL production runs.
- The cross-team-actor convention makes forgotten authorization fail loudly
  in review (ADR-004's safety net).
- `shared` running twice catches server-only imports before they poison the
  browser bundle.

**Negative:**
- CI requires service containers and takes minutes; factories hit the real
  DB, so per-test cost is visible and cumulative.
- Cleanup-convention isolation occasionally leaks state between tests;
  bisecting those failures is miserable.
- The composition gap: multi-client concurrent editing through the
  collaboration server has **no** automated end-to-end harness — the pieces
  are tested (`documentCollaborativeUpdater.test.ts`,
  `ConnectionLimitExtension.test.ts`), the whole rests on CRDT guarantees
  plus staged rollout. The highest-risk subsystem has the thinnest
  system-level coverage; this is the strategy's standing IOU.

## Evidence

- `vitest.config.ts` + `package.json` `test:*` scripts — four projects,
  `TZ=UTC`.
- `server/test/TestServer.ts`, `factories.ts`, `msw.ts`, `setup.ts`,
  `globalTeardown.ts` — the harness.
- Any `server/routes/api/*/**.test.ts` — request-level pattern, cross-team
  403 cases.
- `server/commands/documentMover.test.ts` — command test asserting both
  return value and emitted Event.
- `server/policies/document.test.ts` — policies unit-tested directly.

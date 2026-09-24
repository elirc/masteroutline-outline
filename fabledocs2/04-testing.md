# 04 — Testing Strategy: Real Postgres, Fake Time Zones, No Mocked Routers

## What it is and why it exists

We run Vitest with **projects** — separate configs per runtime environment:
`server` (Node, real Postgres), `app` (jsdom, React), and `shared` twice
(`shared-node` *and* `shared-jsdom`, because shared code must genuinely work
in both runtimes, and "it imported fine" is not evidence). See
`vitest.config.ts` and the `test:*` scripts in `package.json`. Everything runs
under `TZ=UTC` — hardcoded in the npm scripts — because we lost real days to
date-boundary tests that passed in CI and failed on a laptop in Manila.

The server-side doctrine, in one sentence: **integration tests against real
infrastructure, through the real HTTP stack.** `server/test/TestServer.ts`
boots the actual Koa app; tests make actual HTTP calls to `/api/*` and assert
on status codes and JSON. Test data comes from `server/test/factories.ts` —
programmatic builders (`buildUser`, `buildDocument`, `buildCollection`...) —
not fixture files. Outbound HTTP is intercepted with MSW
(`server/test/msw.ts`), so third-party APIs are the *only* thing we fake.

Tests sit next to code (`documentMover.ts` / `documentMover.test.ts`), not in
a parallel tree. If you're looking for the spec of any command or route, the
sibling test file is it.

## The hard decisions

### 1. Real Postgres in tests vs. mocks/SQLite

The rejected options: mock the ORM (you end up asserting that you called the
mock — a tautology with a green checkmark), or run SQLite for speed (we use
Postgres-specific behavior everywhere — `tsvector` search in
`plugins/search-postgres`, `FOR UPDATE` locks, `SET LOCAL`, JSON operators —
so SQLite-green means nothing). We chose real Postgres and paid the price:
the suite needs a database, CI needs a service container, and the full server
project takes minutes, not seconds.

What still bothers me: test isolation is by cleanup convention
(`server/test/setup.ts`, `globalTeardown.ts`), not by
transaction-rollback-per-test, because code under test manages its *own*
transactions (see 03) and nesting a rollback wrapper around
`sequelize.transaction()` calls gets savepoint-shaped and lies to you.
Occasionally a test leaks state and breaks a neighbor, and those failures are
miserable to bisect. Known cost, still paying it.

### 2. Factories over fixtures

We had YAML-ish fixture blobs early on. They rot: every schema migration
breaks them, every test shares them, and nobody can tell which of the forty
fixture fields a given test actually depends on. Factories
(`server/test/factories.ts`) build valid objects with overridable specifics,
so a test reads as "a viewer-role user from *another team*" — exactly the
detail under test and nothing else. The convention that matters most: **every
authz-relevant test builds a second team.** Cross-tenant leakage is the bug
class this codebase can least afford (see the incident in 02), so
"user from team B gets a 403" cases are mandatory, not optional.

Tradeoff I've accepted: factories hit the real database, so they're slow, and
a lazy test that builds ten users takes a visible fraction of a second. We
periodically talk about in-memory model stubs for pure-logic tests and
periodically decide the two-tier mental model isn't worth it.

### 3. What we deliberately under-test

Honesty section. Frontend coverage (`app/`) is thin relative to the server —
mostly models, stores, and hooks, not component rendering. The editor
(`shared/editor/`) is tested at the Prosemirror-document level, but the truly
scary surface — two clients merging concurrent Yjs edits through the
collaboration server — has no automated end-to-end harness. We test the
pieces (`documentCollaborativeUpdater.test.ts`,
`ConnectionLimitExtension.test.ts`) and rely on staged rollouts + the CRDT's
mathematical guarantees for the composition. I have wanted a multi-client
collaboration simulator for years. It has never survived prioritization
against features, and every editor upgrade is more nervous-making than it
should be. If you're looking for a high-leverage infrastructure project:
this one.

## Evidence to go read

- `vitest.config.ts` — the four projects; note which environment `shared`
  runs in (both).
- `server/test/factories.ts` — skim the whole file; you will use it daily.
- `server/test/TestServer.ts` and any `server/routes/api/*/**.test.ts` — the
  request-level test pattern.
- `server/commands/documentMover.test.ts` — a model command test: factories,
  transaction, assertion on both the return value and the emitted Event.
- `server/test/msw.ts` — how outbound HTTP is stubbed.

## An incident this design prevented

A Sequelize major-version upgrade changed subtle `paranoid` (soft-delete)
query behavior — deleted documents started reappearing in one specific list
query under one specific scope combination. Unit tests with a mocked ORM
would have sailed through, because the mocks encode our *beliefs* about the
ORM, and it was a belief that had changed. The real-Postgres integration
suite caught it in the upgrade PR: three route tests failed with unexpected
rows in the response JSON. Cost of the strategy: minutes of CI per run, every
run. Value: an entire category of "the framework changed under us" bugs gets
caught before merge. I re-litigate this tradeoff every time CI feels slow,
and the answer keeps coming out the same.

## Before you move on

1. Why does `shared/` run its tests in two environments, and what actual bug
   class does the second environment catch?
2. Why don't we wrap each server test in a rolled-back transaction, given
   that it would fix test-isolation leaks?
3. Write (on paper) the test list for a new `documents.archive` endpoint.
   If your list doesn't include a cross-team 403 case and an
   emitted-Event assertion, go reread this doc.

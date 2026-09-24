# Testing Strategy

## The pyramid as it exists here (not as diagrams draw it)

| Layer | What it is here | Where | Speed |
| --- | --- | --- | --- |
| Route/integration | HTTP against a real Koa app + **real Postgres**, mocked externals | `*.test.ts` beside routes; server bootstrapped by [getTestServer](../../server/test/support.ts#L16) | seconds/file |
| Command/model/processor | direct function calls, still real DB | `*.test.ts` beside code ([documentUpdater.test.ts](../../server/commands/documentUpdater.test.ts), [RevisionsProcessor.test.ts](../../server/queues/processors/RevisionsProcessor.test.ts)) | fast |
| Presenter contract | snapshot tests | [presenters/__snapshots__](../../server/presenters/__snapshots__) | instant |
| Client | jsdom, store/util/component tests | `app/**/*.test.ts(x)` | fast |
| Shared isomorphic | same tests run in node *and* jsdom projects | [vitest.config.ts](../../vitest.config.ts#L60-L115) | instant |
| E2E browser | **none found** in-repo (no Playwright/Cypress config) — *verified by absence of config; label: inferred* | — | — |

The strategic choice worth articulating: **server tests run against real Postgres, not mocks.** Fidelity over speed — scopes, transactions, paranoid deletes, and JSONB behavior are exactly the things mocks lie about, and exactly what this app's correctness hangs on. The cost (a test DB, slower CI) is paid deliberately; the Makefile's `test` target provisions it ([Makefile](../../Makefile)).

External boundaries are mocked precisely at the trust line: S3 via `vi.mock` in [server/test/setup.ts](../../server/test/setup.ts#L16-L38), outbound HTTP via **msw** with `onUnhandledRequest: "error"` ([setup.ts L8](../../server/test/setup.ts#L1-L10)) — meaning a test that tries to hit a real network *fails loudly*. That single line is a whole testing philosophy: hermetic by default.

## What belongs at each layer

- Authorization behavior → **route tests** (the middleware chain is part of the behavior; testing policies alone misses the scoped-load half). Pattern: build user+resource topology with factories, hit endpoint, assert 403/404/200 and response shape.
- Business rules with branches → **command tests** (no HTTP noise, all DB reality).
- Contract shape → **snapshots** (cheap change detection; normalize volatile fields).
- Pure logic (helpers, string/date/url utils) → colocated unit tests; place in `shared/` if isomorphic so it runs twice.
- Queue behavior → processor/task tests calling `perform` directly; scheduling assertions via [mockTaskSchedule](../../server/test/support.ts#L43).

What *not* to test: presenter field-by-field assertions duplicated with snapshots; Sequelize itself; framework wiring already covered by one canonical route test.

## Fixtures, isolation, flake prevention

- **Factories over fixtures:** [server/test/factories.ts](../../server/test/factories.ts) builds real rows with faker defaults + overrides (`buildUser`, `buildDocument`, `buildDraftDocument`, `buildViewer`, `buildAdmin`…). Factories compose (a document builds its team/user if absent) — learn the graph once, write tests fast forever.
- **Isolation:** parallel test files with a shared DB (`fileParallelism: true` — [vitest.config.ts](../../vitest.config.ts#L66-L80)) work because tests create their *own* rows and assert on them, not on global counts. Anti-flake rule you can carry anywhere: never assert on data you didn't create.
- **Time/randomness:** `TZ=UTC` pinned in the test script ([package.json L31](../../package.json#L31-L35)); date-sensitive tests construct explicit dates (`subDays`, `addMinutes` — see documents.test.ts imports).
- **Unhandled rejections** are tolerated globally (`dangerouslyIgnoreUnhandledErrors: true`, [vitest.config.ts](../../vitest.config.ts#L60-L66)) — a pragmatic tradeoff; know it exists because it can mask real async bugs (*possible risk*).

## Interview angle

1. *"What's your testing strategy?"* — the layered table + the real-DB rationale; naming the msw `onUnhandledRequest: "error"` detail lands.
2. *"How do you prevent flaky tests?"* — own-data assertions, pinned TZ, hermetic network, awaiting real completion instead of sleeps.
3. *"Unit vs integration tradeoff?"* — this repo's answer: integration-heavy where the framework/DB *is* the risk; unit where logic is pure.
4. *"How do you test authorization?"* — route tests with membership topologies; policy unit tests for the lattice ([document.test.ts](../../server/policies/document.test.ts)).

## Drill

Open [documents.test.ts](../../server/routes/api/documents/documents.test.ts) and map its first ~10 test names onto the table above: which layer is each really exercising? Find one test you'd *demote* to a cheaper layer and one gap you'd add (candidates from [good-first-tickets](../06-contribution-practice/01-good-first-tickets.md) tickets 5, 11, 16).

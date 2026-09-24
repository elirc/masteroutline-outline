# Good First Tickets

Sixteen junior-scope practice tickets, spread across the repo. These are *training tickets*: each follows a real pattern in this codebase and is shaped like something a maintainer could accept, but **verify against current upstream issues/code before investing** — the repo moves fast, and a few tickets are explicitly "first, confirm the gap exists." Every ticket answers the contribution-realism questions: why useful, why small blast radius, which existing pattern it follows, how it's tested, and what would get it rejected.

Estimated times assume the fast-track setup is done. Grade yourself with [09-reference/learning-rubrics.md](../09-reference/learning-rubrics.md).

---

## Ticket 1: Improve the validation error to report all issues, not just the first
Difficulty: Easy — 2–3 h. Skills: Zod, middleware, API error contracts.
Story: As an API consumer, I want all validation failures in one response so I can fix my request in one round trip.
Read first: [server/middlewares/validate.ts#L17-L25](../../server/middlewares/validate.ts#L17-L25) — only `err.issues[0]` is reported.
Why good: tiny surface (one middleware), improves DX for every endpoint; follows the existing `ValidationError` contract.
Acceptance criteria: multiple invalid fields produce a message listing each `path: message`; single-issue behavior unchanged; existing tests pass.
Files touched: `server/middlewares/validate.ts`, a new/extended test (see [validation.test.ts](../../server/validation.test.ts) and route tests for message assertions).
What could go wrong: clients (or tests) asserting on the exact old message string — grep test snapshots for `ValidationError` messages first.
Rejection risk: maintainers may prefer the terse first-error message for response-size or UX reasons — check upstream discussion first; keep the format conservative (join with `; `).
Interview story potential: "I improved API error ergonomics without breaking the error contract — and found which tests encoded the old contract."

## Ticket 2: Add a missing JSDoc pass to one undocumented exported helper module
Difficulty: Easy — 1–2 h. Skills: reading unfamiliar code precisely, project conventions.
Story: As a contributor, I want exported functions documented per [AGENTS.md](../../AGENTS.md#L143-L152) so the next reader doesn't re-derive behavior.
Read first: AGENTS.md JSDoc rules; pick a small util in [server/utils](../../server/utils) or [shared/utils](../../shared/utils) with undocumented exports (grep for `export function` without a preceding `/**`).
Why good: zero runtime blast radius; forces you to *actually understand* the function to describe it.
Acceptance criteria: every exported symbol in the chosen file has JSDoc with `@param`/`@returns` in house style; no behavior change.
Rejection risk: wrong or vacuous docs ("does the thing") — a doc that restates the name adds noise and will be rejected.
Interview story potential: weak alone; pairs as the "how I onboarded" detail in a bigger story.

## Ticket 3: Add a regression test for `Store.remove` cascade behavior
Difficulty: Easy–Medium — 3–4 h. Skills: MobX, client architecture, vitest jsdom project.
Story: As a maintainer, I want the client-side cascade delete (documents removed when their collection is removed) locked in by a test.
Read first: [app/stores/base/Store.ts#L174-L221](../../app/stores/base/Store.ts#L174-L221) (remove + inverse relations), [app/models/decorators/Relation.ts](../../app/models/decorators/Relation.ts), existing store tests (glob `app/stores/*.test.ts`).
Why good: tests-only; documents subtle behavior (client mirrors FK semantics) that refactors could silently break.
Acceptance criteria: test proves cascade and `null` behaviors of `onDelete`; runs in the `app` vitest project.
What could go wrong: instantiating stores outside RootStore misses relation registration — follow how existing tests build a RootStore.
Rejection risk: duplicate coverage — check `Collection.test.ts`/store tests first.
Interview story potential: "I found the client cache re-implements referential integrity and pinned it with tests."

## Ticket 4: Snapshot test for an un-snapshotted presenter
Difficulty: Easy — 2 h. Skills: API contracts, vitest snapshots.
Story: As a maintainer, I want presenter output shapes locked so accidental field additions/removals are visible in review.
Read first: [server/presenters/user.test.ts](../../server/presenters/user.test.ts) and [__snapshots__](../../server/presenters/__snapshots__); pick a presenter without a test (candidates: [share.ts](../../server/presenters/share.ts), [pin.ts](../../server/presenters/pin.ts) — verify none exists).
Why good: presenters are the public contract; snapshots make contract diffs reviewable. Follows the exact existing pattern.
What could go wrong: nondeterministic fields (dates, ids) — use factories with fixed values like the user presenter test does.
Rejection risk: snapshotting volatile fields creates flaky churn; normalize them.
Interview story potential: "contract testing on a public API" — small but articulate.

## Ticket 5: Add `documents.info` test for the share + authenticated-user branch
Difficulty: Medium — 3–5 h. Skills: authz reasoning, integration tests, factories.
Story: As a maintainer, I want the trickiest branch of `documents.info` — signed-in user opening a share they *also* have private access to — explicitly tested (`isPublic` should be false, policies present).
Read first: [documents.ts L589–650](../../server/routes/api/documents/documents.ts#L589-L650); existing share tests in [documents.test.ts](../../server/routes/api/documents/documents.test.ts) (confirm this exact case is missing first!).
Why good: tests-only; covers a security-relevant boundary.
Acceptance criteria: asserts field differences between anonymous and member on the same shareId (e.g. `updatedAt` withheld vs present, `policies` undefined vs populated).
Interview story potential: strong — "I identified and tested an authorization edge case in a public-sharing flow."

## Ticket 6: Extract a `getAPIVersion` usage into a documented helper test
Difficulty: Easy — 2 h. Skills: API versioning mechanics.
Story: As a contributor, I want the `x-api-version` behavior (see [documents.info L580, L640–648](../../server/routes/api/documents/documents.ts#L574-L651) and [presenter L38](../../server/presenters/document.ts#L38)) covered by direct tests so version bumps are safe.
Why good: versioning is contract-critical and currently only indirectly tested (verify!).
Rejection risk: if a helper test already exists under routes/api — search first.
Interview story potential: "how we versioned an API in place" supporting detail.

## Ticket 7: Add an index-of-emails README table (docs-in-code exception) — SKIP upstream; do locally
Difficulty: Easy — 1 h. Skills: honest scoping.
This one is a deliberate trap: AGENTS.md forbids new markdown files. The junior lesson is recognizing *which* contributions a repo's conventions preclude. Do it in your fork notes instead, and read three templates in [server/emails/templates](../../server/emails/templates) while you're there.
Interview story potential: "I read the contribution rules before contributing" — genuinely worth saying.

## Ticket 8: Normalize a duplicated `where` clause in `documents.list`
Difficulty: Medium — 3–4 h. Skills: Sequelize, careful refactoring.
Story: As a maintainer, I want the repeated `where[Op.and].push({...})` filter assembly in [documents.list L119–171](../../server/routes/api/documents/documents.ts#L119-L171) readable — e.g., extracted into a small local builder — without behavior change.
Why good: pure refactor of one handler; the route's tests are extensive, so safety is verifiable.
What could go wrong: subtle scope/order semantics of `Op.and` arrays; run the documents.list tests before and after.
Rejection risk: cosmetic refactors of hot files are often declined ("churn without benefit") — pair it with a test improvement or don't ship it; still valuable as practice.
Interview story potential: "refactoring under test coverage" fundamentals.

## Ticket 9: Add a `shouldQueue` fast-path to a processor missing one
Difficulty: Medium — 4–6 h. Skills: async architecture, performance thinking.
Story: As an operator, I don't want empty jobs queued for processors that will no-op. [BaseProcessor.shouldQueue](../../server/queues/processors/BaseProcessor.ts#L11-L20) exists; find a processor whose `perform` starts with a cheap bail-out (e.g., checks event data and returns) and hoist that check into `shouldQueue`.
Read first: BaseProcessor; [RevisionsProcessor L20–23](../../server/queues/processors/RevisionsProcessor.ts#L17-L24) (its `done` guard is a candidate — *investigate whether hoisting is semantically safe*: `shouldQueue` runs in the global-queue worker, same data available).
Why good: real perf win (fewer Redis jobs), follows an explicitly provided extension point.
Acceptance criteria: processor test proves both queued and skipped paths.
Rejection risk: hoisting a check that depends on DB state (not event payload) would change semantics under retry — only hoist pure payload checks.
Interview story potential: strong — "I reduced queue load by using the framework's opt-out hook, and knew which checks were safe to hoist."

## Ticket 10: UI — show a tooltip on a truncated title where one is missing
Difficulty: Easy–Medium — 3 h. Skills: React, styled-components, a11y.
Story: As a user, I want to read full document titles that are visually truncated in lists.
Read first: how existing components do it — grep `Tooltip` in [app/components](../../app/components); pick a list row lacking it (verify by running the app).
Why good: contained visual change; follows the house Tooltip component.
Rejection risk: tooltip spam or duplicate a11y labels; check the component isn't intentionally bare.
Interview story potential: minor, but demonstrates UI polish sensibility.

## Ticket 11: Add a test for `parseAuthentication` transport rules
Difficulty: Easy–Medium — 3 h. Skills: security reasoning, unit tests.
Story: As a maintainer, I want the "API key must not arrive via cookie / OAuth token only via header" rules ([authentication.ts L156–203](../../server/middlewares/authentication.ts#L156-L203)) pinned by tests, since they're CSRF defenses.
Read first: [authentication.test.ts](../../server/middlewares/authentication.test.ts) — confirm which transports are already covered.
Why good: tests encode a *security invariant*; high review value, zero risk.
Interview story potential: strong security-mindedness signal.

## Ticket 12: Fix-or-verify — stale JSDoc/comment hunt in one directory
Difficulty: Easy — 2 h. Skills: skepticism.
Story: As a maintainer, I want comments that lie removed or fixed. Sweep one directory (suggest [server/commands](../../server/commands)) comparing each JSDoc against behavior; fix at most 2–3 genuinely wrong ones.
Why good: small diffs, real value; teaches you to distrust comments (a senior habit).
Rejection risk: opinion-only rewording — only touch provably wrong text.
Interview story potential: seasoning for "attention to detail" answers.

## Ticket 13: Add a factory helper for a model missing one
Difficulty: Medium — 3–4 h. Skills: test infrastructure.
Story: As a test author, I want `buildX` for a model currently constructed by hand in tests. Scan [server/test/factories.ts](../../server/test/factories.ts) exports vs [server/models](../../server/models) index; candidates: `Relationship`? `DocumentInsight`? (verify which lack factories *and* are hand-built somewhere).
Why good: follows an unmistakable existing pattern; makes future tests cheaper.
Acceptance criteria: factory with sensible defaults + overrides; at least one existing test refactored to use it.
Interview story potential: "investing in test infrastructure" — a good mid-level signal.

## Ticket 14: Add missing `Interview`-grade logging context to a task's failure path
Difficulty: Medium — 3 h. Skills: observability.
Story: As an on-call engineer, when a task exhausts retries I want enough context logged to act. Review `onFailed` implementations (grep `onFailed` under server/queues); find one that swallows context and enrich the error log with ids following [worker.ts's logging style](../../server/services/worker.ts#L188-L198).
Why good: operations value; tiny diff; existing style to copy.
Rejection risk: logging PII (document text, emails) — ids only.
Interview story potential: "how I made failures debuggable" — excellent on-call story seed.

## Ticket 15: Type tightening — remove one `any`/`@ts-expect-error` safely
Difficulty: Medium — 3–6 h. Skills: TS depth.
Story: As a maintainer, I want one `oxlint-disable-next-line no-explicit-any` or `@ts-expect-error` site (grep both; e.g. the TODO at [cancan.ts L248](../../server/policies/cancan.ts#L243-L254)) replaced with a real type.
Why good: mechanical value; forces deep type understanding.
What could go wrong: variance issues are sometimes *why* the `any` exists (see the comment at [cancan.ts L5](../../server/policies/cancan.ts#L1-L10)) — if the suppression is load-bearing, write up *why* instead and pick another site. That writeup is still a win.
Interview story potential: strong TS-mastery evidence either way.

## Ticket 16: Add a `documents.search` test for the collection-permission fence
Difficulty: Medium — 4 h. Skills: authorization, search.
Story: As a maintainer, I want a test proving a user cannot get results from a collection they lost access to, even with a direct `collectionId` filter ([documents.search collection branch](../../server/routes/api/documents/documents.ts#L1115-L1125)).
Read first: existing search tests in documents.test.ts (confirm the negative case is absent).
Why good: security regression net over the scariest query in the app.
Interview story potential: strong — cross-tenant/IDOR-adjacent testing story.

---

## Working protocol (applies to every ticket)

1. Reproduce/verify the gap. 2. Write the failing test or before-state note. 3. Smallest diff. 4. `yarn lint`, `yarn tsc`, targeted `yarn test path` (per [AGENTS.md](../../AGENTS.md#L153-L173)). 5. PR description per [07-career-and-collaboration/02-writing-prs-and-rfcs.md](../07-career-and-collaboration/02-writing-prs-and-rfcs.md). 6. Write the one-paragraph STAR note while it's fresh ([08-interview-prep/06-behavioral-star-stories.md](../08-interview-prep/06-behavioral-star-stories.md)).

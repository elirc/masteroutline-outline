# SEALED — rubrics for learn/01-agentic-practice.md

Grade the agent's output **and** your spec. For every rubric item the agent got wrong, ask: did my spec
prevent it, or did my review catch it? If neither, that is the lesson. Candidate placement details for D1–D3
are in `training/_answers/02-change-candidates-answers.md`. They are not repeated here.

---

## D1 — `DocumentUpdateTextTask.test.ts`

**A good spec contains:** calls `perform()` directly (because `schedule()` goes through Bull, which is not a
unit test); the event is a minimal `DocumentEvent`-shaped object with at least `documentId` (check
`@server/types` for the required fields, the way `RevisionsProcessor.ts:46` forwards its event); documents come
from `buildDocument`/`buildTeam` in `@server/test/factories` (house style, as in
`CleanupDocumentInsightsTask.test.ts:1-3`); one collocated file; the task is not modified.

**Rubric (pass = 6/7, and item 1 is mandatory):**
1. Calls `new DocumentUpdateTextTask().perform(event)`, not `.schedule`.
2. Early-return case: a document with falsy `content`, or a missing id, and the test asserts that `text` is **unchanged** after a reload. Asserting only "does not throw" is weak.
3. Happy path: it reloads the row (`document.reload()` or `findByPk`) and asserts `text` equals a **specific** markdown string built from known `content`. `toBeTruthy()` fails this item.
4. The short-text language case is either asserted or explicitly listed as out of scope with a reason. It must not be silently skipped.
5. No new directory, no `any`, and no `as` casts on the event unless justified in a comment. If a cast is unavoidable, the reviewer asks for `Partial<DocumentEvent>` or a small builder instead.
6. Factories are reused, with no raw `Document.create` holding hand-written ProseMirror JSON if a factory option exists. Check the `buildDocument` signature in `server/test/factories.ts`.
7. The PR notes say which command was run and its result. **Until Docker is back this must say "not run: requires Postgres"**. An agent claiming "tests pass" without a running Postgres is a finding.

**Common agent failures:** mocking `Document.findByPk`, so the test proves nothing about Sequelize; asserting on
the in-memory `document` object that was passed in rather than the reloaded row; adding a `franc` mock that
makes the language assertion tautological.

## D2 — persistence-failure metric

There is no single correct placement. You pass if the **trade-off is explicit and the library behaviour is
cited**.

- **Option 1:** `Metrics.increment("collaboration.persist_failed", { documentName })` inside `PersistenceExtension`'s `catch`. Small and honest. The cost is that metric names now live in two files.
- **Option 2:** keep counters in `MetricsExtension`. That requires the failure to be visible to it: rethrow from `PersistenceExtension`, or share a signal. It is only valid if the agent **cited** what `@hocuspocus/server` does when `onStoreDocument` throws: whether later extensions' hooks run, whether the error is logged or retried, and whether clients are affected. Rethrowing without that citation fails this rubric, because it could change user-visible behaviour on every save failure.

**Rubric:**
1. The name follows the flat `collaboration.<verb>` convention and has the `documentName` tag (compare `MetricsExtension.ts:16-65`).
2. `documentCollaborativeUpdater.ts` is untouched.
3. The test stubs the updater (or its transaction) to reject, then asserts that the metric call happened. It does **not** try to create a real 15 s `lock_timeout`.
4. The PR text says: "the wiring is unit-tested; a real lock contention was not reproduced".
5. Bonus: it notices that `collaboration.change` still counts attempts, and proposes (without necessarily doing) a success counter so the dashboard can show a failure **rate**, not just a count (curriculum 09).

**Automatic reject:** any sentence like "Hocuspocus will retry the store" with no file:line in
`node_modules/@hocuspocus/server`.

## D3 — aria-label

**Reference diff:** add `aria-label={t("Viewers")}` to the `NudeButton` at `app/components/Collaborators.tsx:167`.
Nothing else changes.

**Rubric:**
1. Reuses the existing `t("Viewers")` key. No new key and no locale-file edits. Extraction is automatic, per `AGENTS.md`.
2. The diff touches one file.
3. No component-test harness is added. The PR description explains why there is no test (there is no component-test precedent in `app/components/`).
4. Reviewer's check: the accessible name should describe what the button **does** (it opens the viewers list). Is "Viewers" enough, or would a maintainer prefer something like "Show viewers"? It is fine to raise this as an open question in the PR. Don't decide it on the maintainer's behalf by inventing new copy.

## D4 — static guard for `shared/`

**A good outcome:** a new `overrides` entry scoped to `"files": ["shared/**/*.{ts,tsx}"]` with
`no-restricted-imports` `patterns` for `@server/*` and `node:*`, plus `paths` for `fs`, `path` and `crypto` as
**exact names**.

**Rubric:**
1. It doesn't match `crypto-js/md5` (`shared/utils/color.ts:1`). A glob like `crypto*` fails this item.
2. The `prosemirror-tables` restriction still applies to `shared/**`. The reviewer confirmed from oxlint docs whether a later override's rule options **replace** earlier ones (in ESLint's flat config they do). If they replace, the `prosemirror-tables` path has to be repeated in the `shared/` override. An agent that "just added" the override without checking fails this item. This is the core lesson: config changes can silently delete protections.
3. Proof: a planted `@server/env` import in a `shared/utils` file makes `yarn lint` fail, and after reverting it passes. **[needs `yarn install`]**
4. `yarn lint` on the untouched tree doesn't produce new errors. At `37ca0d5` a grep found no `@server/`, `fs`, `path` or `node:` imports under `shared/`, so zero new errors is expected. If there are some, the rule is wrong or the tree has a real violation. Say which.

## D5 — lock-site audit

**Reference list from grep at `37ca0d5`** (`grep -rn "LOCK.UPDATE" server --include=*.ts | grep -v test`). The
learner must open each hit to decide which model it locks. The document-related candidates include:
- `server/collaboration/PersistenceExtension.ts:55`: the lazy conversion, `documents` row. **The one a "document" keyword filter misses.**
- `server/commands/documentCollaborativeUpdater.ts:46-49`: `lock: { of: Document, level: UPDATE }`.
- `server/models/Document.ts:468, 488, 1057, 1142, 1176, 1201, 1256`: model methods. Some of these may lock *collections* or other rows during document operations, so check each one.
- `server/commands/documentMover.ts:48, 57, 189`: check which of these are collection locks.
- `server/routes/api/documents/documents.ts:1738, 1804, 1845, 1906`: route-level locks.
- `server/queues/tasks/DocumentSubscriptionRemoveUserTask.ts:39`: probably a subscription row, not a document. A classic false positive.
- `server/scripts/20230815063834-migrate-emoji-in-document-title.ts:47`: a one-off migration script. Not a runtime path.

Scale: at `37ca0d5`, `LOCK.UPDATE` appears 99 times across 47 non-test files under `server/`. Most of them
lock other models (users, groups, comments, shares and so on). An agent that returns fewer than about 15
candidates probably filtered by keyword. One that returns all 99 as "document locks" didn't open the files.

**Rubric:** the learner (a) found `PersistenceExtension.ts:55`, (b) classified at least 3 hits as
not-a-document lock, with a reason, (c) excluded the migration script as not on a runtime path, and (d) wrote a
false-positive and false-negative count for the agent's list. For fire drill 2, the two that convoy are
`PersistenceExtension.ts:55` and `documentCollaborativeUpdater.ts:46-49`. Naming them earns the drill.

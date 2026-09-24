# Sealed: Change-Candidate Placement Answers

Don't open this until you've written your own placement answer for each
candidate in `training/navigation/02-upstream-contribution-kit.md` §4.

## Candidate A — Collaborators.tsx aria-label

**Placement:** the fix is a single prop on the `NudeButton` at
`app/components/Collaborators.tsx:167`, e.g.
`aria-label={t("Viewers")}` — reusing the exact same translation key
already used two lines below on `PopoverContent` (line 180), not inventing
new copy. This is the smallest correct diff: one line, no new translation
string, matches the existing label for the content the button opens.

**Test:** checked reality — `app/components/` has **zero** `*.test.tsx`
files anywhere in the tree (`app/` tests that exist are all logic/model/
util tests: `app/models/*.test.ts`, `app/utils/*.test.ts`,
`app/scenes/*/hooks/*.test.ts`, etc — none render a component). There is no
existing component-render-test convention to extend. The honest answer to
"is a test warranted" is: not by matching an existing pattern, because
there is none to match — adding the first component-test harness for this
codebase is a separate, much larger PR (test setup, jsdom config choices,
precedent-setting) and out of scope for a first accessibility fix. State
this explicitly in the PR description rather than silently skipping tests —
a reviewer who expects a test and gets none unexplained will bounce the PR.

**What CI wouldn't catch:** a wrong or missing translation key still passes
lint/types/tests — `oxlint`/`tsc` don't know `t("Viewers")` resolves to
sensible copy, and there's no visual/a11y CI gate (no axe-core or similar
in `ci.yml`). A `t("viewers")` (wrong case) or a brand-new key that never
gets added to the locale files would ship silently.

## Candidate B — DocumentUpdateTextTask.test.ts

**Placement:** new file `server/queues/tasks/DocumentUpdateTextTask.test.ts`,
sibling to the task, matching `CleanupDocumentInsightsTask.test.ts`'s shape
exactly: import `buildDocument`/`buildTeam` (and whatever else is needed,
e.g. a document with `content` set) from `@server/test/factories`,
`describe("DocumentUpdateTextTask", ...)`, instantiate with
`new DocumentUpdateTextTask().perform(event)` — but check the task's real
signature first: `perform(event: DocumentEvent)` takes an **event**, not a
document directly, so the factory-built document's id needs to be threaded
through a fake `DocumentEvent` shape (check `@server/types` for the exact
fields `RevisionsProcessor` passes at
`server/queues/processors/RevisionsProcessor.ts:46`,
`new DocumentUpdateTextTask().schedule(event)` — schedule vs perform: the
test should call `.perform()` directly to test the unit, not `.schedule()`
which goes through the real queue).

**Minimum cases:**
1. Happy path: document with `content` set → after `perform()`, re-fetch
   (or check the mutated in-memory `document.text`) and assert `text` is
   the markdown serialization of that content, and `language` got set.
2. Early-return branch: a document (or a `findByPk` miss / a document with
   `content` falsy) → assert `perform()` returns without throwing and
   without mutating `text`/`language`. Line 11
   (`if (!document?.content) { return; }`) is exactly this branch — a
   document that exists but has no `content` yet (e.g. a brand-new draft
   before its first Yjs save) is a real, reachable state, not a
   hypothetical.

That's 2 cases minimum to cover both branches of the one `if`. A third,
optional case worth naming even if not written: language detection with
text below `franc`'s `minLength: 50` (line 25) — `franc` will return `"und"`
(undetermined), and `iso6393To1["und"]` is `undefined`, so `document.language`
would be set to `undefined` — worth an explicit assertion since it's a
silent fallback, not an error.

## Candidate C — persistence-failure metric

**Placement:** `PersistenceExtension.ts` doesn't currently import
`Metrics` (`server/logging/Metrics`) — `MetricsExtension.ts` does, and is
the file responsible for all `collaboration.*` counters today (single
responsibility: only `MetricsExtension` should call `Metrics.increment`,
keeping metric names in one place). The correct fix is **not** to add
`Metrics` to `PersistenceExtension`'s catch block — that would put counters
in two files. Instead, the catch block in `PersistenceExtension.onStoreDocument`
(lines 138-143) should **re-throw** (or Hocuspocus needs a way to signal
failure — check the `@hocuspocus/server` `onStoreDocument` contract: does
returning/throwing from this hook do anything observable to Hocuspocus
itself, e.g. does it retry or just swallow?) so that `MetricsExtension`'s
own `onStoreDocument` hook (currently just `collaboration.change`, line
62-66) can distinguish the outcome — e.g. increment
`collaboration.persist_failed` in a wrapping try/catch there, or (cleaner)
have `PersistenceExtension` catch, log, and **rethrow**, and add a *new*,
separate `onStoreDocument`-adjacent extension or amend `MetricsExtension`
to wrap its own call in try/catch and tag failure vs success. Naming,
matching the existing flat `collaboration.<verb>` convention
(`collaboration.load_document`, `collaboration.connect`,
`collaboration.disconnect`, `collaboration.change`):
`collaboration.persist_failed`, tagged with `documentName` like every other
counter in this file.

**Test:** proving a metric fires on a real Postgres `lock_timeout` isn't
practical in CI without inducing an actual lock contention (two concurrent
transactions holding `LOCK.UPDATE` on the same row past 15s) — expensive
and flaky. The practical test is at the unit level: mock/stub
`documentCollaborativeUpdater` (or the `sequelize.transaction` it opens) to
reject, call `PersistenceExtension.onStoreDocument` directly with a
fabricated payload, and assert `Metrics.increment` (or whatever the chosen
call site is) was called with the failure metric name — this only proves
the wiring, not that a real lock timeout reaches it, and the exercise
answer should say so explicitly rather than overclaiming coverage. This is
also the point to raise in a real PR description: "unit-tested the wiring;
did not reproduce the underlying lock contention in CI."

## The meta-lesson

None of these three fixes required touching `server/collaboration/PersistenceExtension.ts`'s
actual transaction logic, `documentCollaborativeUpdater`'s write path, or
anything in `shared/editor/` — that's exactly why they're plausible first
PRs and the CRDT internals aren't. A good first-issue filter is "does the
fix require understanding X, or does it require *not breaking* X while
sitting next to it." All three candidates are the second kind.

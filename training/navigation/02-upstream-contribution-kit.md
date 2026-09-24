# Upstream Contribution Prep Kit

Outline is a real, actively-maintained OSS project (github.com/outline/outline).
This kit prepares you to send it (or a project like it) a genuine first PR.
Nothing here is submitted for you — it's practice for judgment you'll need
before you open a real PR. `fabledocs/06-contribution-practice/` already has
junior/mid/senior tickets written as if merging into *this* codebase as a
teammate; this kit is specifically about the **outside-contributor** motion
against **upstream**: finding a real issue, reading the repo's actual rules,
and getting a first PR through review.

## 1. This repo's actual contribution rules (as shipped, not assumed)

Evidence, not a generic checklist — this repo has no `CONTRIBUTING.md`; its
rules live in three places:

- **`AGENTS.md`** (repo root) — written for AI coding agents but binding for
  any contributor's diff style: yarn only for deps; `oxfmt`/`oxlint`
  formatting; TypeScript strict, no `any`, prefer `interface` over `type`;
  JSDoc required on every exported function/class; **"Critical – Do not
  create new markdown (.md) files"** (this is why `fabledocs*/` and this
  `training/` tree exist only locally and are never meant to go upstream —
  see `fabledocs/README.md`'s own note on this, and don't violate it in an
  incident/exercise either); class member ordering rules; MobX conventions
  for `app/stores/`; "do not add translation strings manually — they will
  be extracted."
- **`.github/workflows/ci.yml`** — what actually gates a PR merge:
  - `lint` job: `yarn lint --quiet` (oxlint, type-aware)
  - `types` job: `yarn tsc`
  - `test` / `test-server` jobs, **path-filtered** (`dorny/paths-filter`):
    only run `test:app`/`test:shared` if `app/**`/`shared/**` changed, only
    run `test-server` (needs a live Postgres 14.2 + Redis service container
    in the job) if `server/**`/`shared/**` changed. A docs-only or
    single-file `app/` change legitimately skips the server suite — know
    this before promising a reviewer "I ran everything."
  - `audit` job only runs when `package.json`/`yarn.lock` changed.
- **`.github/ISSUE_TEMPLATE/bug_report.yml`** — the shape a maintainer
  expects a bug report in: reproduction steps, expected vs actual,
  environment. Mirror this shape in your own future issues.

Commands (from `package.json` `scripts`, verified present, not run against a
live DB in this pass — see `training/README.md` verification notes):
`yarn lint`, `yarn tsc`, `yarn test:app`, `yarn test:shared`, `yarn
test:server` (needs Postgres+Redis), `yarn format:check`.

## 2. Finding a genuine good-first-issue (method, not a link)

Don't trust a repo's own "good first issue" label blindly — labels rot.
Cross-check before you invest time:

1. Search open issues for the label, then **filter for**: no assigned PR
   already open against it (check linked PRs), last maintainer comment
   isn't "closing, stale," and the issue doesn't require touching
   `server/collaboration/` or `shared/editor/` (CRDT/ProseMirror internals
   are not first-PR territory in *any* editor project — confirm this
   against `fabledocs3/adr-007...md`'s own "scar tissue" framing before you
   pick anything collaboration-adjacent).
2. Read the **linked PR history** on a similar past issue (same label, same
   area) to see what a merged fix there actually touched — this tells you
   the real blast radius faster than reading the issue text alone.
3. Prefer issues that already have a maintainer reply narrowing scope
   ("yes, this should be a ~5 line fix in X") over issues with none —
   unscoped issues are a trap for a first PR.
4. If nothing suitable is open, the three candidates below are the kind of
   issue this codebase plausibly has sitting unfiled — treat filing one of
   them (accurately, with the evidence below) as the exercise, whether or
   not you also open a real upstream issue.

## 3. First-PR checklist (generic steps, project-specific gates)

- [ ] Fork, clone, `yarn install`, `.env` from `.env.sample`, confirm you
      can run the affected test project (`yarn test:app` /
      `yarn test:server` — the latter needs local Postgres+Redis; see
      `fabledocs2/04-testing.md` for the project's testing philosophy).
- [ ] Confirm the issue is still reproducible against current `main` before
      writing code — labels and issue bodies drift.
- [ ] One logical change per PR. Outline's own CI is path-filtered
      precisely because small, scoped diffs are cheap to review — a mixed
      `app/` + `server/` PR pays the full CI matrix for no reason.
- [ ] Match the file's existing conventions exactly before improving them
      (see `AGENTS.md`'s "Class Member Order," styled-components, `handle`-
      prefixed event handlers). A first PR is not the place to also
      refactor.
- [ ] Add or extend a test in the same convention as sibling files (see
      candidate 2 below for what "sibling convention" means concretely).
- [ ] `yarn lint`, `yarn tsc`, and the relevant `yarn test:*` all green
      locally before pushing — don't rely on CI to find your lint errors.
- [ ] PR description: what broke / was missing, why, the fix, how you
      verified it (paste the test command and result), and anything you
      deliberately left out of scope.
- [ ] No new `.md` files in the product tree (`AGENTS.md` rule above) —
      document the change in the PR description and code comments instead.
- [ ] Respond to review by explaining trade-offs, not just applying diffs
      silently — `fabledocs/07-career-and-collaboration/` (existing pack)
      covers this mindset in depth; don't duplicate it here.

## 4. Three candidate change areas (real, evidence-backed, unfiled)

Each is a genuine gap found while tracing this codebase for this kit — not
invented. Each includes a **placement exercise**: before looking at the
sealed answer in `training/_answers/02-change-candidates-answers.md`,
write down (a) which file(s) you'd change, (b) which test file you'd add or
extend, (c) one thing that could make your fix wrong in a way CI wouldn't
catch.

### Candidate A — accessibility: unlabeled popover trigger

`app/components/Collaborators.tsx:164-184`. The `NudeButton` at lines
167-178 is the *only* interactive element that opens the "Viewers" popover
(it wraps a `Facepile` of avatar images, no visible text). Its sibling,
`PopoverContent` two lines below, has `aria-label={t("Viewers")}` — the
*content* is labeled but the *trigger* isn't. A screen reader user tabbing
through the document header hits an unlabeled button.

**Placement exercise:** where does the fix go, and what convention in this
same file (or a neighboring one) should the label text and translation key
follow? Is a test warranted, and if so what would it assert (this app's
test setup is Vitest + Testing Library — check `app/components/*.test.tsx`
for the existing pattern before answering)?

### Candidate B — missing test: the projection-refresh task

`server/queues/tasks/DocumentUpdateTextTask.ts` (33 lines — the task from
step 7 of `01-collaborative-save-path-trace.md`, which turns a
collaboratively-edited `content` JSON back into the `text` markdown
projection plus language detection) has **no `DocumentUpdateTextTask.test.ts`**.
Compare against its siblings in the same directory —
`CleanupDocumentInsightsTask.test.ts`, `DocumentPublishedNotificationsTask.test.ts`,
`RollupDocumentInsightsTask.test.ts`, `UpdateDocumentsPopularityScoreTask.test.ts`
all exist. This task sits directly on the save path you just traced and has
a real edge case visible in its own source: line 11,
`if (!document?.content) { return; }` — early-return on empty content is
untested.

**Placement exercise:** what would a first test for this task assert, using
which of this repo's test helpers (check `server/test/factories.ts` or
similar — `fabledocs2/04-testing.md` describes the factories-over-fixtures
convention; don't re-derive it, cite it)? What's the minimum number of
cases to cover the early-return branch plus one happy path?

### Candidate C — observability: silent persistence failures

`server/collaboration/PersistenceExtension.ts:130-143` (`onStoreDocument`):
`documentCollaborativeUpdater` is called inside a `try/catch` where the
`catch` only does `Logger.error(...)` — no metric, no re-throw, no signal
that a collaborative save silently failed to persist. Compare against
`server/collaboration/MetricsExtension.ts:62-66`, whose own
`onStoreDocument` hook increments `collaboration.change` **unconditionally,
on every store *attempt*** — including cases `documentCollaborativeUpdater`
itself will no-op on (empty collaborator set, or `isEqual` content check —
see trace steps 4-5) and including this failure case. So today,
`collaboration.change` cannot distinguish "saved," "no-op, nothing to
save," and "attempted and failed" — and the failure case has no counter of
its own at all. In production this means a spike in save failures (a
transient `lock_timeout`, say — the exact incident ADR-007 says already
happened once) would be invisible on a `collaboration.change` dashboard.

**Placement exercise:** where would you add a new metric, what would you
name it (check the existing `collaboration.*` naming convention in
`MetricsExtension.ts` before inventing one), and which extension's hook is
the right place to increment it — `MetricsExtension` (which never sees
success/failure today) or `PersistenceExtension` (which does, but doesn't
currently import `Metrics`)? What test, if any, could prove a metric fires
on the failure path without a real Postgres lock-timeout in CI?

## 5. Why these three and not others

Per this batch's skill-gap targeting (see the report file, not duplicated
here): accessibility, frontend/React perf, and observability are
under-trained skills across this archive, while authz/IDOR is saturated —
so no authz-shaped candidate was chosen even though `server/policies/*.ts`
has several files without a matching `.test.ts` (e.g. `share.ts`,
`comment.ts`, `template.ts`) that would otherwise be equally valid
candidates.

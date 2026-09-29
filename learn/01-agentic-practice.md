# 01 — Agentic practice: direct an agent in someone else's codebase

This is the READ tier, so there are no planted incident branches. The drills use the two kinds of real material
this repo has: the **three upstream candidates** (`training/navigation/02-upstream-contribution-kit.md` §4) and
the **fabledocs4 fire drills** (`fabledocs4/01-fire-drills.md`).

Every drill has the same loop:

1. **You** write the spec: goal, files it may touch, files it must not touch, the acceptance test, and the
   out-of-scope list. Keep it to about 15 lines. Don't paste the sealed answer into it.
2. **The agent** implements it on a scratch branch (`git switch -c scratch/dN training`). Never on `master`.
3. **You** review it adversarially, using the checklist at the bottom and then the rubric in
   `learn/_answers/01-agentic-practice.md`. Record how many findings you caught before you opened the rubric.
4. Throw the scratch branch away (`git switch training && git branch -D scratch/dN`). Only D1 and D3 ever
   become a real upstream PR, and only through `training/navigation/04-upstream-pr-dry-run.md`.

---

## D1 — Candidate B: first test for `DocumentUpdateTextTask` (60–90 min)

**Target:** `server/queues/tasks/DocumentUpdateTextTask.ts` has no test, while its siblings do (for example
`CleanupDocumentInsightsTask.test.ts`). Write a spec that gets an agent to add
`server/queues/tasks/DocumentUpdateTextTask.test.ts`.

Your spec must decide and state:
- whether the test calls `.perform(event)` or `.schedule(event)`, and why;
- how the `DocumentEvent` is built (look at what `RevisionsProcessor.ts:46` passes);
- which branches are covered (see `learn/00-onramp.md` E8);
- what the agent must **not** touch (the task itself, `server/test/factories.ts` except to reuse it, and any
  new test directory).

**Adversarial review focus:** does each assertion actually prove something? Does the "happy path" check real
markdown, or only that `text` is truthy? What does it do about the `language = undefined` edge case?

**[needs stack]** Run it with `yarn test server/queues/tasks/DocumentUpdateTextTask.test.ts`. That needs
Postgres. Docker is currently down, so until then, review statically against the rubric.

## D2 — Candidate C: a persistence-failure metric (60–90 min)

**Target:** `PersistenceExtension.onStoreDocument` (lines 130-143) swallows failures, and
`MetricsExtension.onStoreDocument` (line 62) counts every attempt. Spec a change that makes a failed
collaborative save show up as its own metric.

The trap to design around: the obvious agent fix is to add `Metrics.increment(...)` inside the `catch` block of
`PersistenceExtension`. That works, but it spreads `collaboration.*` counters over two files, and it doesn't
touch the question of whether the error should propagate. Your spec has to make the agent:
- state **where** the counter lives and why (`MetricsExtension` owns every counter today);
- find out, with a citation to `@hocuspocus/server` source (after `yarn install`), what happens when an
  `onStoreDocument` hook throws: does the chain stop? Does Hocuspocus retry? Does it log? The agent must **not
  guess**;
- leave the transaction logic in `documentCollaborativeUpdater.ts` untouched;
- test the wiring by stubbing `documentCollaborativeUpdater` to reject, without trying to reproduce a real
  `lock_timeout`.

**Adversarial review focus:** did the agent claim a Hocuspocus behaviour without a file:line in
`node_modules/@hocuspocus/server`? That's an automatic reject.

## D3 — Candidate A: aria-label on the Viewers trigger (30 min)

**Target:** `app/components/Collaborators.tsx:167-178`. The `NudeButton` that opens the viewers popover has no
accessible name. The `PopoverContent` on line 180 already has `aria-label={t("Viewers")}`.

Spec a **one-line** change. Name the three things the agent is likely to do that you will reject:
- inventing a new translation key or editing locale JSON by hand (`AGENTS.md`: "Do not add translation strings
  manually");
- adding a component-test harness (`app/components/` has zero `*.test.tsx`, so that would be a separate
  precedent-setting PR);
- "improving" `Facepile` or the avatars while it's in there.

**Verification [needs stack]:** with a screen reader or the browser's accessibility tree, the trigger reads as
"Viewers, button". [Curriculum 11](../../../opusorganize/apprenticeship/curriculum/11-accessibility-essentials.md)
covers how to check an accessible name.

## D4 — Fire drill 6 → a static guard for `shared/` (60 min)

First do fire drill 6 by hand (`fabledocs4/01-fire-drills.md`, "The bundle that broke the browser"). Its
lesson is "make the machine hold the line". Checked at `37ca0d5`: `.oxlintrc.json` has **no** rule that stops
`shared/**` from importing `@server/*` or Node built-ins. Its only `no-restricted-imports` entry is about
`prosemirror-tables`.

Spec a lint-config change that makes `yarn lint` fail when a file under `shared/` imports `@server/...`, `fs`,
`path`, `crypto` or `node:*`.

**Adversarial review focus** (look at the actual `.oxlintrc.json` structure, with `overrides[0]` at lines 69-71):
- Does the new rule **replace** the existing `prosemirror-tables` restriction, or add to it? If the agent adds
  a second `no-restricted-imports` for `shared/**`, check whether oxlint merges the options or overrides them.
  Cite the oxlint docs, don't guess.
- Does a naive `crypto*` pattern flag `shared/utils/color.ts:1`, `import md5 from "crypto-js/md5"`, which is a
  legitimate browser library?
- Proof that the rule works: plant a violation on the scratch branch
  (`import env from "@server/env"` in a `shared/utils` file) and show that `yarn lint` fails. Then revert.
  **[needs `yarn install`]**

## D5 — Fire drill 2 → audit an agent's code search (45 min)

First do fire drill 2 by hand (the pool drain). Then delegate a **read-only** investigation: "List every code
path in `server/` that takes a `FOR UPDATE` row lock on a `documents` row, with file:line and the caller."

Your job is to catch what the agent's list gets wrong. Search yourself:
`grep -rn "LOCK.UPDATE" server --include=*.ts | grep -v test`.
Then open each hit and decide which model it locks. Typical agent failures: filtering hits by the word
"document" (which misses `PersistenceExtension.ts:55`, because that line never says "document"), counting
locks on `collections` as document locks, and inventing call sites.

**Exit:** your corrected list, with a false-positive and false-negative count for the agent's version.

---

## Verification checklist: before you trust agent output in Outline

Tick every box, or write down why it doesn't apply.

**Gates** (`.github/workflows/ci.yml`)
- [ ] `yarn lint` (oxlint, type-aware, `package.json:16`) and `yarn tsc` both run on every PR, so run both locally.
- [ ] The test job you need is decided by the path filter: `app/**` or `shared/**` → `test:app` + `test:shared`; `server/**` or `shared/**` → `test-server` (Postgres 14.2 service only; Redis is `ioredis-mock` via `server/test/setupMocks.ts:37`). Run the matching `yarn test <file>` locally (`AGENTS.md` prefers single-file runs).
- [ ] `package.json` or `yarn.lock` changed? Then the `audit` job runs too. An agent should almost never touch these in a first PR.

**`AGENTS.md` rules agents break most often**
- [ ] No `any`, no `as`/`!` assertions, `interface` rather than `type` for object shapes, curly braces on every `if`.
- [ ] JSDoc on every exported function or class (`@param` lowercase, ending with a period).
- [ ] No new `.md` files and no new test directories. Tests are collocated `*.test.ts`.
- [ ] No hand-added translation strings. Event handlers are named `handleX`. Named exports for new classes.

**Invariants in this codebase** (break one and CI stays green)
- [ ] Internal projection writes keep `hooks: false` (`documentCollaborativeUpdater.ts:109-111`; the comment warns of infinite processing) and `silent: true` where present (`PersistenceExtension.ts:90-91`).
- [ ] The hot save path keeps `.scope("withoutState")` and `SET LOCAL lock_timeout = '15s'` (`documentCollaborativeUpdater.ts:35-41`).
- [ ] `shared/` code imports nothing from `@server/*` or Node built-ins (ADR-002; not lint-enforced today, see D4).
- [ ] The two no-op guards (the empty Redis set, and `isEqual` on canonical content) both survive.
- [ ] The durability of `Event.schedule` vs `Event.create` is unchanged unless the PR is explicitly about that (ADR-006).

**Security spots**
- [ ] Data access still goes through `authorize(...)` (`server/policies/cancan.ts:182`) or a loader that calls it (`server/commands/documentLoader.ts`).
- [ ] The extension order in `server/services/collaboration.ts:48-69` is unchanged: auth before persistence.
- [ ] No secrets from `.env*`, and nothing from `learn/`, `training/` or `fabledocs*/`, in the diff.

**Honesty**
- [ ] Every claim the agent makes about library behaviour (Hocuspocus, Sequelize, franc) has a file:line in `node_modules` or a doc link. Otherwise it is marked unverified in the PR description.

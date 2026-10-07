# learn/ — How to study Outline as a junior (B08)

> Local study material only. `AGENTS.md` says "Critical – Do not create new markdown (.md) files", so
> `learn/`, `training/` and `fabledocs*/` must **never** appear in an upstream PR. In
> `elirc/masteroutline-outline`, `training` is the default branch and holds all of them; `master` has
> `fabledocs*/` and `training/` but not `learn/`. Either way, branch off upstream Outline, not this
> repository, when you prepare a real PR.

## What this app is and why it's in the portfolio

Outline is a real, actively maintained open-source wiki: a Koa + Sequelize API (`server/`), a React + MobX SPA
(`app/`), code shared by both runtimes (`shared/`), and live co-editing through Yjs/Hocuspocus
(`server/collaboration/`). This repo is a clone of upstream, not your own app. It is in the portfolio at the
**READ** tier for two reasons. First, it teaches you to find your way around a codebase you did not write, at a
size (2,600+ files) you can't hold in your head. Second, it is where your **first real upstream PR** comes from
(`training/navigation/02-upstream-contribution-kit.md`). It already has four strong study packs
(`fabledocs/`, `fabledocs2/`, `fabledocs3/`, `fabledocs4/`). This folder tells you the order to use them in, and
what to do by hand versus what to hand to a coding agent.

**Runtime status when written (2026-09-26):** `node_modules` is not committed, so run `yarn install` first. The
dev stack needs Postgres + Redis (`docker-compose.yml`); on the author's machine Docker was unavailable at the
time. Every step below marked **[needs stack]** needs that stack running. Everything else is reading and
writing, and you can do it without one.

## Prerequisites (do these first)

The "curriculum NN" entries name chapters of a separate apprenticeship curriculum that is **not in this
repository**. If you don't have it, the second half of each row (the fabledocs reading) still covers the idea.

| # | Fundamental | Read |
|---|---|---|
| 1 | CRDTs: why a Y.Doc merges without conflicts, and state vs projections | curriculum 17 — CRDTs & offline sync (outside this repo), then `fabledocs2/README.md` "ground rules" (rule 5) |
| 2 | Commit-then-publish gaps (why an event can be lost after the DB commits) | curriculum 01 — transactional outbox (outside this repo), then `fabledocs3/adr-006-event-spine.md` |
| 3 | Row locks and lock timeouts | curriculum 03 — concurrency races (outside this repo) (the locking half), then `fabledocs4/04-sql-concurrency-lab.md` Experiment 1 (read it even if you can't run it) |
| 4 | Telemetry that lies by omission | curriculum 09 — telemetry you can trust (outside this repo) (needed for upstream candidate C) |
| 5 | Pinning current behaviour before you change it | curriculum 04 — characterization tests (outside this repo) (needed for candidate B and on-ramp E2) |
| 6 | Verifying code you didn't write | curriculum 05 — verify before trust (outside this repo) and curriculum 11 — accessibility (outside this repo) (candidate A) |

## Study route (11 sessions, ~12 h)

Legend: **HAND** = you do it yourself, no agent. **DELEGATE** = give it to a coding agent, then verify what it
did against the exit criterion.

| # | Session (time) | Files / material | HAND | DELEGATE → verify | Exit criterion |
|---|---|---|---|---|---|
| 1 | **Orient** (60 min) | `AGENTS.md`, `docs/ARCHITECTURE.md`, `fabledocs/00-fast-track.md`, `fabledocs2/README.md` | Read. Sketch `app/ server/ shared/ plugins/` and the service flags (`fabledocs3/adr-001-...`) from memory | Ask an agent to list which processes `yarn dev:watch` starts (`package.json:15`). Check its answer against the script | Without notes, you can draw the 4 top-level trees and say which runtime(s) each one runs in |
| 2 | **Trace one HTTP request** (60 min) | `server/routes/api/documents/documents.ts:573-651` (`documents.info`), `server/commands/documentLoader.ts`, `server/policies/document.ts` | On-ramp **E3** (trace), **E1** (predict) | none. Tracing is the skill here | You can name, with file:line, where each of the three failures in E3 is thrown |
| 3 | **Pin behaviour** (45 min) | `shared/utils/parseDocumentSlug.ts` + its `.test.ts` | Write the test cases for **E2** | Have an agent turn your cases into Vitest syntax. Reject it if it "fixes" the function | **[needs `yarn install`]** `yarn test shared/utils/parseDocumentSlug.test.ts` passes with your new negative cases. Without deps: your expected values match `_answers/00-onramp.md` |
| 4 | **The triple representation** (75 min) | `fabledocs2/03-data-access.md`, `fabledocs3/adr-007-crdt-triple-representation.md` | On-ramp **E5** | none | You can explain, without notes, who writes `state`, `content` and `text`, and when |
| 5 | **Collaborative save path** (75 min) | `training/navigation/01-collaborative-save-path-trace.md` with `server/services/collaboration.ts`, `server/collaboration/PersistenceExtension.ts` and `server/commands/documentCollaborativeUpdater.ts` open | On-ramp **E4**, **E6** | none | You can answer E6 part (b): which guard stops a no-op write after the Redis set is already populated, and why |
| 6 | **Event spine** (60 min) | `server/models/Event.ts` (`schedule`, ~line 136), `server/queues/processors/DebounceProcessor.ts`, `RevisionsProcessor.ts`, `server/queues/tasks/DocumentUpdateTextTask.ts` | On-ramp **E7**, **E8**, **E9** | none | Your E9 timeline matches the sealed answer, including the `updatedAt` vs `createdAt` reasoning |
| 7 | **Fire drills, round 1** (60 min) | `fabledocs4/01-fire-drills.md` Drill 2 (pool drain) + Drill 5 (stale search) | Write your hypothesis and 3 diagnostic moves **before** Hint 1. Write the 3-sentence status post | Agentic drill **D5** (have an agent list the lock sites, then audit its list) | Your status post names the same root cause as the "Resolution path" |
| 8 | **Fire drills, round 2** (60 min) | Drills 1, 3, 4, 6 (at least two of them) | Same no-peeking rules | Agentic drill **D4** (static guard for `shared/`) | For each drill: one prevention you'd actually ship, with the file it would live in |
| 9 | **Upstream kit** (60 min) | `training/navigation/02-upstream-contribution-kit.md` §4, then `training/_answers/02-change-candidates-answers.md` | Do all 3 placement exercises before opening the answers | none | For each candidate you wrote file, test, and "what CI wouldn't catch", and graded yourself against the sealed answer |
| 10 | **Direct an agent on a real candidate** (90 min) | `learn/01-agentic-practice.md` D1 (test for `DocumentUpdateTextTask`) and D3 (aria-label) | Write the spec. Do the adversarial review | Implementation | **[needs stack]** `yarn test server/queues/tasks/DocumentUpdateTextTask.test.ts` is green, and every checklist item is ticked. Until the stack runs: a review against the D1 rubric with 0 unresolved findings |
| 11 | **PR dry run → real PR** (90 min) | `training/navigation/02-upstream-contribution-kit.md` §3 (first-PR checklist). A separate `04-upstream-pr-dry-run.md` was planned but is not in this repository | Check the candidate against live upstream issues. Write the issue + PR description yourself | Nothing. Your name goes on it | A PR description that passes the dry-run rubric. Stretch: a real PR opened on github.com/outline/outline |

**After the route (optional, multi-day, [needs stack]):** `fabledocs4/03-capstone-collab-harness.md` (build the
multi-client harness for the path you traced in session 5) and `fabledocs4/04-sql-concurrency-lab.md`.

**Blind review and interview kit (READ tier, no `training/review/` here):** for blind code reading, use
`fabledocs/04-code-reading-gym/`. For interviews, use `fabledocs3/interview-translation.md`,
`fabledocs/08-interview-prep/`, and the "Interview angle" section at the end of `training/navigation/01-...`.
`fabledocs4/07-four-week-plan.md` is the long version of this route. Use it if you want 4 weeks here rather than
~2.

## Do vs. delegate — the rule for this repo

- **Always by hand:** reading traces, fire-drill diagnosis, placement decisions, specs, adversarial review, the
  PR description. In this repo the scarce skill is knowing **where** a change goes and **what it must not break**
  (the `hooks: false` writes, the `withoutState` scope, `lock_timeout`, the `shared/` runtime boundary). An agent
  can't learn those for you.
- **Delegate:** typing test boilerplate in the house style, grep sweeps (then audit them, see D5), lint-config
  edits (then plant a violation to prove the rule fires).
- **Never delegate:** deciding that a candidate is still open upstream, or writing the words a maintainer reads.

## Correction to earlier training material

`training/navigation/02-upstream-contribution-kit.md` §1 says the `test-server` CI job needs "a live Postgres
14.2 + Redis service container". In `.github/workflows/ci.yml` that job declares **only** a `postgres:14.2`
service. Redis is replaced in tests by `ioredis-mock` (`server/test/setupMocks.ts:37`). So locally, server tests
need Postgres but not Redis.

## How you know you're done here

- On-ramp E1–E9 are graded, and you have at least 7 of 9 fully right.
- At least 4 of the 6 fire drills are done cold, each with a 3-sentence status post.
- All 3 placement exercises are graded, and at least one agentic drill has passed its rubric.
- An issue and PR description for one candidate pass the dry-run rubric (a real PR is the stretch goal).

**Next in the competency graph:** **folio** (the CRDT LAB, with 5 measured incidents and a property-test ticket).
There you break and fix the kind of sync engine you only read about here. After that, go back to the
gitjira-app/atlas "execute" track (real deploys).

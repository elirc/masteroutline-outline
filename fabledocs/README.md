# Fabledocs: Outline as a Training Lab

A curriculum that turns this repository — [Outline](../README.md), a real-time collaborative knowledge base — into a structured path from junior fullstack JS engineer to mid-level, with senior judgment and interview readiness as explicit goals.

**Who this is for:** you can build CRUD features in React/Node/TypeScript, but you want to (1) navigate a large production codebase confidently, (2) develop the judgment mid-level and senior engineers are paid for, and (3) walk into a mid-level fullstack interview with concrete, evidence-backed answers instead of memorized trivia.

**The method:** every page pairs *codebase-specific knowledge* (where things live in Outline, how its real flows work) with *transferable skill* (why the pattern exists, what the alternatives are, how it fails, and how to talk about it in an interview). Interview readiness is a first-class goal — most pages end with an "Interview angle."

## What Outline is (the 60-second version)

Outline is a team wiki/knowledge base: documents organized into collections, edited collaboratively in real time. It is a TypeScript monorepo with three main source trees: [`app/`](../app) is a React 17 + MobX single-page client; [`server/`](../server) is a Koa API server using Sequelize over PostgreSQL, Redis + Bull for background work, socket.io for live updates, and Hocuspocus/Yjs for collaborative editing; [`shared/`](../shared) holds code used by both, including the entire ProseMirror-based editor. The API is RPC-over-POST (e.g. `POST /api/documents.update`), authorization is cancan-style policies, and every mutation emits an event onto a Redis-backed event bus that fans out to processors (revisions, notifications, websockets, search, webhooks). It ships as a single deployable that can run one or many of its services (`web`, `worker`, `websockets`, `collaboration`, `cron`, `admin`) per process — see [server/index.ts](../server/index.ts#L182-L190). It is multi-tenant: nearly every table has a `teamId`, and tenant isolation is a recurring security theme in this curriculum.

## Structure

| Module | What it gives you |
| --- | --- |
| [00-fast-track.md](00-fast-track.md) | A weekend: run it, trace two flows, make one safe change |
| [01-codebase-cartography/](01-codebase-cartography/README.md) | Maps: system, reading order, glossary, tooling, 6 key flows traced end to end |
| [02-stack-and-language-mastery/](02-stack-and-language-mastery/README.md) | JS/TS/Node/React mental models, anchored to real files here |
| [03-architecture-and-patterns/](03-architecture-and-patterns/README.md) | Layers, data model, authz, async reliability, a pattern catalog, and an architecture critique |
| [04-code-reading-gym/](04-code-reading-gym/README.md) | Drills: annotation, trace tables, bad-vs-better contrasts, review katas |
| [05-quality-engineering/](05-quality-engineering/README.md) | Testing strategy, test recipes, systematic debugging, performance, security, observability |
| [06-contribution-practice/](06-contribution-practice/README.md) | Junior tickets → mid-level features → senior projects and katas |
| [07-career-and-collaboration/](07-career-and-collaboration/README.md) | Code review mindset, PRs/RFCs, maintainer communication |
| [08-interview-prep/](08-interview-prep/README.md) | 50+ question cards (mostly repo-anchored), system design from this repo, STAR stories, a two-week cram plan |
| [09-reference/](09-reference/command-cheatsheet.md) | Command cheatsheet, risk register, rubrics, verification log |

## How to use it

- **One weekend:** [00-fast-track.md](00-fast-track.md) only.
- **Two weeks (learning):** fast track → cartography (all) → key flows drills → pattern catalog → testing strategy + one test recipe → 3 junior tickets from [06-contribution-practice/01-good-first-tickets.md](06-contribution-practice/01-good-first-tickets.md).
- **Two weeks (interview in 14 days):** go directly to [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md); it pulls in exactly the prerequisite pages you need each day.
- **Eight weeks:** modules in order, one per ~5 days, doing every drill; ship at least two real PRs from module 06 in weeks 6–8.
- **Ongoing contribution:** keep [09-reference/risk-register.md](09-reference/risk-register.md) and [06-contribution-practice/](06-contribution-practice/README.md) open; treat everything else as reference.

Recommended entry points by profile:

- **Brand-new junior:** 00 → 01 → 02 → 04 (drills as you read) → 06 junior tickets.
- **Junior who knows React/Node:** 00 → 01/05-key-flows → 03 → 04 → 06.
- **Mid-level, new to this repo:** 01/01-system-map + 01/05-key-flows → 03/06-architecture-critique → 06 mid-level tickets.
- **Senior doing an architecture review:** 03/06-architecture-critique → 09/risk-register → 05/05-security-checklist.
- **Candidate, interview in two weeks:** [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md).

## Conventions used throughout

- **File anchors.** Every claim about this codebase links to a path, usually with line numbers: [server/middlewares/transaction.ts](../server/middlewares/transaction.ts#L13-L20). Line numbers were verified at commit `8e032f8cd` and will drift; the function name in the surrounding text is your fallback. If an anchor looks wrong, `grep` the symbol.
- **Fake code is labeled.** Any snippet not from this repo starts with `// Illustrative fake code: not from this repo`. Unlabeled excerpts are real (possibly trimmed, marked with `…`).
- **Verification labels.** Commands are marked **verified** (actually executed) or **inferred** (read from scripts/config but not run). Behavior claims that weren't fully traced are labeled *investigate* or *possible risk*, never asserted as bugs. See [09-reference/verification-log.md](09-reference/verification-log.md).
- **Drills and self-grading.** Most pages end with a drill and a Basic/Solid/Strong rubric. Grade yourself honestly; "Strong" descriptions are what a senior answer sounds like.
- **Senior vocabulary.** Words like *invariant*, *boundary*, *contract*, *idempotency*, *isolation*, *blast radius* are used deliberately and defined in context the first time each module needs them. They are also interview vocabulary — using them precisely (and only where they apply) is a mid-level signal.

## The other study trees in this repo

This is the broadest of six AI-written study trees on the same snapshot, and none of the others is linked from here:

- `fabledocs2/` — six narrative chapters (architecture, API design, data access, testing, error handling, observability).
- `fabledocs3/` — ten ADRs (`adr-001` … `adr-010`) plus `interview-translation.md`.
- `fabledocs4/` — practice: six fire drills, two capstones, an SQL concurrency lab, a MobX lab, 43 flashcards, a four-week plan.
- `training/` — a call-by-call trace of the collaborative save path, an upstream-contribution kit, and verdicts on the four `fabledocs*` trees.
- `learn/` — an 11-session route through all of the above, with on-ramp exercises and sealed answers in `learn/_answers/`.

If you want one ordered path through everything, follow `learn/README.md`; it uses this tree for orientation and the code-reading gym.

## A note on repo rules

This repo's own [AGENTS.md](../AGENTS.md) instructs contributors not to create new markdown files in the product tree — these docs exist as a self-contained learning suite in `fabledocs/` at the explicit request of the repo's local user, and nothing here modifies product code. Don't ship `fabledocs/` in a PR to upstream Outline.

## The mindset ladder

- A **junior** asks: *how do I make it work?*
- A **mid-level** engineer asks: *is this the right pattern? What does it cost, and how do I know it's correct?*
- A **senior** asks: *what does this commit us to? Who pays the cost later, and how do we reduce the risk now?*

Mid-level interviews test the second and third question, not the first. When a page here asks "what would you check before merging this?" or "what breaks at 10x?", that is not academic — it is the literal shape of the interview. Practice answering out loud, with file paths.

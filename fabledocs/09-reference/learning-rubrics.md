# Learning Rubrics

Observable behaviors, not vague traits. Use for self-assessment monthly and before interviews. "Interview-ready" = you can demonstrate the mid column *out loud, with this repo's examples, under time pressure*.

## Skill rubrics

| Skill | Junior (locate/explain) | Mid (change safely) | Senior (design/derisk) | Interview-ready check |
| --- | --- | --- | --- | --- |
| Codebase navigation | Finds the file for a feature via grep in minutes | Predicts the file list for a change before looking (±3 files) | Knows which files are load-bearing vs incidental; reads diffs of unfamiliar areas fast | Narrate "where would X live" for 3 random features, correctly, aloud |
| API design | Adds an endpoint copying a sibling | Designs request/response with validation, authz, pagination, rate-limit decisions recorded | Evolves public contracts (versioning, defaults, deprecation) without breakage | Deliver [API cards Q2/Q5/Q8](../08-interview-prep/03-api-and-data-modeling-questions.md) at Solid+ |
| Authorization | Uses `authorize()` where siblings do | Constructs membership topologies in tests; explains policy composition | Designs grant models; spots IDOR in review reliably ([R1](../08-interview-prep/05-debugging-and-code-review-rounds.md) pass) | Explain the [gauntlet](../03-architecture-and-patterns/03-validation-auth-and-permissions.md) unprompted in <3 min |
| Data/persistence | Writes a working migration | Expand/migrate/contract; threads transactions correctly; knows paranoid pitfalls | Plans backfills/cutovers; matches consistency guarantees to stakes | [K-6 kata](../06-contribution-practice/04-refactor-and-design-katas.md) at Strong |
| Async/reliability | Schedules a task following a sibling | Writes idempotent consumers with run-twice tests | Designs pipelines with failure visibility; can find and close delivery gaps | Tell the outbox story ([Story 2](../08-interview-prep/06-behavioral-star-stories.md)) in 2 min |
| Frontend architecture | Builds components against existing stores | Places state correctly; debugs reactivity; handles loading/error states systematically | Designs cache-consistency strategy (push/pull/resync); a11y and bundle discipline | [Frontend cards Q1/Q6/Q8](../08-interview-prep/02-frontend-framework-questions.md) at Solid+ |
| Testing | Writes happy-path tests with factories | Right layer per risk; negative/permission cases by default; anti-flake habits | Shapes the strategy (what's mocked and why); makes contracts testable | Recite the [7 recipes](../05-quality-engineering/02-writing-tests-here.md) categories + when each applies |
| Debugging | Reproduces and bisects with guidance | Binary-searches boundaries, cheap probes first, narrates hypotheses | Designs the observability that makes the next occurrence self-evident | D-rounds 11+/15 twice |
| Review | Spots style and obvious bugs | Finds correctness/security issues; calibrates severity honestly | Reviews designs, not just diffs; negotiates requirements | R-rounds 11+/15 twice |
| Communication | Asks answerable questions | PRs pre-answer reviewer questions; disagreement with evidence | RFCs a reader can specifically disagree with; escalates decisions not people | Your MT design note passes the [RFC drill](../07-career-and-collaboration/02-writing-prs-and-rfcs.md) |

## Self-assessment checklists

**Am I operating at mid-level?** (yes to ≥8)
- [ ] I write the design note *before* the code, and it predicts the hard part
- [ ] My mutations always ship with a permission-failure test
- [ ] I can explain why each of my queries is inside/outside the transaction
- [ ] I check "safe to run twice?" for anything async I write
- [ ] I know which of my changes are contract changes, and treat them differently
- [ ] I measure before optimizing, and put numbers in the PR
- [ ] My review comments carry severity labels and anchors
- [ ] I can say "I don't know, here's how I'd find out" without flinching
- [ ] I can trace a request through every layer of this repo from memory
- [ ] I've disproven at least one of my own confident claims with evidence

**Am I building senior judgment?** (progress, not pass/fail)
- [ ] I argued *against* my own refactor at least once (K-1 style) and meant it
- [ ] I've written a rollback plan that would actually work
- [ ] I've chosen to *not* fix something, with a written reason (stakes vs cost)
- [ ] I've identified who pays the cost of a design later, and said so out loud
- [ ] I've escalated a decision that wasn't mine to make

## Calibration ritual

Monthly: pick one row, do its interview-ready check cold, grade honestly, log the result *in this file's margin or your notes*. Skill you don't re-measure decays silently — the rubric is an instrument panel, not a diploma.

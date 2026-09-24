# training/ — B03 Navigation Pack (READ tier)

This repo already has an excellent, pre-existing curriculum in
`fabledocs/`, `fabledocs2/`, `fabledocs3/`, `fabledocs4/` (see
`navigation/03-existing-curriculum-verdicts.md` for a KEEP verdict on all
four and a recommended reading order across all five trees including this
one). This `training/` tree does **not** duplicate any of it. It adds
exactly three things those four packs don't have:

| File | What it adds | Time |
| --- | --- | --- |
| [`navigation/01-collaborative-save-path-trace.md`](navigation/01-collaborative-save-path-trace.md) | File-by-file, call-by-call trace of the collaborative-editing save path (Yjs/Hocuspocus → persistence → event fan-out → text projection refresh), one level more granular than `fabledocs3/adr-007-crdt-triple-representation.md`, which it explicitly builds on rather than restates. | ~20 min |
| [`navigation/02-upstream-contribution-kit.md`](navigation/02-upstream-contribution-kit.md) | This repo's actual contribution rules (AGENTS.md, CI gates), a method for vetting a genuine upstream good-first-issue (not just trusting the label), a first-PR checklist, and 3 real, evidence-backed candidate change areas in this exact codebase with placement exercises. | ~25 min reading, +30 min if you do the placement exercises before peeking at answers |
| [`navigation/03-existing-curriculum-verdicts.md`](navigation/03-existing-curriculum-verdicts.md) | KEEP/MERGE/CONVERT/IGNORE verdicts on all four existing trees, and the recommended order through all five (four existing + this one). | ~5 min |
| [`_answers/02-change-candidates-answers.md`](_answers/02-change-candidates-answers.md) | Sealed placement answers for the 3 contribution-kit candidates. Don't open before attempting your own answer. | — |

## Suggested order

1. Read `navigation/03-existing-curriculum-verdicts.md` first — it tells
   you where this tree fits relative to the other four and gives you the
   full five-tree reading order.
2. If you're following that order: read `fabledocs/`, `fabledocs2/`,
   `fabledocs3/` (specifically through ADR-007) first, *then*
   `navigation/01-collaborative-save-path-trace.md`, then
   `navigation/02-upstream-contribution-kit.md`, then continue into
   `fabledocs4/`.
3. If you only have 45 minutes: read `01-collaborative-save-path-trace.md`
   and `02-upstream-contribution-kit.md` in order — the trace assumes
   ADR-007's vocabulary (triple representation, lazy conversion, debounced
   persistence) so skim `fabledocs3/adr-007-crdt-triple-representation.md`
   first if you haven't read it.

## Scope note (READ tier)

Per this batch's tier assignment, this project got Phase 4 (codebase
navigation) only — no ladder, no seeded incidents, no agentic/interview
kit. No source files were edited; this tree, `training/`, is the only
addition. See the batch report for verification notes and caveats:
`C:\Users\Owner\Desktop\opusorganize\batch3report\projects\outline.md`.

## Repo baseline

This repo had no `.git` when this pass started. A baseline snapshot was
committed first (`chore: baseline snapshot before training`), respecting
the existing `.gitignore` (no `node_modules`, no local `.env`/`.env.local`/
`.env.production`). This `training/` tree was committed separately on top
of that baseline. See the batch report for both commit hashes.

# Verdicts on the Existing fabledocs Trees + Recommended Order

Four curriculum trees already exist in this repo (`fabledocs/`,
`fabledocs2/`, `fabledocs3/`, `fabledocs4/`), all pre-existing and
excellent — none of them are touched or duplicated by this `training/` tree.
This page is the navigation aid the B03 brief asks for: verdicts, and the
order a learner should actually move through five curricula (four existing
+ this one) without redundant passes.

## Verdicts

| Tree | What it is | Verdict | Why |
| --- | --- | --- | --- |
| `fabledocs/` | 10-module ladder (00 fast-track → 09 reference): cartography, stack mastery, architecture, code-reading drills, quality engineering, contribution practice, career/collab, interview prep (50+ Qs) | **KEEP** | Most complete of the four; has its own graded drills, a two-week interview cram plan, and an explicit AGENTS.md-compliance note (doesn't ship to upstream). Nothing in this `training/` tree substitutes for it. |
| `fabledocs2/` | 6-doc "engineer who built this" onboarding narrative (architecture, API design, data access, testing, error handling, observability) | **KEEP** | Distinct voice/format (first-person day-one onboarding) from `fabledocs/`'s structured curriculum — genuinely complementary, not redundant. Its "ground rules" section (5 rules) is the single best 1-page first-change primer in the repo; this `training/` tree's save-path trace (`01-collaborative-save-path-trace.md`) extends its rule 5 ("Yjs state is authoritative, text/content are projections") with file:line mechanics rather than re-explaining it. |
| `fabledocs3/` | 10 reverse-engineered ADRs + interview-translation | **KEEP** | Rigorous, evidence-cited, and the load-bearing reference for this `training/` tree — `01-collaborative-save-path-trace.md` explicitly builds on ADR-007 rather than restating it. Don't merge into `curriculum/`: it's Outline-specific by construction (each ADR is inferred from *this* repo's scars). |
| `fabledocs4/` | Practice pack: fire drills, 2 capstones (outbox relay, collab harness), SQL concurrency lab, MobX frontend lab, flashcards, 4-week plan | **KEEP** | The only tree with hands-on labs requiring a running stack; its capstone 3 (collab harness, `03-capstone-collab-harness.md`) is literally "build the test harness this save path is missing" — a natural next step after `01-collaborative-save-path-trace.md` in this tree. Not merged or converted; it already is the exercise layer `fabledocs`/`fabledocs2`/`fabledocs3` lack. |

**No MERGE or CONVERT verdicts.** Nothing here belongs in the shared
`apprenticeship/curriculum/` — all four trees (and this one) are grounded in
Outline's specific files, ADRs, and scars by design; a central curriculum
page would either genericize them (against the master prompt's rule 3) or
duplicate what's already project-specific and well-written here. See the
report file's "curriculum needs" section for the one exception: a *link*,
not a merge.

## Recommended order (five trees, no redundant passes)

1. **`fabledocs/00-fast-track.md`** — the weekend orientation. Run it, trace
   two flows, make one safe change. Do this first regardless of your goal.
2. **`fabledocs2/` in full** (6 docs, in order) — the narrative layer. Each
   doc's "three questions" at the end are worth actually answering in
   writing before moving on; it's built to require that.
3. **`fabledocs3/` in full** (10 ADRs + interview-translation) — now that
   you have the narrative, the ADRs give you the *why* and the scars behind
   each ground rule from step 2.
4. **This tree's `01-collaborative-save-path-trace.md`** — right after
   ADR-007 specifically (don't wait until you've read all 10 ADRs if
   collaboration is your focus; this page only assumes ADR-007's
   vocabulary). Then **`02-upstream-contribution-kit.md`** — best read once
   you've seen enough real files (steps 1-3) to recognize why the three
   candidates are plausible and not toy examples.
5. **`fabledocs4/`** — do the fire drills first (`01-fire-drills.md`, no
   peeking), then the labs and capstones. Capstone 3
   (`03-capstone-collab-harness.md`) is the natural "now build it" step
   after this tree's save-path trace — you'll be writing the multi-client
   test harness for exactly the flow you just traced file-by-file.
6. **`fabledocs/`'s remaining modules** (01-codebase-cartography through
   09-reference), used as reference/drill material throughout rather than
   read linearly a second time — its own README already gives per-profile
   entry points (brand-new junior vs. mid-level vs. interview-in-two-weeks);
   defer to that rather than a second full read here.

Total added reading/practice time for this `training/` tree specifically:
~45-60 minutes for the save-path trace + contribution kit (reading only;
the placement exercises add ~30 min if done seriously before checking the
sealed answers). It's designed to slot into step 4 above without requiring
a re-read of any existing pack.

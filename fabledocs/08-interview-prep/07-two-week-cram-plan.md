# Two-Week Cram Plan

For the candidate with a mid-level fullstack interview in ~14 days. Assumes you can run the repo (do [00-fast-track.md](../00-fast-track.md) §1 on day 1 if not). Daily load: ~2.5–3.5 hours. Everything out loud that can be out loud.

**Principles:** cards beat re-reading; spaced repetition beats cramming (hence the revisit slots); one mock per skill under time pressure beats five casual passes.

## Week 1 — build the evidence base

**Day 1** — Fast track §§1–3: run the app, trace Flow 1 and Flow 2 aloud ([key flows](../01-codebase-cartography/05-key-flows.md)). Evening: read the [system map](../01-codebase-cartography/01-system-map.md), draw it from memory.
**Day 2** — [JS/TS/Node cards](01-js-ts-node-deep-dive.md) Q1–Q7 aloud (90s each, two passes). Then [annotation drills](../04-code-reading-gym/01-annotation-drills.md) 1–5.
**Day 3** — Remaining JS cards Q8–Q14. Annotation drills 6–10. Evening: [glossary](../01-codebase-cartography/03-domain-glossary.md) self-quiz.
**Day 4** — [API/data cards](03-api-and-data-modeling-questions.md) Q1–Q6 — the money round. Read [validation/auth/permissions](../03-architecture-and-patterns/03-validation-auth-and-permissions.md) first (30 min).
**Day 5** — API/data Q7–Q12. Read [side-effects/reliability](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md) (30 min); trace Flow 5 aloud.
**Day 6** — [Frontend cards](02-frontend-framework-questions.md) all 12 (they're lighter). One [trace table](../04-code-reading-gym/02-trace-tables.md) (Trace 4 — error paths).
**Day 7** — **Checkpoint.** Mock yourself: 10 random cards (mix files), graded honestly. Any card below "Solid" goes on the Day 11 relearn list. Then STAR [Story 1](06-behavioral-star-stories.md) drafted and spoken twice.

## Week 2 — pressure and polish

**Day 8** — **System design day.** Read [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md) once, then deliver the full 40-minute walkthrough aloud to a wall/friend/recorder. Review recording against the grading list at the bottom of that file.
**Day 9** — Timed [debugging rounds D1 + D2](05-debugging-and-code-review-rounds.md). Evening: STAR Stories 2 and 3 drafted.
**Day 10** — Timed [review rounds R1 + R3](05-debugging-and-code-review-rounds.md). Evening: STAR Stories 4/5 (or 7/8 if you haven't done tickets), spoken.
**Day 11** — Relearn list from Day 7. Then system-design **variation prompts** 1–3 (5-minute deltas, aloud).
**Day 12** — Full behavioral pass: all your stories via the [prompt index](06-behavioral-star-stories.md), 2-minute cap each, recorded. Cut filler. Prepare your 5 questions to ask them (base two on this repo's themes: "how do you handle deploy version-skew?" is a great reverse question).
**Day 13** — **Full mock loop**, ideally with a human: 20 min cards + 40 min system design + 25 min debugging (D3, unseen until now) + 20 min behavioral. Debrief in writing.
**Day 14** — Light. Reread your own STAR bullets and the [pattern catalog](../03-architecture-and-patterns/05-pattern-catalog.md) headlines (just the "Interview angle" lines). Logistics check. Stop by early evening — fluency needs sleep more than one more pass.

## Day-7 and Day-14 self-assessment checkpoints

Score 1–5 on each; retake Day-7's row on Day 14:

| Skill | Evidence |
| --- | --- |
| Concept cards | ≥80% at Solid, your three showcase cards at Strong |
| System design | 40-min walkthrough with ≤2 dead-air moments; every choice paired with a rejected alternative |
| Debugging | D-rounds passed at 11+/15 with narration throughout |
| Review | R-rounds: blocking findings found, severity calibrated |
| Stories | 2-minute versions, concrete, impact-ending |

Anything under 4 on Day 14 → say it plainly in the interview when it comes up ("I'd verify X, I don't hold it in my head") — calibrated honesty consistently outperforms bluffing at the mid level, and interviewers test for it deliberately.

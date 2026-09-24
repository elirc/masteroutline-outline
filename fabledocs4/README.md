# fabledocs4 — Practice Pack: Drills, Capstones, Labs, and Retention

fabledocs2 told you the story. fabledocs3 gave you the decisions. This pack
is where you stop reading and start *doing* — because the difference between
"I understand the lock convoy incident" and "I can diagnose one at 3 a.m." is
reps, and the difference between knowing the outbox gap exists and being able
to close it is a capstone.

Everything here is grounded in the same seams the earlier packs identified:
the commit-to-publish gap in the event spine, the triple document
representation, the explicit-transaction discipline, the policy layer, the
missing collaboration harness, and the Redis-wears-five-hats topology.

## Contents

| File | What it is | Time |
| --- | --- | --- |
| [01-fire-drills.md](01-fire-drills.md) | Six production-incident simulations. Symptoms first, escalating hints, then the resolution path. Practice being on-call for this system. | 45–60 min each |
| [02-capstone-outbox-relay.md](02-capstone-outbox-relay.md) | Close the event spine's commit-to-publish gap with a real relay. The system's most instructive missing piece, built by you. | 2–3 days |
| [03-capstone-collab-harness.md](03-capstone-collab-harness.md) | Build the multi-client collaboration test harness — the testing strategy's standing IOU (ADR-010). | 2–3 days |
| [04-sql-concurrency-lab.md](04-sql-concurrency-lab.md) | Reproduce the lock convoy with two psql sessions. `FOR UPDATE`, `SET LOCAL lock_timeout`, `SKIP LOCKED`, pool math. | half a day |
| [05-frontend-mobx-lab.md](05-frontend-mobx-lab.md) | The `app/` tree hands-on: stores, models, shipped policies, and adding a feature end-to-end through the RPC convention. | half a day |
| [06-flashcards.md](06-flashcards.md) | ~50 spaced-repetition cards covering fabledocs2 + fabledocs3. Question / answer format, import into Anki if you like. | ongoing |
| [07-four-week-plan.md](07-four-week-plan.md) | A schedule that sequences all three packs plus this one into four weeks of ~90 min/day. | 4 weeks |

## Rules of engagement

1. **Fire drills: no peeking.** Read the symptoms, write your hypothesis and
   first three diagnostic moves *before* opening the first hint. The value is
   in the gap between your guess and the answer.
2. **Capstones: design doc first.** One page, reviewed against the acceptance
   criteria in the doc, before you write code. Both capstones are real
   engineering — treat them like a PR you'd ship.
3. **Labs need a running system.** `yarn install`, `.env` from sample,
   `yarn db:migrate`, `yarn dev:watch` (see fabledocs2 README). The SQL lab
   additionally needs two `psql` sessions against your dev database.
4. **Flashcards are for retention, not learning.** Do the reading first;
   cards keep it from evaporating.

## What this pack deliberately doesn't cover

The pre-existing `fabledocs/` curriculum (00–09) already has a code-reading
gym, stack mastery track, and career/interview material — this pack doesn't
duplicate it. And fabledocs3's interview-translation covers spoken answers;
here you build the experiences those answers describe. The honest pitch:
after this pack you're not someone who read about this architecture, you're
someone who has operated it, extended it, and broken it on purpose.

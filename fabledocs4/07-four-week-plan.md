# 07 — The Four-Week Plan: All Four Packs, Sequenced

Assumes ~90 minutes on weekdays, one longer block (3–4 hrs) each weekend.
The sequencing principle: **read → verify in code → do → retell.** Every
week ends with you saying something out loud, because the spoken version is
both the retention test and the interview deliverable.

The pre-existing `fabledocs/` curriculum (00–09) is the deep option beneath
all of this — where a day below says "go deeper," that's where you go.

---

## Week 1 — The system in your head (fabledocs2)

Goal: hold the whole architecture in working memory; running system; first
drill survived.

- **Day 1:** fabledocs2 README + get it running (`yarn dev:watch`, two
  browsers, watch the cursors — the README's day-one exercise is not
  optional). Skim the repo tree top-level.
- **Day 2:** Doc 01 (architecture) + open every file in its Evidence list.
  Answer the three exit questions in writing.
- **Day 3:** Doc 02 (API) + trace one real request in the code:
  `documents.info` from route registration through presenter. Exit questions.
- **Day 4:** Doc 03 (data access). This is the hardest one — take it slow;
  read `PersistenceExtension.ts` and `documentCollaborativeUpdater.ts`
  line by line. Exit questions.
- **Day 5:** Docs 04–06 (testing / errors / observability) in one pass —
  they're shorter and they interlock. All nine exit questions, written.
- **Weekend:** **Fire Drill 1** (poisoned processor) cold, per the rules —
  hypothesis before hints. Then start flashcards (06): full deck, first pass.
  Expect to miss half; that's the point of week 1.

## Week 2 — Decisions and their defense (fabledocs3) + first blood (labs)

Goal: argue every major decision from both sides; feel the concurrency
mechanics in your hands.

- **Day 1:** ADR-001 + 002. For each: cover the Alternatives section and try
  to *predict* the rejected options before reading them. Score yourself.
- **Day 2:** ADR-003 + 004. Same prediction game. Flashcards: misses only.
- **Day 3:** ADR-005 + 006 + 007 — the data trio. These three are the
  interview core; read the Consequences sections twice.
- **Day 4:** ADR-008 + 009 + 010. Then read the fabledocs3 README's
  "how these were reconstructed" method section — the method transfers to
  every codebase you'll ever join.
- **Day 5:** **SQL & Concurrency Lab (04)** — all six experiments. Don't
  rush Experiment 3; the convoy is the codebase's defining incident.
- **Weekend:** **Fire Drills 2 + 3** (pool drain, Redis outage) — Drill 2
  should feel like déjà vu after the lab; that's the design. First
  interview-translation pass: read three spoken answers *aloud*, then
  deliver them from memory to a wall or a friend.

## Week 3 — Build something real (capstone + frontend)

Goal: one capstone shipped end-to-end; the frontend no longer foreign.

- **Day 1:** **Frontend Lab (05)** Parts 1–2 (read + sabotage).
- **Day 2:** Frontend Lab Part 3 — ship the word-count feature, including
  the cross-team test. Narrate the request path out loud as the DoD demands.
- **Day 3:** **Outbox capstone (02) Stage 0** — the design doc. Grade it
  against the checklist honestly; revise before touching code.
- **Day 4:** Capstone Stages 1–2 (migration + relay).
- **Day 5:** Capstone Stage 3 (idempotency audit) — the table is the
  thinking; don't skip to the code.
- **Weekend:** Capstone Stage 4 (tests) + acceptance checklist. Then
  **Fire Drill 4** (Sentry flood) as a palate cleanser — it's the quickest
  drill. Flashcards: full deck second pass; you should be >80% now.

## Week 4 — The IOU, the polish, the telling

Goal: second capstone (or a scoped slice of it); every spoken answer
delivery-ready; the drills finished.

- **Day 1:** **Collab harness capstone (03) Stage 0** (recon — written
  answers with file refs) + Stage 1 started.
- **Day 2:** Stage 1 finished: harness connects two authenticated clients
  and syncs an edit. (If Node-websocket friction eats the day, that *is* the
  learning — write down the fix.)
- **Day 3:** Stage 2: tests 1–4 (echo through offline merge).
- **Day 4:** Stage 2: tests 5–6 (persistence truth, authz) + Stage 3 flake
  hunt (the 20× loop). Scope honestly: tests 1–5 solid beats 6 flaky.
- **Day 5:** **Fire Drills 5 + 6** (stale search, poisoned bundle). Drill 5
  ties together everything you now know about ADR-006 + 007 — treat it as
  the final exam it secretly is.
- **Weekend — the retelling:** All ten interview-translation answers
  delivered aloud, timed (~90s each), ideally recorded and reviewed once.
  Then the capstone stories: write and rehearse two 2-minute narratives —
  "I closed an event-delivery gap with a transactional outbox relay" and
  "I built the multi-client test harness for a CRDT collaboration server" —
  using the four-beat structure (constraint, decision, rejected alternative,
  accepted cost). Final flashcard pass, misses only.

---

## After week 4 — keeping it warm

- Flashcards: misses-only, twice a week for a month, then weekly.
- One fire drill re-run monthly, cold, from the Page line — diagnosis speed
  is perishable.
- The two capstones are portfolio pieces: clean them up on a branch with
  real commit messages and a README each. A repo you can screen-share beats
  any claim in a resume bullet.

## Am I actually done? (the honest bar)

You are done with this curriculum when all four are true:

1. You can whiteboard the full path of one keystroke — browser Y.Doc →
   Hocuspocus → debounced persist → event row → queue → processors →
   websocket broadcast → another client's sidebar — naming the file at each
   hop.
2. You can argue *for* GraphQL, RLS, Kafka, and microservices here for
   thirty seconds each, convincingly, and then say precisely why this
   codebase said no.
3. Your outbox relay passes its four tests, and you can explain to a
   skeptic why duplicates are the correct thing to choose.
4. A cold fire drill takes you under fifteen minutes from Page to a correct
   first diagnostic move.

That's not "read a codebase." That's "operated, extended, and broken it on
purpose" — which was the pitch on the tin.

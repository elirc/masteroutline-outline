# Debugging and Code Review Rounds (Timed Simulations)

Seven timed simulations converted from earlier drills. These rounds test **method under time pressure** — interviewers watch *how* you work more than whether you finish. Narrate constantly: state hypotheses, say what evidence would falsify them, announce dead ends without embarrassment.

Setup: timer visible; talk out loud; write findings as you would in a real ticket/review. Grade with the rubrics at the bottom.

---

## Debugging round D1 (25 min): stale UI after teammate's edit
Prompt as an interviewer would give it: "Users report they sometimes don't see teammates' edits until refresh. Here's the codebase. Where do you start?"
Source material: [Scenario 1](../05-quality-engineering/03-systematic-debugging.md).
Interviewer follow-ups to self-ask at 15 min: "Suppose the WS frame *does* arrive — now what?" and "How would you detect this in production before users report it?"
Pass bar: articulated the transport-vs-merge binary search within 5 minutes; named a concrete probe for each side.

## Debugging round D2 (25 min): 403 on a visible document
Source: [Scenario 2](../05-quality-engineering/03-systematic-debugging.md). Follow-ups: "The policy function is pure — so what class of bug remains?" (data loading); "Write the regression test's *name* and setup in one sentence."
Pass bar: reached "check what the policy saw, not what the rule says" unprompted.

## Debugging round D3 (30 min): missing emails
Source: [Scenario 3](../05-quality-engineering/03-systematic-debugging.md). Follow-ups: "Queues look healthy — next?" (Notification rows? SMTP? recipient filtering-by-design); "What alert prevents the next occurrence?"
Pass bar: walked the pipeline stages *in order* with an evidence check per stage; considered working-as-intended before code changes.

## Debugging round D4 (20 min): the failing test you didn't write
Simulation: `git stash` a small deliberate breakage you make yourself (e.g., flip a condition in [documentUpdater](../../server/commands/documentUpdater.ts)), run the test file, then debug from the failure output alone (pretend you didn't make the change; `git diff` is cheating).
Pass bar: from red test to exact line in <10 min using only test output + reading; bonus for narrating the "what would make this fail" reasoning before opening files.

## Review round R1 (20 min): the IDOR endpoint
Material: [Kata 1](../04-code-reading-gym/04-review-katas.md) verbatim. Deliver a written review in 20 minutes: findings labeled blocking/important/optional, each with anchor + suggested fix.
Interviewer follow-up: "The author replies 'the frontend only shows your own docs, so it's fine.' Respond." (Client checks are UX; the API is the public surface; write the two-sentence reply.)
Pass bar: IDOR + missing validation both found and correctly ranked; reply is firm, kind, and cites the house pattern.

## Review round R2 (25 min): side effects in the wrong place
Material: [Kata 3](../04-code-reading-gym/04-review-katas.md) (Slack call in `document.publish`). Follow-up: "Author says the event pipeline adds 2–5s latency and product wants instant notifications. Now what?" — the senior move is negotiating the *requirement* (is 2–5s actually unacceptable?) then, if yes, a bounded-timeout fire-and-forget *after* commit, never inside the transaction.
Pass bar: transaction-scope reasoning explicit; the latency pushback handled without abandoning the architecture.

## Review round R3 (25 min): the tempting-but-wrong cache
Material: [Kata K-8](../06-contribution-practice/04-refactor-and-design-katas.md) (per-document Redis cache of presenter output).
Pass bar: correctness kill-shot (viewer-dependent payload) delivered *first*, measurement demanded, narrower valid alternative offered. This round most reliably distinguishes mid from senior — practice it twice.

---

## Grading rubrics

**Debugging rounds:** Method (did you binary-search, state falsifiable hypotheses, use cheap probes first?) /5. Communication (could the interviewer follow your model of the system at every moment?) /5. Outcome (root cause or principled narrowing) /3. Regression thinking (test/alert proposed) /2. 11+/15 = ready.

**Review rounds:** Coverage (blocking issues found) /5. Severity calibration (no inflated nits, no deflated security) /4. Actionability (each finding has a path) /3. Tone (kind, specific, evidence-based) /3. 11+/15 = ready.

Repeat any failed round after 48 hours with fresh eyes. Two consecutive passes on each = move on; over-drilling past that point yields nothing.

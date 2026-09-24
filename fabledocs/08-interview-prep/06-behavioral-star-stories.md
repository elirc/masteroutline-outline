# Behavioral STAR Story Worksheets

Nine worksheets. Six are sourced from work this curriculum has you actually do (tickets/projects/katas) — fill them in *as you complete the work*, not the night before an interview. Three are study-based (legitimate as long as you present them honestly: "working in/studying a production open-source codebase"). Each maps to the prompts it answers and ends with a one-line resume bullet.

Rehearsal check for every story: under 2 minutes spoken? Concrete (numbers, file-level specifics)? Ends with impact + what you'd do differently? Senior-signal detail included (a tradeoff weighed, a risk reduced, a person helped)?

---

## Story 1: Learning a 400k-line codebase systematically
Prompts: "Tell me about ramping up somewhere unfamiliar" / "How do you approach complexity?" / ambiguity questions.
Source: your modules 01–03 work.
S/T: Needed to become productive in Outline — multi-tenant, real-time, 6-service TypeScript monolith — without anyone to onboard me.
A: Built a system map from entry points ([server/index.ts](../../server/index.ts)); traced six end-to-end flows before writing code; kept a verification log distinguishing verified facts from inferences; validated understanding with annotation drills against real excerpts.
R: Could locate the responsible layer for any behavior in minutes; first contributions landed without architectural rework.
Senior signal: the *method* is reusable and you can name it in steps.
Resume bullet: "Reverse-engineered and documented the architecture of a production open-source collaboration platform (Koa/React/Postgres/Redis), producing a flow-level engineering guide."

## Story 2: Finding a reliability gap and designing the fix (outbox)
Prompts: "Technical initiative" / "A risk you identified" / "Disagree with existing design?"
Source: [risk register](../09-reference/risk-register.md) event-loss analysis + [SP-1](../06-contribution-practice/03-senior-build-projects.md)/[K-2](../06-contribution-practice/04-refactor-and-design-katas.md) RFC.
S/T: Discovered post-commit event publication could silently drop notifications/webhooks on crash ([Event.ts L82–97](../../server/models/Event.ts#L82-L97)).
A: Proved the mechanism by code reading; sized likelihood honestly (rare) vs detectability (none); wrote an RFC proposing a minimal outbox relay reusing the existing events table; included the "do nothing" option costed fairly.
R: A one-page design that closes the gap with ~50 lines and one migration; the pickup-metric doubles as proof of how often the bug fired.
Senior signal: calibrated risk talk — "rare, silent, cheap to fix" — instead of alarmism.
Resume bullet: "Identified a silent event-loss window in a queue-backed architecture and designed a minimal transactional-outbox remediation."

## Story 3: The bug that was actually a data-loading problem
Prompts: "Hardest bug" / "Tell me about debugging something non-obvious."
Source: perform [debugging Scenario 2](../05-quality-engineering/03-systematic-debugging.md) for real (inject the bug, solve it).
Fill in: your actual narrowing steps, the moment you checked what the policy *saw* rather than what it *said*, the regression test you wrote.
Senior signal: "authorization bugs are usually data-loading bugs" — a portable lesson stated crisply.
Resume bullet: "Diagnosed permission failures caused by unscoped ORM loads; added topology-based regression tests."

## Story 4: Shipping a cross-layer feature safely
Prompts: "Walk me through something you built end-to-end."
Source: any completed [mid-level ticket](../06-contribution-practice/02-mid-level-feature-tickets.md) (MT-2 and MT-5 make the best stories).
Fill in: the design note you wrote first; the layer path (migration → task → presenter → store → UI); the dual-write-path hazard you designed around; expected-vs-actual file-list calibration.
Senior signal: you predicted the hard part in writing before coding.
Resume bullet: (per ticket — write it when you finish.)

## Story 5: A review that prevented an incident
Prompts: "Conflict with a peer" / "Quality bar" / "Influence without authority."
Source: [review katas](../04-code-reading-gym/04-review-katas.md), especially the Slack-in-transaction or cache-correctness scenarios — or better, a real review you gave anywhere.
Fill in: the finding, how you ranked severity honestly, the phrasing that kept the author collaborative, the outcome.
Senior signal: you negotiated the requirement ("is 2–5s latency actually unacceptable?") not just the code.
Resume bullet: "Prevented a transaction-scoped external API call from reaching production via architecture-level code review."

## Story 6: Making an implicit contract explicit
Prompts: "Improved a process/system" / "Technical debt."
Source: [K-5 kata / SP-2](../06-contribution-practice/03-senior-build-projects.md) (typed presenter contracts) done on your fork.
Fill in: the drift risk you demonstrated, the incremental migration (one presenter per PR), what the compiler caught.
Senior signal: incrementalism — you made it adoptable, not a rewrite.
Resume bullet: "Introduced compile-time API contract checking between server presenters and client models, incrementally."

## Story 7 (study-based): Explaining a tradeoff you didn't make
Prompts: "Tell me about a technical decision and its tradeoffs" — works even when the decision wasn't yours, *if you say so*.
Source: the modular-monolith services design ([system map](../01-codebase-cartography/01-system-map.md)) or CRDT-over-OT ([system design Step 5](04-system-design-from-this-repo.md)).
Frame: "A codebase I know well chose X; here's the pressure that shaped it, what it costs, and where I'd revisit it." Honest, and demonstrates exactly the judgment being probed.

## Story 8 (study-based): The mistake/lesson story
Prompts: "Tell me about a mistake."
Source: your own drill failures — e.g., a kata where your confident finding turned out to be a reasonable tradeoff you'd misjudged (K-1's other-side argument, or a perf suspect that measured innocent in K-7).
Frame: what you asserted, what the evidence showed, the check you now always run first. Interviewers want the *changed behavior*, not the confession.

## Story 9 (study-based): Teaching/mentoring
Prompts: "Helped someone grow" / collaboration.
Source: teach-backs — explain one key flow to a real person (colleague, friend, study group) and iterate on their confusion.
Frame: how you diagnosed *their* mental-model gap (not just repeated yourself louder); evidence they got it.

---

## Prompt-to-story index

| Prompt | Stories |
| --- | --- |
| Ambiguity / ramp-up | 1, 4 |
| Hardest bug | 3, and D-round experiences |
| Conflict / disagreement | 5, 2 |
| Mistake | 8 |
| Technical tradeoff | 7, 2, 6 |
| Initiative / leadership | 2, 6, 9 |
| End-to-end delivery | 4 |

Two-minute versions of stories 1, 2, and 4 cover ~80% of behavioral rounds. Record yourself; cut every sentence that doesn't carry either evidence or judgment.

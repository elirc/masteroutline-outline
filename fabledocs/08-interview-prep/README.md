# 08 — Interview Prep

First-class module, not an afterthought. Target: **mid-level fullstack JS/TS roles.** Over 50 question cards live here, and more than 60% are anchored to real code in this repo — because the winning interview move is answering with a concrete example, a tradeoff, and a failure mode instead of a definition. That's the golden rule of this module; every card drills it.

## How mid-level fullstack loops typically run

| Round | Format | What's actually graded | Prep here |
| --- | --- | --- | --- |
| Screen | 30–45 min, resume + light technical | can you talk about your work concretely | [06-behavioral-star-stories.md](06-behavioral-star-stories.md) |
| Technical deep-dive | 45–60 min on JS/TS/Node/React | depth + honesty about limits | [01](01-js-ts-node-deep-dive.md), [02](02-frontend-framework-questions.md) |
| Practical coding | build/extend a small feature | working code + tests + narration | modules [04](../04-code-reading-gym/README.md)/[05](../05-quality-engineering/README.md) habits |
| System design | 45–60 min whiteboard | structure, tradeoffs, failure modes | [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md) |
| Debugging / code review | timed, realistic artifact | method over luck | [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md) |
| Behavioral | STAR prompts | evidence of judgment + collaboration | [06-behavioral-star-stories.md](06-behavioral-star-stories.md) |

## Using this repo as your portfolio of talking points

You studied (and ideally contributed to) a production-grade open-source codebase: multi-tenant, real-time, queue-backed, policy-authorized. Practically every classic question maps onto something you can *point at*:

- "Design X" → [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md)
- "How do you handle auth/permissions?" → policies + serialized abilities ([03-architecture/03](../03-architecture-and-patterns/03-validation-auth-and-permissions.md))
- "Background jobs? Consistency?" → event bus, idempotency, outbox gap ([03-architecture/04](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md))
- "Testing philosophy?" → real-DB integration strategy ([05-quality/01](../05-quality-engineering/01-testing-strategy.md))
- "Tell me about a codebase you learned" → your literal last few weeks

Honesty calibration: "I studied and built against Outline, an open-source knowledge base" is entirely legitimate; claiming maintainership isn't. The former, told with file-level specifics, outperforms vague claims about proprietary work anyway.

## Files

1. [01-js-ts-node-deep-dive.md](01-js-ts-node-deep-dive.md) — 14 cards
2. [02-frontend-framework-questions.md](02-frontend-framework-questions.md) — 12 cards
3. [03-api-and-data-modeling-questions.md](03-api-and-data-modeling-questions.md) — 12 cards
4. [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md) — full walkthrough + 5 variations
5. [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md) — 7 timed simulations
6. [06-behavioral-star-stories.md](06-behavioral-star-stories.md) — 9 worksheets
7. [07-two-week-cram-plan.md](07-two-week-cram-plan.md) — day-by-day

Practice protocol for all cards: answer aloud, 90 seconds, then self-grade Basic/Solid/Strong against the card. Out loud is non-negotiable — fluency is a motor skill.

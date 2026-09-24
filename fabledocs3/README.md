# fabledocs3 — Reverse-Engineered Architecture Decision Records

Nobody wrote ADRs for this codebase at the time. The decisions got made
anyway — in PRs, in migrations, in the shape of directories — and the code is
the only record. This pack reconstructs the ten most significant ones in
standard ADR format (Context, Decision, Alternatives Considered, Consequences,
Status), each grounded in specific files and classes you can open and verify.

"Status" here means: **Active** (still the law of the land), **Active-Eroding**
(still enforced but under visible pressure), or **Active-with-Scar-Tissue**
(the decision stands, and the code contains visible remediation for what it
cost).

## The ten decisions

| ADR | Title | Domain |
| --- | --- | --- |
| [ADR-001](adr-001-modular-monolith-service-flags.md) | Modular monolith with runtime service composition | Structure / distributed |
| [ADR-002](adr-002-three-runtime-layering.md) | Three-runtime layering: `app` / `server` / `shared` (+ `plugins`) | Structure |
| [ADR-003](adr-003-rpc-post-api.md) | RPC-over-POST API contract (`noun.verb`), Zod at the boundary | API contract |
| [ADR-004](adr-004-policy-authorization.md) | Centralized CanCan policy layer, serialized to clients | API / security |
| [ADR-005](adr-005-sequelize-explicit-transactions.md) | Sequelize with decorators; explicit transaction-per-mutation | Data access |
| [ADR-006](adr-006-event-spine.md) | Events table + Redis/Bull queues as the async integration spine | Data / distributed |
| [ADR-007](adr-007-crdt-triple-representation.md) | Yjs CRDT collaboration with triple document representation | Data / distributed |
| [ADR-008](adr-008-redis-horizontal-scale.md) | Redis as the horizontal-scale substrate (pub/sub, locks, limits, cache) | Distributed |
| [ADR-009](adr-009-error-philosophy.md) | Typed error factories, stable ids, `isReportable`, one funnel | Error handling |
| [ADR-010](adr-010-integration-first-testing.md) | Integration-first testing on real infrastructure | Testing |

Then: [interview-translation.md](interview-translation.md) — each ADR
compressed into a 90-second spoken answer, plus the follow-up question an
interviewer is most likely to throw at you.

## How these were reconstructed

Method, so you can trust (or challenge) the results: I looked for **paid
costs** — code that exists only because a decision made it necessary. A
`CleanupOldEventsTask` implies someone chose an unbounded events table. A
`SET LOCAL lock_timeout` implies someone got burned by a lock. A clamp
forcing `webProcessCount = 1` implies someone chose in-process state and hit
its scaling wall. Decisions leave scars; the scars are the evidence. Where I
attribute intent, it's inference from that evidence — flagged where the
inference is weaker.

Read them in order; 001 and 002 are load-bearing for the other eight.

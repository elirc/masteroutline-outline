# 03 — Architecture and Patterns

The judgment module. Cartography told you *where*; this module teaches *why it's shaped that way, what it costs, and when you'd choose differently* — which is precisely what mid-level interviews test.

| Page | Purpose |
| --- | --- |
| [01-boundaries-and-layers.md](01-boundaries-and-layers.md) | The layer contract: what each layer owns, real leaks |
| [02-data-model-and-persistence.md](02-data-model-and-persistence.md) | Schema, transactions, migrations, how to change data safely |
| [03-validation-auth-and-permissions.md](03-validation-auth-and-permissions.md) | Every gate a request passes; what juniors miss vs seniors check |
| [04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md) | The event/queue system; idempotency, retries, failure visibility |
| [05-pattern-catalog.md](05-pattern-catalog.md) | 16 recognition cards for patterns used here |
| [06-architecture-critique.md](06-architecture-critique.md) | Strengths, risks, and "what I'd change" — doubles as system-design prep |

Vocabulary this module defines and uses precisely: **boundary, contract, invariant, ownership, idempotency, isolation, consistency, blast radius, migration/rollback**. Each is flagged where introduced — they are interview words, and misusing them is worse than not using them.

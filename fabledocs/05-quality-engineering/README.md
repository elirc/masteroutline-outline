# 05 — Quality Engineering

How this repo keeps itself correct, fast, secure, and observable — and how you practice each skill here.

| Page | Purpose |
| --- | --- |
| [01-testing-strategy.md](01-testing-strategy.md) | The test pyramid as it actually exists here; what belongs where |
| [02-writing-tests-here.md](02-writing-tests-here.md) | Seven concrete test recipes with this repo's real helpers |
| [03-systematic-debugging.md](03-systematic-debugging.md) | The method + five repo-specific scenarios |
| [04-performance-thinking.md](04-performance-thinking.md) | Measure-first, likely hotspots, N+1/render/bundle hunting |
| [05-security-checklist.md](05-security-checklist.md) | Threats mapped to real defenses; pre-merge checklist |
| [06-observability-and-operations.md](06-observability-and-operations.md) | Logs/metrics/traces here; "how would I know this broke?" |

The through-line: quality is *mechanisms*, not virtue. Every claim like "we test carefully" should decompose into a mechanism you can point at (a factory, a snapshot, a rate limiter, an alert). Interviews reward mechanism-level answers overwhelmingly.

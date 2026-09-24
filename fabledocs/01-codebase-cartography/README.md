# 01 — Codebase Cartography

You cannot reason about code you cannot locate. This module builds your map of Outline before you try to change it.

Transferable skill: *every* large repo has the same questions — where do requests enter, where does business logic live, where are contracts defined, where do side effects happen, what is generated vs. hand-written? Learning to answer them here teaches you to answer them anywhere. Interviewers probe this directly with "tell me about a large codebase you had to learn — how did you approach it?" (see [08-interview-prep/06-behavioral-star-stories.md](../08-interview-prep/06-behavioral-star-stories.md), Story 1).

| Page | Purpose |
| --- | --- |
| [01-system-map.md](01-system-map.md) | The repo's shape: trees, services, ownership, public vs private surfaces |
| [02-file-reading-order.md](02-file-reading-order.md) | 30 files in the order that builds understanding fastest, with junior/mid/senior paths |
| [03-domain-glossary.md](03-domain-glossary.md) | Product nouns (Document, Collection, Share, Star…) mapped to code, plus confusables |
| [04-runtime-and-tooling-map.md](04-runtime-and-tooling-map.md) | Yarn 4, Vite, Vitest, oxlint/oxfmt, service boundaries, env config |
| [05-key-flows.md](05-key-flows.md) | Six flows traced end to end — the backbone the rest of the curriculum rotates through |

Suggested order: 01 → 05 → 02 (using it as a checklist over several days) → 03/04 as reference.

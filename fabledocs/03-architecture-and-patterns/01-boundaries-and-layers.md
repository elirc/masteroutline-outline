# Boundaries and Layers

A **boundary** is a line where two pieces of code can evolve independently as long as a **contract** (an agreed shape + semantics) holds. A **layer** is a boundary stacked horizontally: everything above may call down; nothing below may know about above. **Ownership** means exactly one layer is allowed to make a given kind of decision.

## The server's layer stack

```
HTTP (Koa)                     server/routes/api/**            parses, authenticates, authorizes, presents
  │  contract: ctx.input (Zod), ctx.state.{auth,transaction}
Commands                       server/commands/**              multi-model orchestration
  │  contract: (ctx, props) → models; must thread ctx.state.transaction
Models                         server/models/**                per-entity invariants, scopes, hooks
  │  contract: Sequelize instances; events via saveWithCtx/hooks
Policies (sidecar)             server/policies/**              pure authorization decisions
Presenters (sidecar)           server/presenters/**            wire-format translation
Queues (async annex)           server/queues/**                everything after the response
```

What each layer owns — and must not own:

- **Routes** own HTTP: status codes, headers, pagination, calling `authorize`. They must not own business rules. Test: could you paste handler logic into a CLI command? If not, it's leaking HTTP-ness downward or business-ness upward. [documents.update](../../server/routes/api/documents/documents.ts#L1251-L1294) is a clean example: parse → load → authorize ×3 → delegate → present.
- **Commands** own "what happens when X" across models. They must not read raw request objects (they take typed props) nor decide permissions (callers `authorize` first — see documentUpdater receiving already-authorized work).
- **Models** own single-entity invariants (e.g. Document's title/content coupling, `publish` semantics) and data-access idioms (scopes). They must not know about HTTP or presenters.
- **Policies** own *decisions* and must be pure/synchronous — the design forces data loading to happen before (see [Pattern 5](05-pattern-catalog.md)).
- **Presenters** own the wire shape and *withholding* (public-share field stripping — [presenter L97–129](../../server/presenters/document.ts#L97-L129)). They must not mutate or fetch-per-item (the batch variant exists to prevent that — [L147–176](../../server/presenters/document.ts#L147-L176)).

## The client's layer stack

```
Scenes/Components   app/scenes, app/components    render observables, dispatch actions
Stores              app/stores                    cache + fetch orchestration (business logic home)
Models              app/models                    per-entity behavior, dirty tracking, save()
Transport           app/utils/ApiClient.ts        HTTP mechanics only (CSRF, retry, dedupe, errors)
```

Ownership rule from [AGENTS.md](../../AGENTS.md#L98-L105): business logic in stores, not components. The transport layer's discipline shows in `setUnauthorizedHandler` ([ApiClient L82–91](../../app/utils/ApiClient.ts#L82-L91)) — the comment says it outright: session policy is kept *out of the transport layer*.

## Real examples of good boundaries

1. **Policy serialization** — one rule engine, two consumers (server enforcement + client rendering) via the `policies` payload. The client never re-implements a rule. ([cancan serialize](../../server/policies/cancan.ts#L127-L155) → [usePolicy](../../app/hooks/usePolicy.ts).)
2. **Search provider seam** — routes call `SearchProviderManager.getProvider().searchForTeam(...)` ([documents.search](../../server/routes/api/documents/documents.ts#L1095-L1110)); Postgres vs external engines swap behind it.
3. **The queue boundary** — request-side code *never* performs slow side effects; it emits events. Everything in [server/queues](../../server/queues) runs after and independently. Ownership is crisp: if it must be atomic with the write → same transaction; otherwise → event.
4. **Plugins** — auth providers and integrations extend via hooks without core edits ([Pattern 15](05-pattern-catalog.md)).

## Boundary leaks worth studying (judgment practice)

- **The 2,106-line route file.** [documents.ts](../../server/routes/api/documents/documents.ts) contains real query-building logic inline (e.g. the `where` assembly in documents.list L119–171, and a large search handler). Is that a leak? Argue both sides: *for* — filter assembly is HTTP-adjacent (it derives from request params); *against* — it's untestable without HTTP and duplicates between endpoints. Mid-level skill is having this argument with specifics, not declaring "too long, must split."
- **Collection `documentStructure`** — a denormalized document tree living on the collection row. Documents own themselves, but their *ordering/nesting* is owned by the collection JSON. Two owners of one fact = the classic leak; the mitigation is that all structure edits funnel through commands ([documentMover](../../server/commands/documentMover.ts)) under transactions. Watch for any code path editing structure without the command.
- **Client stores knowing server cascade rules** — [Store.remove](../../app/stores/base/Store.ts#L174-L221) re-implements FK semantics via decorators. Necessary (the client must predict server behavior without a round trip) but it's duplicated knowledge that can drift — e.g., if a server migration changes an `onDelete`, someone must remember the client mirror.
- **`data` vs `changes` on Event** — [Event.ts L59–73](../../server/models/Event.ts#L59-L73) documents an in-progress migration between two overlapping columns, with the warning that `data` is client-visible. A boundary (internal metadata vs public payload) being retrofitted — good example that boundaries are maintained, not declared.

## Interview angle

1. *"How is your codebase layered, and where does business logic live?"* — describe routes/commands/models with the ownership tests above; concrete beats abstract.
2. *"Tell me about a boundary violation you found."* — use `documentStructure` or the client-side cascade mirror; state the risk **and** why it's a reasonable tradeoff. Balanced critique is the senior signal.
3. *"How do frontend and backend stay consistent about permissions?"* — the policy serialization boundary, verbatim.

## Drill

Take [documents.duplicate](../../server/routes/api/documents/documents.ts#L1296-L1352). For each line, label which layer's job it's doing (HTTP / authz / orchestration / presentation). Then answer: why do the *three* authorize calls live in the route instead of inside `documentDuplicator`? (Hint: the command is also called from contexts with different authorization needs — check its other call sites with grep.)

Self-grade — Basic: labels mostly right. Solid: articulates the authorize-in-route rationale. Strong: proposes and critiques an alternative (authorization inside commands) with a concrete failure it would cause here.

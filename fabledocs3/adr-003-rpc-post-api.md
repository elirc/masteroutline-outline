# ADR-003: RPC-over-POST API Contract with Zod at the Boundary

**Status:** Active

## Context

The API is a public product surface — the first-party SPA, Zapier, MCP
clients, and curl-wielding customers all consume it. The domain's operations
are verb-heavy ("move document and reindex siblings," "restore revision,"
"add user to collection") and map poorly to resource-oriented CRUD. Requests
need consistent validation, authentication, rate limiting, and auditability
across ~40 resource groups.

## Decision

Model the API as **RPC over POST**: every endpoint is
`POST /api/<noun>.<verb>` (`documents.info`, `documents.update`,
`collections.add_user`), parameters in the JSON body. Each resource directory
(`server/routes/api/<resource>/`) contains handlers plus a `schema.ts` of Zod
request schemas; the `validate()` middleware (`server/middlewares/validate.ts`)
parses `ctx.request` against the schema and populates `ctx.input`, so handlers
never read raw request bodies. Responses are uniformly
`{ data, policies?, pagination? }`, shaped by presenters
(`server/presenters/`). The pipeline order is fixed: rate limit → authenticate
→ validate → (transaction) → handler → presenter.

The frontend mirrors the convention: `app/stores/base/Store.ts` defines an
`RPCAction` enum (`Info`, `List`, `Create`, `Update`, `Delete`, `Count`) and
derives endpoint names from model names — the contract is so regular that the
client generates calls mechanically.

## Alternatives Considered

1. **REST.** Rejected: verbs like "move," "restore," "duplicate" force
   awkward sub-resource modeling and idempotency debates; `noun.verb` names
   operations directly, which also makes per-operation rate limiting and
   audit logs trivial.
2. **GraphQL.** Rejected: per-object conditional authorization (ADR-004)
   makes resolver-level field leakage too easy; a query language is a worse
   contract for integration customers than "POST this JSON here"; per-named-
   operation caching/limits get harder.
3. **Per-handler ad hoc validation.** Rejected: validation in *some* handlers
   is the observed steady state of that approach. Centralizing in middleware
   plus colocated `schema.ts` makes the request shape reviewable next to the
   handler.

## Consequences

**Positive:**
- Uniformity: any engineer can find any operation by name; clients branch on
  stable structure; the SPA's store layer automates calls via `RPCAction`.
- Zod schemas double as request documentation and produce typed `ctx.input`.
- Everything is a POST, so no cache-layer surprises on authenticated reads
  and no debates about which verb a "move" is.

**Negative:**
- Over-fetching: presenters return fat payloads; the growing options bag on
  `presentDocument` (`includeText`, `includeData`, `includeCommentCount` in
  `server/presenters/document.ts`) is field-selection being reinvented one
  boolean at a time.
- Read endpoints as POSTs forfeit HTTP caching semantics entirely.
- `validate.ts` surfaces only the **first** Zod issue per request — hostile
  to API consumers iterating on a payload; unfixed because it would change
  the error contract.

## Evidence

- `server/routes/api/documents/documents.ts` + `schema.ts` — the canonical
  resource module.
- `server/middlewares/validate.ts` — the entire validation strategy in ~30
  lines, including the first-issue-only behavior.
- `server/routes/api/schema.ts` — `BaseSchema` shared by resource schemas.
- `app/stores/base/Store.ts` — `RPCAction`, client-side mirror of the
  convention.
- `server/presenters/document.ts` — the options-bag consequence.

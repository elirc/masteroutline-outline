# Security Checklist

Threats mapped to this repo's actual defenses, then a pre-merge checklist. Multi-tenant SaaS + user-generated rich content + public sharing + integrations = a genuinely broad attack surface; that breadth is why this page exists. See also the repo's own [docs/SECURITY.md](../../docs/SECURITY.md).

## Threat → defense map

| Threat | Defense here | Anchors |
| --- | --- | --- |
| **Authorization bypass / IDOR** | scoped loads + per-object `authorize` + tenant checks in policies | [gauntlet table](../03-architecture-and-patterns/03-validation-auth-and-permissions.md); [documents.duplicate triple-check](../../server/routes/api/documents/documents.ts#L1296-L1352) |
| **CSRF** | token cookie + verification middleware on mutations; auth-transport restrictions (API key not via cookie) | [csrf.ts](../../server/middlewares/csrf.ts), [authentication.ts L200–203](../../server/middlewares/authentication.ts#L200-L205) |
| **XSS via rich content** | ProseMirror schema whitelists nodes/marks (parse = sanitize); `sanitizeUrl` for href/src in `toDOM` (AGENTS.md calls this out as policy — raw DOM writes don't auto-escape) | [shared/utils/urls.ts L207](../../shared/utils/urls.ts#L207), [AGENTS.md L200–204](../../AGENTS.md#L200-L208) |
| **XSS via templating** | React escaping app-side; server HTML responses escape interpolations (`escape` from es-toolkit in [redirectOnClient](../../server/index.ts#L100-L156)); CSP with nonces | [csp.ts](../../server/middlewares/csp.ts), helmet at [server/index.ts L92](../../server/index.ts#L92) |
| **SSRF** (unfurls, webhooks, imports fetch URLs) | outbound request utilities + proxy env handling; *verify depth*: [server/utils/fetch.ts](../../server/utils/fetch.ts) exists — read it for internal-IP blocking before claiming coverage (**investigate**) | [UploadAttachmentFromUrlTask](../../server/queues/tasks/UploadAttachmentFromUrlTask.ts) |
| **Injection (SQL)** | Sequelize parameterization; search input sanitized via `pg-tsquery` | [plugins/search-postgres](../../plugins/search-postgres) |
| **Secrets exposure** | env-validated config; presenters as field allowlists; `Event.data` explicitly documented as client-visible | [Event.ts L59–65](../../server/models/Event.ts#L59-L65) |
| **Brute force / abuse** | global + per-route rate limits (Redis-backed) | [rateLimiter.ts](../../server/middlewares/rateLimiter.ts), [Pattern 16](../03-architecture-and-patterns/05-pattern-catalog.md) |
| **Session risks** | JWT in cookie with expiry; suspended-user gate on every request; OAuth scopes + `canAccess` per token | [authentication.ts L156–260](../../server/middlewares/authentication.ts#L156-L260) |
| **Public-share overexposure** | publicness recomputed, fields stripped, comment marks removed, subtree fencing, signed URL TTLs | [Flow 2](../01-codebase-cartography/05-key-flows.md) |
| **File upload risks** | presigned uploads (server never proxies bytes), `sanitize-filename`, content-disposition on downloads, attachment expiry cleanup | [attachments routes](../../server/routes/api/attachments), [CleanupExpiredAttachmentsTask](../../server/queues/tasks/CleanupExpiredAttachmentsTask.ts) |
| **Webhook/integration trust** | signature validation middleware for inbound hooks; outbound payload minimization | [validateWebhook.ts](../../server/middlewares/validateWebhook.ts) |
| **Open redirects** | escaped/structured redirect helper; OAuth `state` handling in passport middleware | [server/index.ts L100–156](../../server/index.ts#L100-L156), [passport.ts](../../server/middlewares/passport.ts) |
| **Dependency risk** | targeted `resolutions` policy, CodeQL workflow, patch-package auditability | [tooling page](../02-stack-and-language-mastery/04-tooling-and-build-system.md) |

Two entries above carry *investigate* labels on purpose — copying a checklist without verifying depth is exactly the anti-pattern this curriculum trains against. Reading `server/utils/fetch.ts` and the CSP policy end-to-end is [a senior kata](../06-contribution-practice/04-refactor-and-design-katas.md).

## The instinct to install

Every input has a **trust boundary crossing** where it must be (a) shape-validated, (b) authorization-checked, (c) neutralized for its *output context* (SQL, DOM, URL, email HTML, shell — each context needs its own escaping). Asking "which context does this string end up in?" catches more real bugs than memorizing OWASP acronyms — though you should also know the acronyms (XSS, CSRF, SSRF, IDOR) because interviews use them.

## Pre-merge security checklist (for your PRs here and anywhere)

- [ ] Every id in the request (including nested/related ids) is authorized against the session user
- [ ] New queries filter by tenant (`teamId`) or go through a scoped loader
- [ ] New response fields pass through a presenter decision (would this field be OK on a public share?)
- [ ] User-controlled strings: know their output context; URL fields go through `sanitizeUrl`
- [ ] New mutation covered by CSRF (cookie-auth path) — automatic if it's under /api with standard middleware
- [ ] Expensive/abusable endpoint has a rate-limit strategy
- [ ] Anything fetching a user-supplied URL goes through the hardened fetch path
- [ ] No secrets/PII in logs, events (`data` is client-visible!), or error messages
- [ ] Tests include at least one negative (cross-team or role-based) case

## Interview angle

1. *"How do you prevent XSS with user-generated rich text?"* — schema-whitelisted editor + the `toDOM`/`sanitizeUrl` nuance (raw DOM ≠ React escaping) — an unusually concrete answer.
2. *"CSRF — when do you need protection and why?"* — cookie-transported credentials only; tie to the transport restrictions here.
3. *"What's IDOR and how does your codebase structurally prevent it?"* — scoped loads + policy, [validation page](../03-architecture-and-patterns/03-validation-auth-and-permissions.md).
4. *"How do you handle security of public sharing?"* — Flow 2's checklist.

## Drill

Red-team `documents.import` ([documents.ts L1503+](../../server/routes/api/documents/documents.ts#L1502-L1560)): list five attack ideas (file type spoofing, zip bombs, path traversal in archive entries, huge payloads, HTML smuggling), then find which defense — if any — each hits. Write findings as investigate-items, not accusations; verify two of them by reading [documentImporter](../../server/commands/documentImporter.ts).

# ADR-002: Three-Runtime Layering — `app` / `server` / `shared`, Plus `plugins`

**Status:** Active

## Context

A collaborative editor has an unusual constraint: the document schema and
editing logic must behave *identically* in the browser (where users type) and
on the server (where the collaboration service merges updates and where
exports, search indexing, and revision diffs are computed). Divergence
between client and server document models isn't a bug, it's data corruption.
Meanwhile the product carries ~20 optional integrations (Slack, OIDC, GitHub,
S3 storage, Postgres search...) that must not entangle core code.

## Decision

Split by **runtime**, not by feature:

- `app/` — browser-only React/MobX SPA (scenes, stores, models, hooks).
- `server/` — Node-only Koa API, models, policies, queues, collaboration.
- `shared/` — code that must run in **both** runtimes; critically
  `shared/editor/` (the Prosemirror schema, nodes, marks, plugins) plus
  types, i18n, and validation constants.
- `plugins/` — optional features, each self-contained with client and server
  halves, registered through typed hooks in `server/utils/PluginManager.ts`
  (`Hook.API`, `Hook.AuthProvider`, `Hook.Processor`, `Hook.SearchProvider`,
  ...) with `PluginPriority` ordering. Core iterates hooks; it never imports
  a plugin by name.

The dual-runtime claim on `shared/` is *enforced by testing*, not just
convention: Vitest runs the `shared` project twice, once under Node and once
under jsdom (`vitest.config.ts`, `test:shared` script).

## Alternatives Considered

1. **Feature-sliced monorepo packages** (e.g. `packages/documents`,
   `packages/auth`). Rejected: the hard boundary in this system is "which
   runtime can execute this," not "which feature owns this" — a feature
   slice would still need runtime sub-splits inside every package.
2. **Duplicate editor code per side** (a client editor and a server-side
   approximation). Rejected: the collaboration server materializes real
   Prosemirror documents from Yjs state (`server/editor`,
   `documentCollaborativeUpdater.ts`); an approximation guarantees drift.
3. **Runtime-loadable third-party plugins** (npm-installable, sandboxed).
   Rejected: a plugin that can register model hooks can corrupt data;
   sandboxing Node is not a feature-sized problem. Plugins are in-tree,
   compile-time, trusted code.

## Consequences

**Positive:**
- One editor schema, two consumers — the invariant that makes collaborative
  editing safe is structural, not procedural.
- Integrations are additive: a new provider touches `plugins/<name>/` and
  nothing in core (compare `plugins/slack/` vs `plugins/search-postgres/`).
- The BSL/enterprise split (`plugins/enterprise`) is possible without
  scattering license checks through core.

**Negative:**
- `shared/` is a standing foot-gun: one careless server-only import makes it
  browser-poisonous, and only the dual test environment catches it.
- Plugin hook typing has limits — `PluginValueMap` in `PluginManager.ts`
  carries an explicit `any` escape hatch for heterogeneous email-template
  props.
- Runtime-split means feature logic is spread across three trees
  (`app/models/Document.ts`, `server/models/Document.ts`, shared types);
  newcomers must learn the correspondence.

## Evidence

- Top-level layout: `app/`, `server/`, `shared/`, `plugins/`.
- `shared/editor/` — schema shared by `app/editor` and `server/editor`.
- `server/utils/PluginManager.ts` — `Hook` enum, `PluginPriority`, glob-based
  loading.
- `vitest.config.ts` — `shared-node` and `shared-jsdom` projects.
- `tsconfig.json` path aliases (`@server/*`, `@shared/*`, `~/*`) encoding the
  layer boundaries.

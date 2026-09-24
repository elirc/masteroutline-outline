# Tooling and Build System

Build tooling is architecture with worse error messages. This page turns this repo's toolchain into transferable understanding.

## The two build pipelines

**Frontend:** Vite ([vite.config.ts](../../vite.config.ts)) — dev = native-ESM serving with instant HMR; prod = Rollup bundle with code-splitting (route chunks are lazy-loaded — [app/routes](../../app/routes/index.tsx), `lazyWithRetry` in [app/utils/lazyWithRetry.ts](../../app/utils/lazyWithRetry.ts), which also handles the deploy-time "chunk vanished" failure — a tooling failure mode most people learn in production).

**Backend:** compiled to `build/` by [build.js](../../build.js) and *run as plain Node* — dev loop is nodemon watch → rebuild → restart ([package.json L14](../../package.json#L13-L15)). Why not ts-node/tsx? Legacy decorators + metadata semantics need a controlled transform (SWC), and prod-parity matters.

**Why SWC specifically** — the comment block in [vitest.config.ts L20–41](../../vitest.config.ts#L20-L41) is a masterclass in constraint documentation: legacy decorators for MobX 4/sequelize-typescript; `decoratorMetadata` off because the circular model graph would crash; `useDefineForClassFields: false` because TC39 class-field semantics would shadow MobX observables. Memorize the *shape* of this story: "our runtime libraries pinned our language dialect, which pinned our compiler." That chain of constraint is what "tooling is architecture" means.

## Testing infrastructure

Four Vitest projects in [vitest.config.ts](../../vitest.config.ts#L60-L115): `server` (node env, real Postgres, setup in [server/test/setup.ts](../../server/test/setup.ts)), `app` (jsdom), `shared-node` + `shared-jsdom` (the same shared tests run in *both* runtimes — enforcing the isomorphism contract mechanically; that's a pattern worth stealing for any shared/ directory).

## Lint/format as contracts

oxlint (type-aware) + oxfmt — Rust-speed equivalents of ESLint/Prettier ([package.json L16–19](../../package.json#L16-L19)). Notice lint *rules as architecture enforcement*: `no-restricted-imports` steering code away from raw `node-fetch` (visible as opt-outs in [TestServer.ts L4–7](../../server/test/TestServer.ts)); `oxlint-disable` comments always carrying reasons. Pre-commit via husky/lint-staged means style never reaches review — reviewers' attention is spent on substance.

## Dependency management as risk management

- `resolutions` ([package.json L372–385](../../package.json#L372-L385)) pin transitive deps — including targeted security overrides using the `pkg@npm:<range>` syntax that [AGENTS.md](../../AGENTS.md#L40-L59) documents as policy: override *only the vulnerable range*, not the world. This is supply-chain hygiene you can describe in interviews.
- `patch-package` (`postinstall`, [patches/](../../patches)) — surgical fixes to node_modules, versioned in-repo. Know when it beats forking (small diffs, upstream slow) and its cost (upgrade friction).
- Engines pinned to specific Node ranges ([package.json L44–46](../../package.json#L44-L46)) — including *excluded* patch versions, meaning someone got burned by a specific Node release and encoded the lesson.

## CI shape

[.github/workflows/ci.yml](../../.github/workflows/ci.yml) — the canonical gate (lint, types, tests). Also present: CodeQL (static security analysis), Docker image builds, bots for stale issues and translation PRs. Transferable checklist: does CI run what a maintainer *actually* requires? (When in doubt, read the workflow, not the docs — CI config is the executable truth of a project's quality bar.)

## Interview angle

Cards in [08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md) (Q7, Q14):
1. "Walk me through what happens when you run the dev server."
2. "Why might a team still be on legacy decorators, and what does it cost?" — the constraint chain above.
3. "How do you handle a vulnerable transitive dependency?" — targeted resolutions.
4. "Monorepo tooling — how does code sharing work without workspaces?" — path aliases mirrored across tsconfig/vitest/vite.
5. "What's your CI, and why those steps?"

## Drill

Without looking, list the four vitest projects and what runtime each simulates; then explain to an imaginary junior *why* shared tests run twice. Verify against [vitest.config.ts](../../vitest.config.ts#L60-L115).

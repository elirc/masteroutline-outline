# 03 — Capstone: The Multi-Client Collaboration Harness (Paying the IOU)

## The problem you're solving

ADR-010's confessed gap: the highest-risk subsystem — two-plus clients
merging concurrent edits through the Hocuspocus server — has no automated
end-to-end test. The pieces are covered (`documentCollaborativeUpdater.test.ts`,
`ConnectionLimitExtension.test.ts`); the *composition* rests on CRDT math and
staged rollouts. fabledocs2 doc 04 called building this "a high-leverage
infrastructure project." That's you, now.

**Prerequisites:** fabledocs2 docs 03/04, ADR-007/010, and skim
`server/collaboration/*.ts` — you need to know the extension chain
(authentication → connection limit → persistence → logging/metrics) cold.

## Stage 0 — Recon (half a day)

Answer in writing, from code:

1. How does a client connect? Find the websocket URL shape and the document
   naming convention (`PersistenceExtension.onLoadDocument` splits
   `documentName` on `"."` — what's the full format? Trace who constructs it
   on the client: look in `app/` for the Hocuspocus provider setup, and in
   `shared/collaboration/` for anything shared).
2. How does auth happen? Read `server/collaboration/AuthenticationExtension.ts`
   — what token does it expect and how would a test mint one? (Factories +
   the JWT utilities in `server/utils/jwt.ts` are your friends; the
   websockets service does `getUserForJWT` — the collaboration path is
   analogous.)
3. What does "persisted" mean and when? Re-read the debounce behavior and
   `isLastConnection` in `documentCollaborativeUpdater.ts` — your assertions
   must wait for *persistence*, not just message delivery, and last-
   connection-close is a forcing event you can exploit for determinism.
4. What client library can you use from a test? `@hocuspocus/provider` is
   already in the dependency tree for the app — verify, and check it runs
   under Node (websocket transport in a non-browser environment is the
   classic snag; you may need a `ws` polyfill).

## Stage 1 — Harness skeleton (1 day)

Build a `CollabTestHarness` under `server/test/` (peer of `TestServer.ts` —
match its style):

- Boots the collaboration service against the test database (study how
  `TestServer` boots the web service; you're doing the same for
  `server/services/collaboration.ts`).
- `connect(user, document)` → returns a client handle wrapping a Hocuspocus
  provider + Y.Doc, authenticated as that user.
- `type(client, text, at)` → applies a Yjs update as if typed (mutate the
  `prosemirror` fragment the way `y-prosemirror` expects — crib from how the
  shared editor binds, `shared/editor/` + `y-prosemirror` docs).
- `waitForSync(clients)` → resolves when all clients' state vectors converge
  (Yjs gives you `Y.encodeStateVector` comparison — poll or hook update
  events; no sleeps).
- `waitForPersist(documentId)` → resolves when the DB row's state reflects
  the merged doc (poll the model; bound it with a timeout so failures are
  loud, not hung).
- Teardown that actually closes sockets and flushes — leaking connections
  will poison neighboring tests, and you know from ADR-010 how miserable
  cleanup-convention leaks are to bisect.

## Stage 2 — The test suite (1 day)

Write these six, in ascending order of nastiness:

1. **Echo:** A types, B sees it. (Harness smoke test.)
2. **Concurrent divergence:** A and B type simultaneously at different
   positions → both converge to a doc containing both edits, identical state
   vectors.
3. **Same-position conflict:** A and B type at the *same* position →
   converged docs are identical on both clients (you're not asserting which
   order won — CRDT doesn't promise that — you're asserting *agreement*).
4. **Offline merge:** disconnect B, A types, B types locally, reconnect B →
   convergence, nothing lost. (This is the CRDT sales pitch; prove it.)
5. **Persistence truth:** after edits + last client disconnects → DB `state`
   materializes to the converged doc; and the projections refresh path
   emitted its event (check the events table — ADR-006 ties in here).
6. **Authorization:** a user from another team (factory-built — the ADR-010
   convention) attempts to connect to the document → rejected by
   `AuthenticationExtension`, and — sharper — a user whose access is revoked
   *mid-session* … discover and document what actually happens. This last
   one may find a real behavior worth writing down; harnesses earn their
   keep the first time they surprise you.

## Stage 3 — Make it CI-shaped (half a day)

- New Vitest project or tag within the `server` project? Decide and defend
  (these tests are slower and stateful; a separate project keeps them out of
  the fast path, but a project nobody runs is a project that rots —
  consider running it in CI but not in `test:watch`).
- Flake policy: every wait is condition-based with a hard timeout; zero
  `sleep`s. Run the suite 20× in a loop (`vitest run --project collab` in a
  shell loop) and fix anything that flakes *now*, because a flaky collab
  suite will be deleted by a frustrated teammate within a quarter, and then
  the IOU is back.

## Acceptance criteria

- [ ] Recon answers written, with file references.
- [ ] Harness supports N clients, condition-based waits, clean teardown.
- [ ] All six tests pass; suite survives 20 consecutive runs.
- [ ] Persistence test asserts DB state *and* event emission.
- [ ] A README section in the harness explaining how to add a test in
      <10 lines (adoption is the difference between infrastructure and
      artifact).

## Interview payoff

"Our collaborative-editing path had no end-to-end coverage, so I built a
multi-client harness against the real collaboration server — the hard parts
were deterministic convergence waits without sleeps, and forcing persistence
via last-connection semantics" is a *senior systems-testing story*, and
almost nobody interviewing against you has one. The likely follow-up —
"how did you make it deterministic?" — you now answer with state-vector
convergence and condition-based waits, from experience.

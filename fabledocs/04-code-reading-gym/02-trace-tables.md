# Trace Tables

Fill in the blank cells *from the code*, then check against the completed rows. Columns: Step | File:lines | Value shape at this point | Owner (layer) | Transformation | Risk. Partial tables are provided; the blanks are the drill.

## Trace 1: UI → API — user stars a document

Route: `stars.create`. Start: [app/models/Document.ts](../../app/models/Document.ts) star action → [StarsStore](../../app/stores/StarsStore.ts) → [server/routes/api/stars](../../server/routes/api/stars) → [starCreator](../../server/commands/starCreator.ts).

| Step | File | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | app UI component | `documentId: string` | UI | click → store call | double-click double-star |
| 2 | StarsStore | ? | client cache | optimistic add? (check!) | ? |
| 3 | ApiClient | `POST /api/stars.create {documentId}` | transport | JSON + CSRF header | retry duplicating? |
| 4 | stars route | ? | API | validate → authorize("star", doc) | ? |
| 5 | starCreator | `{index: fractional}` | command | computes sort index | concurrent index collision |
| 6 | response + websocket | `star + policies` | presenter | — | echo handling in store |

Your blanks: steps 2 and 4 — open the files and fill shapes + risks. Answer-key hints: does the store add before or after the server confirms (read the code — the answer defines whether star UI can lie); the route authorizes `star` on Document ([policy L66–75](../../server/policies/document.ts#L66-L75)).

## Trace 2: Persistence — publishing a draft

`documents.update` with `publish: true`, collection specified.

| Step | File | What's written | Owner | Risk |
| --- | --- | --- | --- | --- |
| 1 | [documents.ts L1276–1278](../../server/routes/api/documents/documents.ts#L1276-L1279) | nothing — `authorizeDocumentPublish` | API | publish into foreign collection |
| 2 | [documentUpdater L110–114](../../server/commands/documentUpdater.ts#L110-L118) | delegates to `document.publish(ctx, …)` | command | — |
| 3 | Document.publish (find it in [Document.ts](../../server/models/Document.ts)) | ? | model | ? |
| 4 | collection.documentStructure update | ? | ? | the denormalization! |
| 5 | events row `documents.publish` | audit + bus | model hook | post-commit publish gap |
| 6 | async: notifications, revision, search, webhooks | see [Flow 5/6](../01-codebase-cartography/05-key-flows.md) | processors | idempotency |

Fill steps 3–4 by reading `Document.publish`: which tables change, in what order, and under whose transaction? Key check: who adds the document to the collection's structure tree, and what happens if that write fails after the document row committed? (It shouldn't be possible — same transaction. Verify.)

## Trace 3: Auth — request with an expired API key

| Step | File:lines | Value | Outcome |
| --- | --- | --- | --- |
| 1 | [authentication.ts L84–133](../../server/middlewares/authentication.ts#L84-L133) | header `Bearer ol_…` → `{token, transport:"header"}` | continue |
| 2 | [L200–205](../../server/middlewares/authentication.ts#L200-L241) | `ApiKey.match(token)` true | API-key branch |
| 3 | `ApiKey.findByToken` | hashed lookup ([ApiKey model](../../server/models/ApiKey.ts)) | found |
| 4 | [L218–220](../../server/middlewares/authentication.ts#L218-L224) | `expiresAt < now` | throw `AuthenticationError("API key is expired")` |
| 5 | [apiErrorHandler](../../server/routes/api/middlewares/apiErrorHandler.ts) | error → response | ? status, ? body shape |
| 6 | client | [ApiClient error mapping](../../app/utils/ApiClient.ts) | ? typed error, ? handler |

Fill 5–6: what exact JSON does the client receive, and which typed error class does ApiClient raise? Then the senior question: why does step 3 hash the token before lookup (storage of credentials), and why does `canAccess` (L222) exist even after a valid key? (**Least privilege** — scoped keys.)

## Trace 4: Error path — Zod rejection vs policy rejection

Same endpoint (`documents.update`), two bad requests: (a) `title: 12345` (wrong type), (b) valid shape but no access.

| | (a) validation | (b) authorization |
| --- | --- | --- |
| Stopped at | [validate.ts L13–24](../../server/middlewares/validate.ts#L11-L27) | [cancan authorize](../../server/policies/cancan.ts#L182-L191) |
| DB touched? | no | yes — document was loaded (scoped) |
| Transaction? | never opened? or opened-and-rolled-back? (check middleware order at [L1251–1254](../../server/routes/api/documents/documents.ts#L1251-L1255)!) | opened, rolled back by throw |
| Status/body | ? | ? |
| Client sees | ? | ? |
| Audit trail | none | none — authorization failures aren't evented (should they be? judgment question) |

Fill the `?`s. The middleware-order detail in row 3 is the drill's payoff: `validate` runs *before* `transaction` in that chain — confirm by reading the route signature, then state why that ordering saves resources.

## Trace 5: Async — from `documents.update` event to a teammate's email

| Step | Queue/process | Data | Latency source |
| --- | --- | --- | --- |
| 1 | `globalEvents` queue | Event row JSON | commit → pickup |
| 2 | worker fan-out ([worker.ts L53–88](../../server/services/worker.ts#L53-L88)) | `{event, name}` per processor | — |
| 3 | debounce → `documents.update.debounced` | ? (read [DebounceProcessor](../../server/queues/processors/DebounceProcessor.ts)) | the debounce window |
| 4 | [RevisionsProcessor](../../server/queues/processors/RevisionsProcessor.ts) | revision row + `revisions.create` event | — |
| 5 | [NotificationsProcessor → RevisionCreatedNotificationsTask](../../server/queues/tasks/RevisionCreatedNotificationsTask.ts) | Notification rows per subscriber | recipient computation |
| 6 | email queue → [DocumentPublishedOrUpdatedEmail](../../server/emails/templates/DocumentPublishedOrUpdatedEmail.tsx) | rendered HTML | SMTP |

Fill step 3 (this forces the DebounceProcessor read flagged as *investigate* elsewhere). Then answer: end-to-end, what's the minimum and typical delay between "I stop typing" and "subscriber gets email," and which two knobs would you tune?

---

**Rubric (all traces):** Basic = filled blanks plausibly. Solid = filled from code with line references. Strong = per trace, you added one row the table missed (there's at least one legitimate extra step in each) and identified the step you'd instrument in production.

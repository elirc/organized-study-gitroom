# Key Flows

Seven end-to-end traces. These rotate through the rest of the curriculum — drills, tickets, and interview cards each pick different ones. Read each with a finger on the actual files.

---

## Flow 1: Schedule a post (UI → API → DB → Temporal → social network)

Why this flow matters: it's the product. It also demonstrates the repo's core architectural bet — delegating *time* to Temporal instead of cron/queues — and crosses every layer you'd be asked about in a system-design round.

Open these files first:
- [posts.controller.ts](../../../apps/backend/src/api/routes/posts.controller.ts#L170-L179) — entry, `@CheckPolicies` quota gate
- [posts.service.ts](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L750-L808) — orchestration
- [posts.repository.ts](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L488-L631) — persistence
- [post.workflow.v1.0.5.ts](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L52-L269) — durable execution

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Frontend | [add.edit.modal.tsx](../../../apps/frontend/src/components/new-launch/add.edit.modal.tsx#L36-L60), [store.ts](../../../apps/frontend/src/components/new-launch/store.ts#L28-L60) | Composer builds per-channel values in a zustand store | `{type, date, posts:[{integration, settings, value:[{content, image[], delay}]}]}` | Client state is the only shape guarantee until step 3 |
| 2 | API | [posts.controller.ts](../../../apps/backend/src/api/routes/posts.controller.ts#L170-L179) | `POST /posts`; body typed `any`; full body `console.log`ged | raw JSON, up to 50 MB ([main.ts](../../../apps/backend/src/main.ts#L58-L60)) | PII in logs; no framework validation at the door |
| 3 | Service | [mapTypeToPost](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L243-L292) | Looks up each integration **scoped by org**, stamps `settings.__type`, then runs a *manual* `ValidationPipe` against [CreatePostDto](../../../libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts#L93-L125) | DTO with per-provider settings chosen by discriminator ([create.post.dto.ts](../../../libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts#L71-L80)) | Validation happens *after* the org lookup; content sanitized via `@Transform` |
| 4 | Service | [createPost](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L756-L775) | Optionally rewrites links to short links; strips links for providers that demand it | `string[]` messages | External shortlink service on the write path |
| 5 | Repository | [createOrUpdatePost](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L488-L631) | Upserts one row per thread item, chains `parentPostId`, rewrites tags, soft-deletes the old group on edit | `Post` rows; `group` uuid ties a thread | **No transaction** around ~4 write steps — partial failure leaves orphans |
| 6 | Service | [startWorkflow](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L690-L748) | Terminates any running workflow for the post, then `workflow.start('postWorkflowV105', {workflowId: 'post_'+postId, ...})` with search attributes | workflow args `{taskQueue, postId, organizationId}` | Fire-and-forget with silent `catch {}` — post can be QUEUE'd with no workflow ([L747](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L747)); mitigated by Flow 5 |
| 7 | Orchestrator | [post.workflow.v1.0.5.ts](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L96-L102) | `sleep(publishDate - now)` — durable timer, survives restarts/deploys | — | Clock is Temporal's, not Node's |
| 8 | Orchestrator | [same, L104-L158](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L104-L158) | Re-checks state, `refreshNeeded`, `disabled` at fire time; decides whether thread items become comments | slimmed posts ([slimPost](../../../apps/orchestrator/src/activities/post.activity.ts#L28-L52)) | Checks are point-in-time, not locks |
| 9 | Activity | [postSocial](../../../apps/orchestrator/src/activities/post.activity.ts#L203-L264) | Subscription gate (only if Stripe configured), HTML-strip content, resolve media, call provider `post()` | `PostDetails[]` → `PostResponse[]` | Activity retried by Temporal (max 3, [L19-L29](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L19-L29)) — provider call must tolerate replays |
| 10 | Provider | [x.provider.ts post()](../../../libraries/nestjs-libraries/src/integrations/social/x.provider.ts#L427) | Uploads media, creates tweet(s) | provider-specific | Typed failures: `RefreshToken` / `BadBody` ([social.abstract.ts](../../../libraries/nestjs-libraries/src/integrations/social.abstract.ts#L5-L27)) |
| 11 | Workflow | [L219-L237](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L219-L237) | On `refresh_token` failure: refresh token via activity, swap it into memory, retry loop (max 5 via `iterate`) | — | Two retry systems stacked (Temporal's + the manual loop) |
| 12 | Activity | [updatePost](../../../apps/orchestrator/src/activities/post.activity.ts#L108-L111) → [repo](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L367-L378) | Marks `PUBLISHED`, stores `releaseId`/`releaseURL`; notification + webhooks + plugs follow | — | Webhook fan-out unsigned (Flow 4) |

Validation and authorization: quota via `@CheckPolicies([Create, POSTS_PER_MONTH])` → [PoliciesGuard](../../../apps/backend/src/services/auth/permissions/permissions.guard.ts#L22-L64) → [PermissionsService.check](../../../apps/backend/src/services/auth/permissions/permissions.service.ts#L39-L172); tenant isolation by the org-scoped integration lookup in step 3 and `organizationId` connect in step 5.

Persistence and side effects: Post rows + TagsPosts; Temporal workflow; short-link service; later notifications, emails, webhooks, plugs.

Tests that cover it: **No direct coverage found** — zero spec files exist in the repo (verification log).

What juniors usually miss: the *two* validation moments (none at the controller, full DTO in the service); that `type: 'update'` skips workflow start ([L791](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L791-L798)).

What seniors notice: the missing transaction in step 5; the silent catch in step 6 creating a gap between DB truth ("QUEUE") and scheduler truth (no workflow) — and that the system *knows* this, because Flow 5 exists to sweep up the debris. That is a designed compensation, worth saying in interviews.

Interview angle: "Design a post scheduler" — this flow *is* the answer sheet, including the failure modes. See [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md).

Drill: with the app running, schedule a post 5 minutes out, then find its workflow in the Temporal UI (search attribute `postId`). Kill the orchestrator process, restart it, watch the post still publish.
Self-grade — Basic: you can name the four services involved. Solid: you can explain why step 7 survives a restart. Strong: you can articulate the step-6 gap, the compensating cron, and what you'd change (transactional outbox — see [03-architecture-and-patterns/04](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md)).

---

## Flow 2: Connect a social channel (OAuth with Redis-held state)

Why this flow matters: it's the repo's auth *for third parties* — a different problem from user auth — and a textbook "state parameter as capability" implementation with real trade-offs.

Open these files first:
- [integrations.controller.ts](../../../apps/backend/src/api/routes/integrations.controller.ts#L193-L254) — leg 1, generates the URL
- [no.auth.integrations.controller.ts](../../../apps/backend/src/api/routes/no.auth.integrations.controller.ts#L42-L307) — leg 2, the callback

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | API (authed) | [getIntegrationUrl](../../../apps/backend/src/api/routes/integrations.controller.ts#L193-L254) | Validates provider is allowlisted; provider generates `{url, state, codeVerifier}` (PKCE) | strings | `@CheckPolicies([Create, CHANNEL])` enforces plan channel count |
| 2 | Redis | [same, L229-L248](../../../apps/backend/src/api/routes/integrations.controller.ts#L229-L248) | Writes `login:{state}`, `organization:{state}`, plus optional `refresh:`/`redirect:`/`onboarding:` keys, TTL 1h | key→string | State is the *only* link back to the org — Redis flush mid-flow kills all pending connects |
| 3 | Browser | provider site | User consents; provider redirects to frontend with `?code&state` | — | — |
| 4 | API (**unauthenticated**) | [connectSocialMedia](../../../apps/backend/src/api/routes/no.auth.integrations.controller.ts#L42-L94) | Redeems `login:{state}` and `organization:{state}` from Redis, deletes them (single-use) | — | Endpoint has `@CheckPolicies` but the global guard **bypasses** `/integrations/social-connect` by path ([permissions.guard.ts L24-L31](../../../apps/backend/src/services/auth/permissions/permissions.guard.ts#L24-L31)) — the state token is the entire authentication |
| 5 | Provider | [same, L107-L176](../../../apps/backend/src/api/routes/no.auth.integrations.controller.ts#L107-L176) | `authenticate({code, codeVerifier})` exchanges the code for tokens; wrapped in a `new Promise(async …)` anti-pattern to normalize errors | `AuthTokenDetails` | eslint-disabled `no-async-promise-executor`; rejection would be unhandled |
| 6 | Service | [createOrUpdateIntegration call, L212-L240](../../../apps/backend/src/api/routes/no.auth.integrations.controller.ts#L212-L240) | Upserts the Integration keyed by `(organizationId, internalId)` ([schema L345](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L345)); custom-instance details encrypted with `fixedEncryption`, but access/refresh tokens stored **plaintext** ([schema L322-L325](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L314-L356)) | Integration row | DB dump = live social credentials |
| 7 | Service | [L242-L246](../../../apps/backend/src/api/routes/no.auth.integrations.controller.ts#L242-L246) | Kicks off a token-refresh workflow (fire-and-forget) | — | — |

Validation and authorization: leg 1 fully authed + plan-gated; leg 2 authenticates *by possession of state*. Refresh flows additionally assert the reconnected account is the same one ([L186-L190](../../../apps/backend/src/api/routes/no.auth.integrations.controller.ts#L186-L190)).

Tests that cover it: No direct coverage found.

What juniors usually miss: why leg 2 can't just use the auth cookie (the OAuth dance may land in a fresh browser context; and extension flows have no cookie at all).

What seniors notice: single-use deletion of state keys (replay defense), the 1-hour TTL trade-off, and that `checkPreviousConnections` blocks trial users from re-connecting previously seen channels ([L201-L210](../../../apps/backend/src/api/routes/no.auth.integrations.controller.ts#L201-L210)) — an anti-abuse invariant living in a controller.

Interview angle: "Explain OAuth2 + PKCE" with a real implementation, including where `state` lives and why it must be single-use. Card Q22 in [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md).

Drill: list every Redis key written in leg 1 and prove each is deleted (or expires) in leg 2. One isn't always deleted — find it.
Self-grade — Basic: name the two legs. Solid: explain PKCE's `codeVerifier` path through Redis. Strong: identify `webhookUrl:{state}` handling and the plaintext-token risk, with the fix you'd propose.

---

## Flow 3: Login and org resolution (every authenticated request)

Why this flow matters: it's the boundary in front of *everything*; multi-tenancy starts here.

Open these files first:
- [auth.middleware.ts](../../../apps/backend/src/services/auth/auth.middleware.ts#L33-L104)
- [api.module.ts](../../../apps/backend/src/api/api.module.ts#L48-L110) — which controllers sit behind it
- [helpers auth.service.ts](../../../libraries/helpers/src/auth/auth.service.ts#L35-L56) — JWT + bcrypt

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | API | [auth.controller.ts](../../../apps/backend/src/api/routes/auth.controller.ts) → [routeAuth](../../../apps/backend/src/services/auth/auth.service.ts#L35-L113) | Register/login; register lowercases email, **login does not** ([L46-L49](../../../apps/backend/src/services/auth/auth.service.ts#L42-L49)) | `CreateOrgUserDto`/`LoginUserDto` | Case-mismatch lockout for legacy rows; unique key is `(email, providerName)` ([schema L117](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L117)) |
| 2 | Helper | [signJWT](../../../libraries/helpers/src/auth/auth.service.ts#L42-L44) | JWT payload = **entire user row minus password**, signed, **no expiry** | JWT in `auth` cookie | Tokens live forever; stale claims (e.g. `isSuperAdmin`) persist until re-login |
| 3 | Middleware | [auth.middleware.ts L33-L48](../../../apps/backend/src/services/auth/auth.middleware.ts#L33-L48) | Reads `auth` cookie or header, verifies, requires `activated` | `User` | Any error → generic 403 |
| 4 | Middleware | [L50-L76](../../../apps/backend/src/services/auth/auth.middleware.ts#L50-L76) | Super-admin + `impersonate` cookie → swap `req.user`/`req.org` to the target | — | Support power; audit trail is only what you log |
| 5 | Middleware | [L79-L99](../../../apps/backend/src/services/auth/auth.middleware.ts#L79-L99) | Loads user's orgs, filters disabled memberships, picks org by `showorg` header or first; generates org `apiKey` if missing | `req.org` | `organization[0]` when list is empty throws inside the try → 403 (invisible failure cause); the `!organization` check on an array is dead code |
| 6 | Guard | [PoliciesGuard](../../../apps/backend/src/services/auth/permissions/permissions.guard.ts#L22-L64) | CASL ability built per-request from subscription/plan ([PermissionsService.check](../../../apps/backend/src/services/auth/permissions/permissions.service.ts#L39-L172)); throws `SubscriptionException` on limit | — | Self-hosted (no Stripe key) short-circuits to allow-all ([L50-L63](../../../apps/backend/src/services/auth/permissions/permissions.service.ts#L50-L63)) |
| 7 | Controller | [@GetOrgFromRequest()](../../../libraries/nestjs-libraries/src/user/org.from.request.ts) | Handlers receive `org` and must pass `org.id` down — isolation is **by convention**, not by construction | — | One forgotten `orgId` in a repo where-clause = cross-tenant bug (a real one exists: [editTag](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L804-L814)) |

Authorization vocabulary: authentication = steps 3–5 (who are you, which org context); authorization = step 6 (may you do this here) + per-query org scoping (may you touch *that row*). The last one is where this repo is weakest — see [03-architecture-and-patterns/03](../03-architecture-and-patterns/03-validation-auth-and-permissions.md).

Tests: No direct coverage found.

Interview angle: "JWT vs session?" — answer with this repo: JWT-in-cookie, no expiry, whole-user payload; then critique it (short-lived + refresh, minimal claims). Card Q20 in [08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md).

Drill: trace a request to `GET /posts` from cookie to SQL and write down every place `org.id` appears.
Self-grade — Strong: you can name the exact line where a missing org filter would be caught… and explain that no such place exists (that's the point).

---

## Flow 4: Publish-time fan-out — notifications, webhooks, plugs

Why this flow matters: side effects after the critical action; teaches at-least-once thinking, unsigned webhooks, and delay-ordered job lists.

Open: [post.workflow.v1.0.5.ts L271-L437](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L271-L437), [sendWebhooks activity](../../../apps/orchestrator/src/activities/post.activity.ts#L310-L336).

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | Workflow | [L272-L276](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L271-L276) | `sendWebhooks` for org's hooks (all, or those bound to this integration) | — |
| 2 | Activity | [L321-L335](../../../apps/orchestrator/src/activities/post.activity.ts#L310-L336) | Plain `fetch(url, POST, JSON)`; errors swallowed per-hook | **No signature, no retry, no delivery record** — receiver can't verify sender; failures invisible |
| 3 | Workflow | [L279-L315](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L279-L315) | Collects internal plugs (settings-driven, e.g. "repost from these accounts") + global plugs (metadata-driven with `totalRuns` expansion) + optional repeat-post entry; sorts by delay | Long-lived workflow — a 7-day repeat keeps it running for a week |
| 4 | Workflow | [L318-L437](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L318-L437) | `while(list)` → `sleep(delay)` → process; global plug success cancels its future runs by splicing the list; repeat-post spawns a **child workflow** with `parentClosePolicy: 'ABANDON'` ([L417-L436](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L417-L436)) | Splice-while-iterating logic; manual retry loops copy-pasted per branch |

What seniors notice: webhook delivery is fire-and-forget *inside an activity that Temporal will retry* — a partial success on attempt 1 means duplicates on attempt 2. Idempotency is the receiver's problem, and nothing tells them (no delivery id header). Contrast with Stripe's incoming webhook (Flow 6) which *is* signature-verified.

Interview angle: "How would you design outgoing webhooks?" — signing (HMAC + timestamp), retries with backoff, delivery log, dead-letter. You can cite exactly what's missing here. Card Q26.

Drill: sketch the webhook payload a receiver gets ([getPostByForWebhookId](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L844-L868)) and design the signature header you'd add.

---

## Flow 5: Missed-post recovery (the safety net)

Why this flow matters: the compensating control for every fire-and-forget in Flows 1–4; the difference between "we hope" and "we guarantee eventually".

Open: [missing.post.workflow.ts](../../../apps/orchestrator/src/workflows/missing.post.workflow.ts), [searchForMissingThreeHoursPosts activity](../../../apps/orchestrator/src/activities/post.activity.ts#L73-L106), [repo query](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L36-L64).

| Step | What happens | Anchor |
| --- | --- | --- |
| 0 | On backend boot, **only if `RUN_CRON` is set**, a `missingPostWorkflow` is started with fixed id `missing-post-workflow`; the workflow itself is an infinite loop that runs the sweep, then `sleep('1 hour')` forever — a durable cron | [infinite.workflow.register.ts L8-L19](../../../libraries/nestjs-libraries/src/temporal/infinite.workflow.register.ts#L8-L19), [missing.post.workflow.ts L13-L19](../../../apps/orchestrator/src/workflows/missing.post.workflow.ts#L13-L19) |
| 1 | The sweep scans for posts still `QUEUE` with `publishDate` in `[now-2d, now)`, healthy integration only | repo L36-L64 |
| 2 | For each, `signalWithStart('postWorkflowV105', …, workflowIdConflictPolicy: 'USE_EXISTING', signal: 'poke')` — starts the workflow if missing, pokes it if present | activity [L77-L105](../../../apps/orchestrator/src/activities/post.activity.ts#L73-L106) |
| 3 | Deterministic `workflowId = post_{postId}` makes this **idempotent**: can't double-start | same |

What juniors miss: why `USE_EXISTING` + a stable workflowId matters — that pair is the idempotency key. What seniors notice: the 2-day lower bound silently abandons anything older; the whole safety net is **off by default** (no `RUN_CRON`, no sweep — a deployment invariant that lives in an env var); and the `poke` signal handler sets a flag the v1.0.5 workflow never reads ([L75-L78](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L75-L78)) — vestigial code worth a ticket.

Interview angle: "How do you make a distributed action exactly-once?" Honest answer: you don't; you make it at-least-once + idempotent. This flow is your concrete example.

Drill: write the SQL (or Prisma filter) for "posts that will never publish and nothing will save" — states/conditions the sweep excludes.

---

## Flow 6: Stripe webhook → subscription state

Why this flow matters: incoming money events; the one place *inbound* verification is done right, and a good contrast for Flow 4.

Open: [stripe.controller.ts](../../../apps/backend/src/api/routes/stripe.controller.ts#L18-L53), [main.ts rawBody](../../../apps/backend/src/main.ts#L22-L24).

| Step | What happens | Anchor |
| --- | --- | --- |
| 1 | `POST /stripe` is **outside** the auth middleware (not in `authenticatedController` — [api.module.ts L71-L81](../../../apps/backend/src/api/api.module.ts#L69-L81)) | — |
| 2 | Signature verified against the **raw body** (`rawBody: true` at bootstrap — required because any re-serialization breaks HMAC) | controller L19-L25 |
| 3 | Filters events by `metadata.service === 'gitroom'` so multiple products can share a Stripe account | L28-L35 |
| 4 | Switch on event type → subscription create/update/delete in [SubscriptionService](../../../libraries/nestjs-libraries/src/database/prisma/subscriptions/subscription.service.ts) | L38-L49 |
| 5 | Downstream: [PermissionsService](../../../apps/backend/src/services/auth/permissions/permissions.service.ts#L39-L172) reads subscription per request; posting activities re-check it ([post.activity.ts L114-L124](../../../apps/orchestrator/src/activities/post.activity.ts#L113-L135)) | — |

What seniors notice: handler returns synchronously — a slow DB write risks Stripe timeout + redelivery; is `createSubscription` idempotent under redelivery? (Investigate — Stripe retries are at-least-once.)

Interview angle: "How do you verify a webhook?" — raw body + signature + replay window. You've now seen both the right way (here) and the missing way (Flow 4).

Drill: explain why `express.json()` before signature verification would break this, in two sentences.

---

## Flow 7: Media upload (multipart → storage → post attachment)

Why this flow matters: file handling is a classic interview minefield (validation, storage abstraction, URL vs path duality).

Open: [media.controller.ts](../../../apps/backend/src/api/routes/media.controller.ts#L87-L152), [upload.factory](../../../libraries/nestjs-libraries/src/upload/upload.factory.ts), [updateMedia](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L340-L437).

| Step | What happens | Anchor | Risk |
| --- | --- | --- | --- |
| 1 | `POST /media/upload-server` (multer) or `upload-simple`; `CustomFileValidationPipe` gates type/size | controller L87-L102, L130-L152 | Validation pipe is the whole defense — read it before trusting it |
| 2 | `UploadFactory.createStorage()` picks local disk vs Cloudflare R2 from env — a **strategy pattern** chosen at process start | controller [L32](../../../apps/backend/src/api/routes/media.controller.ts#L32) | Static singleton: can't vary per-org |
| 3 | Media row saved org-scoped (`Media.organizationId`, [schema L209-L231](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L209-L231)) | — | — |
| 4 | At publish, `updateMedia` re-resolves ids → paths/URLs, optionally converts PNG→JPEG via sharp for picky providers, silently returns original list on any error | posts.service.ts L340-L437 | `catch → return imagesList` hides broken media until the provider rejects it |
| 5 | R2 multipart route `POST /media/:endpoint` proxies presigned operations | controller L154-L179 | `@Param('endpoint')` routing — order matters vs the other `/media/*` POSTs |

Interview angle: "Where do you validate uploads and why not trust the file extension?" — both the internal and public upload paths here sniff **magic bytes** with `file-type` rather than trusting the client's mimetype ([custom.upload.validation.ts L21-L41](../../../libraries/nestjs-libraries/src/upload/custom.upload.validation.ts#L21-L41), and the public API's own allowlist at [public.integrations.controller.ts L42-L52](../../../apps/backend/src/public-api/routes/v1/public.integrations.controller.ts#L42-L52)). That's the right answer; be ready to say why (`Content-Type` is attacker-controlled).

Drill: read `CustomFileValidationPipe` end to end and list the three rejection conditions; then say which attack each one blocks.

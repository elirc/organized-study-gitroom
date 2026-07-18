# Side Effects, Async & Reliability

Every side effect is a promise to the outside world you might fail to keep. This file maps them and grades their reliability posture.

## The side-effect map

| Side effect | Where triggered | Reliability mechanism | Grade |
| --- | --- | --- | --- |
| Publish to social API | [postSocial activity](../../../apps/orchestrator/src/activities/post.activity.ts#L203-L264) | Temporal retry (3) + in-workflow refresh loop (5); typed failures | B — but **not idempotent at provider** (retry can duplicate) |
| Token refresh | [refreshTokenWithCause](../../../apps/orchestrator/src/activities/post.activity.ts#L388-L415) + [read-path refresh](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L100-L123) | retry + disconnect on hard fail | C — two independent refreshers can race |
| Outgoing webhooks | [sendWebhooks activity](../../../apps/orchestrator/src/activities/post.activity.ts#L310-L336) | none (fire-and-forget `fetch`, errors swallowed) | D — no signing, no retry, no delivery log, duplicates on activity replay |
| In-app + email notification | [inAppNotification](../../../apps/orchestrator/src/activities/post.activity.ts#L266-L283) | via Temporal activity (retried) | B |
| Digest / streak / missing-post | dedicated workflows ([digest.email](../../../apps/orchestrator/src/workflows/digest.email.workflow.ts), [streak](../../../apps/orchestrator/src/workflows/streak.workflow.ts), [missing.post](../../../apps/orchestrator/src/workflows/missing.post.workflow.ts)) | durable + idempotent ids | A |
| Short-link creation | [createPost L764-L770](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L756-L775) | on the request path, no compensation | C |
| Media re-upload (PNG→JPEG) | [updateMedia](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L373-L423) | try/catch → return original | C — hides failure |
| Workflow start | [startWorkflow](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L690-L748) | swallowed error + missing-post sweep | B (sweep exists) / D (if `RUN_CRON` unset) |
| Stripe subscription writes | [stripe.controller](../../../apps/backend/src/api/routes/stripe.controller.ts#L37-L52) | inbound signature verified; Stripe redelivers | B — idempotency under redelivery: investigate |

## The reliability vocabulary, grounded

- **Idempotency** (safe to run twice): achieved for scheduling via deterministic `workflowId = post_{postId}` + `USE_EXISTING`/`TERMINATE_EXISTING` ([activity L79-L84](../../../apps/orchestrator/src/activities/post.activity.ts#L73-L106), [service L723-L746](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L722-L748)); **absent** for provider posting and webhooks.
- **Retries**: Temporal activity policy (max 3, backoffCoefficient 1 = fixed interval — [workflow L19-L29](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L19-L29)); provider `fetch` fixed-5s recursion ([social.abstract L127-L152](../../../libraries/nestjs-libraries/src/integrations/social.abstract.ts#L101-L152)); manual 5-iteration refresh loop. Three retry systems stacked — reason about them together (debugging scenario 2).
- **At-least-once** is the real guarantee everywhere; "exactly once" is approximated by idempotency where it exists.
- **Compensation** (reconciliation): the [missing-post sweep](../../../apps/orchestrator/src/workflows/missing.post.workflow.ts) repairs lost workflow starts. No compensation for lost webhooks or short-links.
- **Backpressure / rate limits**: per-provider Temporal task queues + `maxConcurrentJob` ([x.provider L35](../../../libraries/nestjs-libraries/src/integrations/social/x.provider.ts#L26-L45)) serialize calls to rate-limited APIs; global API throttle 90/h ([app.module L35-L43](../../../apps/backend/src/app.module.ts#L35-L43)).
- **Timeouts**: `startToCloseTimeout: '10 minute'` per activity — long enough for video uploads, long enough to hide a hung provider.
- **Observability of failure**: `state=ERROR` + `Errors` row + notification is the failure-visibility path ([changeState L413-L425](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L393-L428)); webhook failures have none.

## The outbox pattern (the missing piece, and the senior conversation)

Today: DB commit, then fire-and-forget workflow start ([service L777-L798](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L777-L798)); a swallowed start error + a compensating sweep. This is a *poor-man's outbox*: the DB row is the intent, the sweep is the dispatcher, but the two are coupled by a 2-day scan rather than a durable queue.
Proper transactional outbox: write the post **and** an `outbox` row in one transaction; a dispatcher drains outbox → starts workflows → marks sent. Guarantees no lost start without a time-bounded sweep. Senior project S3.

## Where side effects sit in risky places

- Webhook `fetch` inside a **retried** activity → duplicates ([activity L321-L335](../../../apps/orchestrator/src/activities/post.activity.ts#L310-L336)).
- Token refresh on the **HTTP request path** with a 10s sleep ([posts.service L116-L118](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L96-L123)) blocks a request thread.
- `updateMedia` re-uploads a file as a side effect of a *read* ([getPost → updateMedia](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L527-L547)) — reads should be idempotent; this one mutates storage.

Interview angle:
1. "How do you guarantee a job runs exactly once?" → you don't; at-least-once + idempotency key; show the workflowId scheme.
2. "Design reliable outgoing webhooks" → sign, retry w/ backoff, delivery log, dead-letter — everything Flow 4 lacks.
3. "What's the outbox pattern and when do you need it?" → the post-scheduling gap here is the motivating example.
4. "Where should side effects *not* live?" → in reads; in retried units without idempotency; on the request thread.

Drill: for each row graded C or D above, write the one change that raises it a grade, and its blast radius. Rank by (value ÷ blast radius). Your ranking is a mini roadmap — compare it to [06-architecture-critique.md](06-architecture-critique.md).

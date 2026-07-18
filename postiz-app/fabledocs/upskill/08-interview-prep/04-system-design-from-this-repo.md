# System Design From This Repo

The whiteboard exercise: **"Design Buffer/Hootsuite — a social-media post scheduler."** You have studied a production implementation; this file turns that into a 40-minute answer, showing at each step what Postiz actually chose (with anchors), what a simpler/stronger alternative looks like, and what junior/mid/senior answers sound like.

Companion file: [03-architecture-and-patterns/06-architecture-critique.md](../03-architecture-and-patterns/06-architecture-critique.md) — the same material as critique instead of design.

## Step 1 — Requirements (5 min)

Functional: connect channels via OAuth (30+ providers), compose posts (threads, media, per-platform settings), schedule/publish at time T, retries + token refresh, calendar view, teams/organizations, plan limits, webhooks out, analytics in.
Non-functional: publish-time accuracy (±1 min fine), **durability of scheduled intent** (a post scheduled Friday must survive a Saturday deploy), tenant isolation, provider rate limits, and "self-hostable" (a real Postiz constraint that shapes everything — e.g. billing checks vanish without a Stripe key: [permissions.service.ts L50-L63](../../../apps/backend/src/services/auth/permissions/permissions.service.ts#L50-L63)).

- Junior answer: lists features.
- Mid answer: separates *scheduling durability* from *delivery guarantees*, asks about scale (posts/day, channels/org).
- Senior answer: names the invariant early — "a QUEUE'd post must eventually publish or become a visible ERROR; nothing silently disappears" — then designs to protect it.

## Step 2 — API sketch (5 min)

What Postiz chose: REST-ish JSON under a NestJS app; the write endpoint is `POST /posts` taking `{type: draft|schedule|now|update, date, posts:[{integration, settings, value:[{content, image[], delay}]}]}` ([create.post.dto.ts L93-L125](../../../libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts#L93-L125)); one endpoint handles create *and* edit *and* immediate-post (discriminated by `type`). Auth = JWT cookie + `showorg` header for org context ([auth.middleware.ts L33-L99](../../../apps/backend/src/services/auth/auth.middleware.ts#L33-L104)). A separate public API `/public/v1` for API-key access ([public.integrations.controller.ts](../../../apps/backend/src/public-api/routes/v1/public.integrations.controller.ts#L63-L76)).

Alternative: separate `POST /posts` vs `PUT /posts/:group`; idempotency keys on create (Postiz has none — retrying a timed-out create duplicates the post; say this unprompted and you sound senior).

## Step 3 — Data model (8 min)

What Postiz chose ([schema.prisma](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L393-L446)):

```
Organization 1—N UserOrganization N—1 User        (membership + role)
Organization 1—N Integration (channel; token, providerIdentifier, internalId; unique(orgId, internalId))
Organization 1—N Post
Post: state QUEUE|PUBLISHED|ERROR|DRAFT, publishDate, group (thread id),
      parentPostId (thread linked-list), settings/image as JSON strings,
      releaseId/releaseURL (remote ids), intervalInDays (recurrence)
Subscription 1—1 Organization (Stripe mirror)
```

Talking points with anchors:
- Threads as an **adjacency list** chained during an upsert loop ([posts.repository.ts L488-L631](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L488-L631)); alternative: `position` column, simpler reads, no recursion ([getPostsRecursively](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L294-L322)).
- Recurrence stored as `intervalInDays` and **expanded at read time** into virtual occurrences ([getPosts L199-L219](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L199-L219)) — no materialized occurrence rows. Trade-off: no per-occurrence state; a skipped occurrence isn't representable.
- Per-provider settings as validated-at-birth JSON strings with a class-transformer **discriminator** ([create.post.dto.ts L71-L80](../../../libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts#L71-L80)); alternative: Postgres JSONB + zod per provider.
- Soft deletes everywhere; state machine is just an enum column — nothing enforces legal transitions (`ERROR → PUBLISHED` is a plain `update` away: [changeState](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L393-L428)).

- Junior: tables and foreign keys.
- Mid: unique constraints as invariants — `(organizationId, internalId)` dedupes reconnected channels ([schema L345](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L345)); indexes match query shapes (`publishDate`, `state`, `group` all indexed: [L431-L445](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L431-L445)).
- Senior: what the model *can't* express (occurrence state, transition legality), and migration posture — this repo uses `prisma db push` with **no migration files** ([package.json L34](../../../package.json#L34)), fine for self-hosters, unacceptable for a team; you'd introduce `prisma migrate`.

## Step 4 — The scheduler (10 min — the heart)

Three classic designs; know all three, then say which Postiz picked and why:

| Design | How | Pros | Cons |
| --- | --- | --- | --- |
| DB poller (cron every minute: `WHERE state=QUEUE AND publishDate<=now`) | one query + worker pool | simple, one source of truth | polling lag, herd on the minute, locking care (`FOR UPDATE SKIP LOCKED`) |
| Delayed queue (BullMQ/SQS delay) | enqueue with delay | scales, familiar | max-delay limits, reschedule = find & remove job, Redis durability config |
| **Durable workflow (Temporal — Postiz's choice)** | workflow-per-post `sleep()`s until T ([workflow L96-L102](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L96-L102)) | survives restarts, retries/backoff built in ([L19-L29](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L19-L29)), reschedule = terminate+restart ([posts.service.ts L690-L748](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L690-L748)), per-provider task queues throttle rate-limited APIs ([L64-L73](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L63-L73)) | heavy infra (Temporal+ES+its own Postgres — [docker-compose.dev.yaml](../../../docker-compose.dev.yaml)), **two sources of truth**, workflow versioning (five versions kept: [post-workflows/](../../../apps/orchestrator/src/workflows/post-workflows)) |

The dual-source-of-truth problem is the sharpest thing you can discuss: workflow start is fire-and-forget with a swallowed error ([L747](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L747)), so Postiz runs a **compensating sweep** — an infinite workflow that re-`signalWithStart`s any QUEUE'd post from the last 2 days with a deterministic `workflowId = post_{postId}` making repair idempotent ([missing.post.workflow.ts](../../../apps/orchestrator/src/workflows/missing.post.workflow.ts#L13-L19), [post.activity.ts L73-L106](../../../apps/orchestrator/src/activities/post.activity.ts#L73-L106)).

- Junior: "use cron".
- Mid: compares two designs, picks one, handles reschedule/cancel.
- Senior: at-least-once + idempotency keys as the *actual* guarantee; the reconciliation loop as the pattern that makes fire-and-forget acceptable; workflow versioning as the tax on the Temporal choice.

## Step 5 — Delivery, retries, token refresh (5 min)

Postiz: activity retry (3 attempts) × an in-workflow refresh-retry loop (5 iterations) keyed on **typed failures** — provider code maps API error text to `RefreshToken` (retryable after refresh) or `BadBody` (terminal) ([social.abstract.ts L101-L173](../../../libraries/nestjs-libraries/src/integrations/social.abstract.ts#L101-L173), [workflow catch L219-L261](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L219-L261)). Failures set `state=ERROR`, persist an `Errors` row with the raw body ([changeState L393-L428](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L393-L428)) and notify in-app/email.
Gap to name: the provider `post()` call is retried but posting is **not idempotent at the provider** — attempt 1 may have tweeted before the timeout; attempt 2 tweets again. X's duplicate-content rejection accidentally mitigates it ([x.provider handleErrors 'duplicate-rules'](../../../libraries/nestjs-libraries/src/integrations/social/x.provider.ts#L47-L108)). Senior fix: record `releaseId` per item *before* declaring failure, or providers with idempotency tokens where available.

## Step 6 — Multi-tenancy & authz (4 min)

Org-scoped queries by convention (every repo method takes `orgId` — and the one that forgets is your cautionary tale: [editTag](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L804-L814)); plan limits as CASL abilities computed per request from live counts ([permissions.service.ts L39-L172](../../../apps/backend/src/services/auth/permissions/permissions.service.ts#L39-L172)). Alternatives: Postgres RLS (fails closed), middleware-injected Prisma filter.

## Step 7 — Scaling concerns (3 min)

- Read side: calendar month-view minified payloads ([minifyPosts](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L328-L338)); Redis caches analytics 1h ([checkPostAnalytics L212-L219](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L205-L229)).
- Write side: Temporal absorbs bursts; per-provider task queues serialize rate-limited APIs (`maxConcurrentJob = 1` on X: [x.provider.ts L35](../../../libraries/nestjs-libraries/src/integrations/social/x.provider.ts#L26-L45)).
- Global API throttle: 90 req/h/key via Redis-backed throttler ([app.module.ts L35-L43](../../../apps/backend/src/app.module.ts#L35-L43)) — surprisingly strict; discuss burst vs sustained limits.
- 10x growth pressure points: workflow count (one per post — fine, Temporal shards), the per-request quota counting queries, and month-view fetch volume.

## Variation prompts (practice each for 10 minutes)

1. **Add real-time collaboration** on the composer (two teammates editing one draft) — where does zustand break, what moves server-side (CRDT? lock? last-write-wins with presence)?
2. **10x providers (300 networks)** — the registry array and optional-member interface strain; plugin architecture, per-provider packages, conformance tests.
3. **Guaranteed exactly-once publishing** — walk the outbox pattern: post row + workflow-start intent in one transaction, dispatcher drains the outbox; compare with the existing sweep.
4. **Multi-region** — Temporal namespaces per region, token residency, publish-time locality.
5. **Remove Temporal** (infra budget cut) — design the poller replacement preserving the invariant from Step 1; what do you lose first? (Per-provider throttling, then durable retry state.)

## How to rehearse

Whiteboard the full thing once cold, 40 minutes, out loud. Then re-do Steps 4–5 only, 10 minutes, citing at least three concrete mechanisms by name (durable timer, typed failure taxonomy, compensating sweep, idempotent signalWithStart). If you can't cite mechanisms, re-trace [Flow 1](../01-codebase-cartography/05-key-flows.md) and [Flow 5](../01-codebase-cartography/05-key-flows.md).

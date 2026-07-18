# API & Data Modeling Question Cards

Fourteen cards on REST/RPC, validation, auth, schema, migrations, transactions, caching — anchored to this repo's real API and Prisma schema.

---

## Q20: "JWT vs session — which and why?"
Anchor: [signJWT no-expiry, whole-user payload](../../../libraries/helpers/src/auth/auth.service.ts#L42-L44); [middleware](../../../apps/backend/src/services/auth/auth.middleware.ts#L33-L104).
Junior: "JWT is stateless."
Mid: JWT avoids a session store but can't be revoked; this app stores the *entire user* and never expires → stale claims, no logout-everywhere. Sessions revoke easily but need shared storage.
Senior: short-lived access + refresh rotation; minimal claims (sub + orgId); the reset-password token here reinvents `exp` manually ([auth.service L223-L245](../../../apps/backend/src/services/auth/auth.service.ts#L217-L245)) and isn't single-use — a concrete "what not to do."
Follow-up: "how do you revoke a JWT?" (denylist/short TTL).

## Q21: "How do you enforce multi-tenancy?"
Anchor: org-scoped queries; the [editTag IDOR](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L804-L814).
Mid: `organizationId` in every where-clause; the risk is it's convention, so one miss = cross-tenant access.
Senior: defense in depth — app-level scoping *plus* a structural backstop (Postgres RLS or a Prisma client extension injecting orgId); cites the live bug as why convention isn't enough.
Follow-up: "which queries are *safe* to leave unscoped?" (workflow-internal calls with owned ids — [changeState](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L393-L428)).

## Q22: "Walk me through OAuth2 + PKCE."
Anchor: [leg 1 generate](../../../apps/backend/src/api/routes/integrations.controller.ts#L193-L254), [leg 2 callback](../../../apps/backend/src/api/routes/no.auth.integrations.controller.ts#L42-L307).
Mid: authorize → code → exchange for tokens; PKCE's `codeVerifier` prevents code interception; here the `state` + verifier live in Redis (TTL 1h, single-use) linking the anonymous callback to the initiating org.
Senior: why `state` must be single-use (replay), the Redis-as-single-point trade-off, and that the callback endpoint is authenticated *by possession of state* (bypasses the normal guard — [permissions.guard L27](../../../apps/backend/src/services/auth/permissions/permissions.guard.ts#L24-L31)).
Follow-up: "what if two tabs start the flow?" (distinct states).

## Q23: "Design the schema for scheduled posts with threads and recurrence."
Anchor: [Post model](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L393-L446).
Mid: `state`, `publishDate`, `group` (thread id), `parentPostId` (adjacency list), `intervalInDays` (recurrence). Draw it.
Senior: threads as adjacency list vs position column; recurrence-as-field (expanded at read, unbounded — [repo L204-L216](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L199-L219)) vs materialized occurrences (enables skip/edit); what each can't express.
Follow-up: "how do you skip one occurrence?" (you can't in the current model — that's the point).

## Q24: "Where and how do you validate request input?"
Anchor: [global ValidationPipe](../../../apps/backend/src/main.ts#L52-L56), [CreatePostDto discriminator](../../../libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts#L71-L80), the `any` gap.
Mid: DTO + class-validator at the boundary; per-provider settings via a runtime discriminator.
Senior: the `any` body ([posts.controller L174](../../../apps/backend/src/api/routes/posts.controller.ts#L170-L179)) defeats the pipe → manual re-validation, a duplicated layer; zod as a schema-first alternative unifying runtime + static types.
Follow-up: "validate vs sanitize?" (both — sanitize is fused in `@Transform` here).

## Q25: "Pagination — offset vs cursor?"
Anchor: [getPostsList offset pagination](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L222-L296) (`skip/take`, `count`, `hasMore`).
Mid: offset is simple, supports jump-to-page, but `skip` degrades on large offsets and can skip/dupe rows under concurrent writes; cursor (keyset) is stable and fast but no random access.
Senior: this list is a bounded future-posts view so offset is fine; for an unbounded feed you'd keyset on `(publishDate, id)`. Note the `count` runs every page (cost).
Follow-up: "why is cursor better for infinite scroll?"

## Q26: "Design outgoing webhooks."
Anchor: [sendWebhooks — the missing version](../../../apps/orchestrator/src/activities/post.activity.ts#L310-L336).
Mid: POST the event to registered URLs; here it's unsigned, unretried, errors swallowed.
Senior: HMAC signature + timestamp (replay window), retries with exponential backoff, delivery log + dead-letter, idempotency id so receivers dedupe — everything this repo lacks; and it runs inside a *retried* activity so it already duplicates.
Follow-up: "how does the receiver verify it's you?" (shared secret HMAC).

## Q27: "When do you use a transaction?"
Anchor: [createOrUpdatePost — no transaction](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L488-L631).
Mid: when multiple writes must be all-or-nothing; here a thread's N upserts + tag rewrite + group soft-delete aren't atomic → partial threads on crash.
Senior: `prisma.$transaction`; and the subtle point that the workflow start must stay *outside* it (must run after commit, else the worker reads uncommitted data — the outbox motivation).
Follow-up: "what isolation level and why?"

## Q28: "How do you cache, and how do you invalidate?"
Anchor: [analytics cached in Redis by date](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L205-L228) (1h prod / 1s dev).
Mid: key must encode every input (`org:post:date`); TTL-based expiry here, shorter in dev.
Senior: the two hard problems (naming things, invalidation); TTL vs event-based; the commented-out read-cache above ([L198-L203](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L198-L203)) shows someone disabled cache-read while keeping cache-write — ask why (staleness bug?).
Follow-up: "cache stampede — how do you prevent it?"

## Q29: "REST vs RPC — what is this API?"
Anchor: [posts.controller routes](../../../apps/backend/src/api/routes/posts.controller.ts) (resourceful-ish) vs action endpoints like `/should-shortlink`, `/separate-posts`.
Mid: mostly resource-oriented with RPC-style action endpoints where actions don't map to CRUD; pragmatic, common in real apps.
Senior: the `type` discriminator on `POST /posts` (create/update/now/draft in one endpoint) is RPC-in-REST-clothing; trade-off vs separate endpoints (idempotency, caching, clarity).
Follow-up: "is `POST /posts` idempotent?" (no — retry duplicates; no idempotency key).

## Q30: "How do you do a zero-downtime schema migration?"
Anchor: [package.json db-push, prod auto-push](../../../package.json#L16-L36); no migration files.
Mid: expand-contract (add → backfill → dual-write → switch → drop) across deploys.
Senior: this repo's `db push --accept-data-loss` on every prod deploy makes it *harder* — no reviewable diff, no rollback; you'd adopt `prisma migrate`; and with three processes sharing one schema, every change must be backward-compatible during the rolling window.
Follow-up: "rename a column with zero downtime — steps?"

## Q31: "How do you rate-limit an API?"
Anchor: [ThrottlerModule 90/h Redis-backed](../../../apps/backend/src/app.module.ts#L35-L43), [proxy-aware guard](../../../libraries/nestjs-libraries/src/throttler/throttler.provider.ts).
Mid: fixed window per key in Redis (shared across instances); proxy-aware to key on real client IP.
Senior: fixed-window burst problem at boundaries vs sliding window/token bucket; per-plan tiers (ticket M9); the flat 90/h is coarse.
Follow-up: "distributed rate limiting — why Redis?"

## Q32: "How do you store secrets/tokens?"
Anchor: [social tokens plaintext](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L314-L356) vs [third-party encrypted](../../../libraries/helpers/src/auth/auth.service.ts#L49-L55).
Mid: encrypt at rest; here it's inconsistent — the highest-value secrets (social OAuth tokens) are plaintext while API keys are encrypted.
Senior: envelope encryption / KMS; the fixed-IV AES here leaks equality of plaintexts; AES-GCM + random IV is the modern default; key rotation.
Follow-up: "fixed IV — what does it leak?"

## Q33: "Idempotency in APIs — how?"
Anchor: [deterministic workflowId + USE_EXISTING](../../../apps/orchestrator/src/activities/post.activity.ts#L73-L106); the *missing* idempotency on `POST /posts` and provider posting.
Mid: idempotency keys so retries don't double-act; the scheduler achieves it via stable workflow ids; the create endpoint and provider posting don't.
Senior: client-supplied `Idempotency-Key` stored server-side; at-least-once + idempotent handler as the real "exactly once."
Follow-up: "where would you add an idempotency key here first?" (`POST /posts` create).

## Q34: "Model a soft-delete correctly."
Anchor: [deletedAt everywhere](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L52-L65), [Customer unique with deletedAt](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L301-L312).
Mid: `deletedAt?` + filter every read; unique constraints must include `deletedAt` or re-creating a deleted name collides.
Senior: NULL-distinct semantics in Postgres (two deleted rows with same name are fine since NULLs differ… wait — deleted rows have a *timestamp*, actives have NULL; think it through); GDPR "right to erasure" tension with soft delete; the forgotten-filter bug class (find one: [getPostUrls](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L114-L127)).
Follow-up: "how do unique constraints interact with soft delete?"

---

Practice: for each, do a 90-second answer that hits example→trade-off→failure. Then take the follow-up cold. If you can do Q20–Q23 and Q26–Q28 fluently, you're at the mid-level API bar. System-design synthesis: [04](04-system-design-from-this-repo.md).

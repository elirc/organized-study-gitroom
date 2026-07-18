# Validation, Auth & Permissions

Three distinct questions, three distinct mechanisms — juniors blur them, interviewers separate them:
- **Validation**: is the input well-formed? (shape/type)
- **Authentication**: who is this? (identity)
- **Authorization**: may they do *this*, to *that*? (permission + ownership)

## The four validation layers

1. **Global ValidationPipe** ([main.ts L52-L56](../../../apps/backend/src/main.ts#L52-L56)) — validates typed DTOs automatically… but is bypassed wherever a param is `any` ([posts.controller createPost](../../../apps/backend/src/api/routes/posts.controller.ts#L170-L179)).
2. **Manual ValidationPipe** ([mapTypeToPost L280-L291](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L243-L292)) — re-runs DTO validation in the service for the `any` endpoint, with the per-provider discriminator.
3. **Transform-time sanitization** ([create.post.dto.ts L35](../../../libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts#L31-L36) `sanitizePostContent`; `ValidContent` custom validator) — cleaning fused into validation.
4. **File magic-byte validation** ([CustomFileValidationPipe](../../../libraries/nestjs-libraries/src/upload/custom.upload.validation.ts#L21-L41)) — sniffs bytes, ignores client mimetype.

Lesson: validation is layered and inconsistent; the `any` gap forced a duplicate layer. A single typed DTO would collapse layers 1–2.

## Authentication

Cookie/header JWT → verify → require `activated` → resolve org ([auth.middleware.ts L33-L104](../../../apps/backend/src/services/auth/auth.middleware.ts#L33-L104)). JWT is the whole user row, **never expires** ([helpers auth.service.ts L42-L44](../../../libraries/helpers/src/auth/auth.service.ts#L42-L44)). Applied only to controllers in the [authenticatedController list](../../../apps/backend/src/api/api.module.ts#L48-L68) — a **fail-open** design (unlisted = unauthenticated). Public surfaces (Stripe, OAuth callback, public API) are outside it deliberately, each with their own auth (signature, Redis state, API key).

## Authorization — two sub-mechanisms

**A. Plan/role gates (CASL)** — declarative `@CheckPolicies([action, section])` → [PoliciesGuard](../../../apps/backend/src/services/auth/permissions/permissions.guard.ts#L22-L64) → [PermissionsService.check](../../../apps/backend/src/services/auth/permissions/permissions.service.ts#L39-L172) building an ability from live counts vs [pricing](../../../libraries/nestjs-libraries/src/database/prisma/subscriptions/pricing.ts). Self-host short-circuits to allow-all. Gaps: path-string bypasses (with a duplicated `/auth`, [guard L24-L31](../../../apps/backend/src/services/auth/permissions/permissions.guard.ts#L24-L31)); counting queries per request.

**B. Resource ownership (tenant isolation)** — **not** centralized; enforced by `orgId` in each repository where-clause. This is the weak spot:
- Correct: [changeDate](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L439-L464), [deleteTag](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L816-L826), [getPost with orgId](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L339-L365).
- Broken: [editTag](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L804-L814) — takes orgId, ignores it → cross-org write (IDOR). Ticket J2.
- Trusted-internal (unscoped by design): [changeState](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L393-L428), [updatePost](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L367-L378) — called only by the workflow with ids it already owns. Knowing which unscoped queries are *safe* is the senior skill.

## What a junior misses vs what a senior checks

| Junior | Senior |
| --- | --- |
| "It has auth middleware, so it's secure" | "Is this controller in the list? Does every query carry orgId? Which queries are intentionally unscoped and are they reachable from user input?" |
| Validates shape | Also asks: authorization *after* validation? (yes — guard runs before handler, but ownership checks are inside repo calls) |
| Trusts the DTO | Notices drafts skip settings validation ([@ValidateIf L71](../../../libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts#L71-L80)) |

## IDOR audit method (do this as a drill)

1. Grep `posts.repository.ts` (and siblings) for `.update(`, `.delete(`, `.findUnique(`.
2. For each, ask: is `organizationId` in the `where`? If not, who calls it and can a user influence the id?
3. Classify: scoped / safe-internal / **vulnerable**.
You will re-find editTag and possibly others — that *is* the exercise.

Interview angle:
1. "authentication vs authorization" → this repo cleanly separates them; give the middleware-vs-guard-vs-repo story.
2. "What's IDOR and how do you prevent it?" → org-scoped queries + a live example of the failure; defense-in-depth (RLS as backstop).
3. "Where does authorization belong?" → route-level for capability, data-level for ownership; you need both.

Drill self-grade — Strong: you produced the three-way classification for ≥10 methods and proposed a *structural* fix (Prisma client extension injecting `organizationId`, or Postgres RLS) rather than "remember to add orgId."

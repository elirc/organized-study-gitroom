# Trace Tables

Fill these in yourself first (cover the "value shape" column), then compare. Columns: step, file/line, value shape, owner, transformation, risk.

## Trace 1: `content` string, composer → published tweet (UI→API→persistence→provider)

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | [store.ts values](../../../apps/frontend/src/components/new-launch/store.ts#L9-L14) | `{content: '<p>hi</p>', media:[]}` (HTML from tiptap) | browser | user types; stored as HTML | none |
| 2 | POST /posts body | `{posts:[{value:[{content}]}]}` | browser→API | JSON serialize | 50MB cap only |
| 3 | [sanitizePostContent transform](../../../libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts#L31-L36) | sanitized HTML string | DTO | XSS sanitize + ValidContent check | sanitizer correctness |
| 4 | [createPost shortlink](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L756-L775) | maybe shortened-URL string | service | link→shortlink unless provider strips | external service on write path |
| 5 | [createOrUpdatePost content](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L524) | DB column `content: String` | repo | store as-is | no transaction |
| 6 | [postSocial stripHtmlValidation](../../../apps/orchestrator/src/activities/post.activity.ts#L229-L235) | plain text (per provider editor mode) | activity | HTML→text, mention formatting | regex-based html detection L234 |
| 7 | [updateTags](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L553-L575) | text with `(post:id)`→URL | service | resolve internal post refs | whole-array JSON round-trip |
| 8 | [x.provider post()](../../../libraries/nestjs-libraries/src/integrations/social/x.provider.ts#L427) | tweet payload | provider | length check, media attach | 200/4000 char limit by premium |

Risk to name: the content is transformed at 4 places (3, 6, 7 + provider) — where does character-count validation happen relative to shortening (step 4) and mention expansion (step 6)? (After — so the client's count can disagree with the server's.)

## Trace 2: `org.id` through an authenticated request (auth trace)

| Step | File/line | Value | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | request cookie `auth` | JWT string | browser | — | never expires |
| 2 | [verifyJWT](../../../apps/backend/src/services/auth/auth.middleware.ts#L39) | `User` object | middleware | verify+decode | whole-user payload |
| 3 | [getOrgsByUserId + filter](../../../apps/backend/src/services/auth/auth.middleware.ts#L79-L83) | `Organization[]` | middleware | DB read, filter disabled | empty→crash→403 |
| 4 | [pick by showorg](../../../apps/backend/src/services/auth/auth.middleware.ts#L82-L83) | one `Organization` | middleware | header select or `[0]` | header is user-supplied |
| 5 | [req.org set](../../../apps/backend/src/services/auth/auth.middleware.ts#L95-L99) | `req.org` | middleware | attach | ts-ignored |
| 6 | [@GetOrgFromRequest](../../../libraries/nestjs-libraries/src/user/org.from.request.ts) | `org` param | controller | read req.org | untyped |
| 7 | `this._x.method(org.id, …)` | `orgId: string` | service→repo | pass down | must reach every where-clause |
| 8 | Prisma `where: {organizationId: orgId}` | SQL predicate | repo | — | **the one place a miss = IDOR** |

## Trace 3: OAuth `state`, connect leg 1 → leg 2 (auth/redis trace)

| Step | File/line | Value | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | [generateAuthUrl](../../../apps/backend/src/api/routes/integrations.controller.ts#L226-L227) | `{state, codeVerifier, url}` | provider | PKCE gen | — |
| 2 | [ioRedis.set login/organization/…](../../../apps/backend/src/api/routes/integrations.controller.ts#L241-L248) | Redis keys, TTL 3600 | API | persist context | Redis flush = orphan |
| 3 | browser → provider → callback | `?code&state` | provider | user consent | — |
| 4 | [ioRedis.get login/organization](../../../apps/backend/src/api/routes/no.auth.integrations.controller.ts#L60-L72) | codeVerifier, orgId | API (no auth) | redeem | state = the whole credential |
| 5 | [ioRedis.del](../../../apps/backend/src/api/routes/no.auth.integrations.controller.ts#L74-L94) | — | API | single-use delete | race on double callback |
| 6 | [authenticate(code, codeVerifier)](../../../apps/backend/src/api/routes/no.auth.integrations.controller.ts#L109-L116) | `AuthTokenDetails` | provider | code→tokens | — |
| 7 | [createOrUpdateIntegration](../../../apps/backend/src/api/routes/no.auth.integrations.controller.ts#L212-L240) | Integration row | service | upsert; tokens **plaintext** | R1 |

## Trace 4: publish-time state transition of one Post (persistence/state trace)

| Step | File/line | state | Owner | Trigger |
| --- | --- | --- | --- | --- |
| 1 | [createOrUpdatePost](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L530-L535) | QUEUE (or DRAFT) | repo | user schedules |
| 2 | [workflow re-check](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L90-L93) | still QUEUE required | workflow | fire time |
| 3a | [updatePost success](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L367-L378) | PUBLISHED (+releaseId/URL) | repo | provider ok |
| 3b | [changeState error](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L393-L428) | ERROR (+Errors row) | repo | provider fail |
| 4 | [changePostStatus](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L818-L841) | back to QUEUE/DRAFT | service | user reschedules |

Risk to name: nothing enforces legal transitions; step 3a-then-error (published, then updatePost throws) yields the illegal PUBLISHED→ERROR you traced in [debugging scenario 2](../05-quality-engineering/03-systematic-debugging.md).

## Trace 5: a plug from settings to child action (async trace)

| Step | File/line | Value | Owner | Transformation |
| --- | --- | --- | --- | --- |
| 1 | post `settings` JSON | `{plug--name--field: val}` | DB | user config in composer |
| 2 | [checkInternalPlug parse](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L577-L617) | `[{type:'internal-plug', delay, …}]` | service | key-splitting into structured triggers |
| 3 | [internalPlugs activity](../../../apps/orchestrator/src/activities/post.activity.ts#L299-L307) | same list | activity | pass-through |
| 4 | [workflow sort by delay](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L311-L315) | ordered list | workflow | merge internal+global+repeat, sort |
| 5 | [sleep + processInternalPlug](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L318-L359) | — | workflow | durable delay then execute |

## Trace 6: a media id → published attachment URL (persistence/error trace)

| Step | File/line | Value | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | post `image` column | `[{id}]` (ids only) or `[{path}]` | DB | — | JSON string |
| 2 | [updateMedia getMediaById](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L346-L351) | `Media` rows | service | id→row | — |
| 3 | [url/path assembly](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L356-L371) | `{url, path}` absolute | service | env-var prefixing | 3 env vars must agree |
| 4 | [optional PNG→JPEG](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L373-L423) | new file + url | service | sharp transcode + re-upload | side effect on read |
| 5 | [catch → return imagesList](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L434-L436) | original input on any error | service | swallow | defers failure to provider |
| 6 | [postSocial media param](../../../apps/orchestrator/src/activities/post.activity.ts#L238-L242) | provider media list | activity | → provider upload | — |

Self-grade for the module — Strong: for each trace you can name the single step where the data's *guarantees* are weakest (validation gap, missing scope, swallowed error) and what test would pin it.

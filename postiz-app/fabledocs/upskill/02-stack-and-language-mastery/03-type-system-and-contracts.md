# Type System & Contracts

The question this file teaches you to ask everywhere: **is this type checked at compile time, at runtime, or not at all?**

## Where types are real

- **Prisma**: schema → generated client; `where`/`select` shapes are fully typed. The repository generic [PrismaRepository<'post'>](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L26-L34) keys models by name.
- **DTOs with class-validator**: compile-time shape + runtime enforcement in one class — [CreatePostDto](../../../libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts#L93-L125). The crown jewel: per-provider settings validated via a runtime **discriminator** on `settings.__type` ([L71-L80](../../../libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts#L71-L80)) — a runtime discriminated union. Content is sanitized *during transform* ([L35](../../../libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts#L31-L36)) — validation and normalization fused.
- **Typed failures across process boundaries**: `RefreshToken`/`BadBody` extend `ApplicationFailure` so the *type name* survives Temporal serialization and drives workflow control flow ([social.abstract.ts L5-L27](../../../libraries/nestjs-libraries/src/integrations/social.abstract.ts#L5-L27) → [workflow L221-L226](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L219-L237)). A contract enforced by string `type` tag — runtime, deliberate, documented by usage.

## Where types are theater

- `@Body() rawBody: any` on the flagship endpoint ([posts.controller.ts L174](../../../apps/backend/src/api/routes/posts.controller.ts#L170-L179)) — the global ValidationPipe can't see through `any`, so validation was rebuilt manually downstream ([mapTypeToPost L280-L291](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L243-L292)). Lesson: an `any` at a boundary doesn't remove the need for types; it relocates and duplicates it.
- `req.user` / `req.org` — Express request augmentation never declared, so every access is `@ts-expect-error`/`@ts-ignore` ([auth.middleware.ts L61-L99](../../../apps/backend/src/services/auth/auth.middleware.ts#L50-L104), [permissions.guard.ts L43-L50](../../../apps/backend/src/services/auth/permissions/permissions.guard.ts#L43-L51)). Ticket J10 fixes this properly.
- `PostWithConditionals` ([posts.service.ts L53-L56](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L53-L56)) — hand-widening a Prisma result because includes vary by call. The honest tool is `Prisma.PostGetPayload<{include: {...}}>` per shape.
- JSON-string columns: `settings`, `image` are `string` to both TS and Postgres; their real shape exists only in the DTO at write time. Every `JSON.parse(post.settings || '{}')` is an unchecked cast ([post.activity.ts L237](../../../apps/orchestrator/src/activities/post.activity.ts#L226-L246)).

## Contract inventory (what promises what, to whom)

| Contract | Producer → Consumer | Enforced by |
| --- | --- | --- |
| REST DTOs | frontend → backend | class-validator at runtime |
| Minified posts codec | backend → browser | shared module ([posts.list.minify.ts](../../../libraries/helpers/src/utils/posts.list.minify.ts)) — drift-proof by construction, but untested |
| Activity signatures | workflow → activity | `proxyActivities<PostActivity>` — **compile-time only**; at runtime it's name-based dispatch; renaming a method breaks running workflows silently |
| `settings.__type` | service → DTO → provider | runtime discriminator |
| Provider interface | manager → 30 providers | TS interface + optional members probed at runtime (`!!getIntegration.comment`) |
| Webhook payload | activity → customer servers | nothing — an implicit public contract ([getPostByForWebhookId shape](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L844-L868)) |
| SDK / public API | backend → external devs | swagger decorators; version prefix `/public/v1` |

Senior habit: for each contract, ask "what happens when producer and consumer deploy at different times?" Temporal workflow contracts are the sharpest instance (five kept versions), the webhook payload the most neglected.

Interview angle:
1. "How do you validate input in a Node API?" → DTO + pipe, plus the discriminator trick; then the `any` cautionary tale.
2. "TS types at runtime?" → erased; you need class-validator/zod/decorept metadata; cite the discriminator.
3. "How do you type Express middleware additions?" → module augmentation (and what happens when you don't — 20 ts-ignores).
4. "What is a contract test and where would you add one here?" → the minify/expand codec round-trip; the webhook payload snapshot.

Drill: pick `Integration.additionalSettings` (a `String? @default("[]")` — [schema L336](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L314-L356)). Trace every read/write and write down its implied schema. You'll find the type lives in scattered code expectations — then sketch the zod schema + parse boundary that would make it real.
Self-grade — Strong: your sketch has exactly one parse site and exported inferred types.

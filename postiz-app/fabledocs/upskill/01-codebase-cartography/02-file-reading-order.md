# File Reading Order

28 files. Per file: why it matters, what to look for, what to skip. Paths are junior (J: 1–12), mid (M: 1–22), senior (S: all, plus the "senior extras").

| # | File | Why / look for | Ignore |
| --- | --- | --- | --- |
| 1 | [CLAUDE.md](../../../CLAUDE.md) | House rules; note the drift (Vite claim) — calibrates your trust in docs | — |
| 2 | [package.json](../../../package.json#L13-L41) | Scripts; `postinstall` runs prisma generate; single dep universe | the 200-line dep list (skim) |
| 3 | [docker-compose.dev.yaml](../../../docker-compose.dev.yaml) | What infra must exist; Temporal has its own PG+ES | pgadmin/redisinsight blocks |
| 4 | [apps/backend/src/main.ts](../../../apps/backend/src/main.ts) | Bootstrap order, CORS allowlist, rawBody, 50MB `/posts` limit, config checker | — |
| 5 | [apps/backend/src/app.module.ts](../../../apps/backend/src/app.module.ts#L35-L56) | Global guards: Redis-backed throttle (90/h) + PoliciesGuard | Sentry/module noise |
| 6 | [api.module.ts](../../../apps/backend/src/api/api.module.ts#L48-L110) | **The auth boundary**: `authenticatedController` array; who's outside (Stripe, Public, NoAuthIntegrations, OAuth) | provider list |
| 7 | [auth.middleware.ts](../../../apps/backend/src/services/auth/auth.middleware.ts) | JWT→user, showorg→org, impersonation; find the empty-org crash | — |
| 8 | [schema.prisma](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma) | Read `Organization`, `User`, `UserOrganization`, `Integration`, `Post` fully; skim the rest; note JSON-string columns and soft deletes | mastra_* tables |
| 9 | [posts.controller.ts](../../../apps/backend/src/api/routes/posts.controller.ts) | Route ordering (`/tags` before `/:id`), the `any` body, streaming generator endpoint | — |
| 10 | [posts.service.ts](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts) | The biggest brain: createPost, startWorkflow, mapTypeToPost, updateMedia | popular-posts block |
| 11 | [posts.repository.ts](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts) | createOrUpdatePost (thread upsert), getPosts (interval expansion), org-scoping habits — and lapses | — |
| 12 | [post.workflow.v1.0.5.ts](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts) | The durable core; retry loops; typed failures; plugs; child workflow | — (read every line) |
| — | **Junior path ends; you can trace Flow 1** | | |
| 13 | [post.activity.ts](../../../apps/orchestrator/src/activities/post.activity.ts) | Activities = all I/O; slimPost; subscription gates; webhook fan-out | — |
| 14 | [social.abstract.ts](../../../libraries/nestjs-libraries/src/integrations/social.abstract.ts) | Shared fetch/retry, RefreshToken/BadBody taxonomy | — |
| 15 | [integration.manager.ts](../../../libraries/nestjs-libraries/src/integrations/integration.manager.ts) | Registry; metadata-scanning for plugs/tools | — |
| 16 | [x.provider.ts](../../../libraries/nestjs-libraries/src/integrations/social/x.provider.ts) | One full provider: handleErrors, post/comment/analytics, @Plug | media upload minutiae |
| 17 | [integrations.controller.ts](../../../apps/backend/src/api/routes/integrations.controller.ts) | OAuth leg 1; channel CRUD; the dynamic `functionIntegration` dispatch | telegram/moltbook extras |
| 18 | [no.auth.integrations.controller.ts](../../../apps/backend/src/api/routes/no.auth.integrations.controller.ts) | OAuth leg 2; state redemption; trust chain | extension refresh |
| 19 | [permissions.service.ts](../../../apps/backend/src/services/auth/permissions/permissions.service.ts) + [guard](../../../apps/backend/src/services/auth/permissions/permissions.guard.ts) | CASL + plan limits; path bypasses | — |
| 20 | [create.post.dto.ts](../../../libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts) | Discriminated settings validation; sanitizer transform | — |
| 21 | [custom.fetch.func.ts](../../../libraries/helpers/src/utils/custom.fetch.func.ts) + [custom.fetch.tsx](../../../libraries/helpers/src/utils/custom.fetch.tsx) | The frontend's single HTTP door; never-resolving promise trick | — |
| 22 | [new-launch/store.ts](../../../apps/frontend/src/components/new-launch/store.ts) + [add.edit.modal.tsx](../../../apps/frontend/src/components/new-launch/add.edit.modal.tsx) | Composer state machine; staged mount gates | — |
| — | **Mid path ends; you can trace Flows 1–4 and critique them** | | |
| 23 | [missing.post.workflow.ts](../../../apps/orchestrator/src/workflows/missing.post.workflow.ts) + [infinite.workflow.register.ts](../../../libraries/nestjs-libraries/src/temporal/infinite.workflow.register.ts) | The reconciliation loop and its env-gate | — |
| 24 | [stripe.controller.ts](../../../apps/backend/src/api/routes/stripe.controller.ts) + subscription service | Inbound webhook verification done right | — |
| 25 | [media.controller.ts](../../../apps/backend/src/api/routes/media.controller.ts) + [upload/](../../../libraries/nestjs-libraries/src/upload/upload.factory.ts) | Storage strategy; magic-byte validation | R2 proxy details |
| 26 | [helpers/auth.service.ts](../../../libraries/helpers/src/auth/auth.service.ts) | JWT/bcrypt/legacy-AES; no-expiry finding | EVP details |
| 27 | [public.integrations.controller.ts](../../../apps/backend/src/public-api/routes/v1/public.integrations.controller.ts) | The public contract; SSRF-safe fetch, MIME allowlist | — |
| 28 | [calendar.context.tsx](../../../apps/frontend/src/components/launches/calendar.context.tsx) + [calendar.tsx](../../../apps/frontend/src/components/launches/calendar.tsx) exports | SWR-fed context; Day/Week/Month views | drag-drop internals |

Senior extras: [refresh.integration.service.ts](../../../libraries/nestjs-libraries/src/integrations/refresh.integration.service.ts) (cross-process token race), [temporal.module.ts](../../../libraries/nestjs-libraries/src/temporal/temporal.module.ts), [throttler.provider.ts](../../../libraries/nestjs-libraries/src/throttler/throttler.provider.ts) (proxy-aware rate limiting), [exception.filter.ts](../../../libraries/nestjs-libraries/src/services/exception.filter.ts), the older workflow versions (diff v1.0.4 → v1.0.5 to see what changed and why versioning matters).

Pause-and-predict habit: before opening each file, write one sentence guessing what's inside from its name and imports; score yourself. Calibration is the skill.

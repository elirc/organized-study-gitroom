# System Map

## Shape: pnpm monorepo, one dependency universe

Single root [package.json](../../../package.json) holds *all* dependencies (unusual — most monorepos give each app its own); apps declare only scripts. Workspace globs in [pnpm-workspace.yaml](../../../pnpm-workspace.yaml). Node `>=22.12 <23`, pnpm 10.6.1 pinned.

```
postiz-app/
├── apps/
│   ├── backend/        NestJS HTTP API              :3000
│   ├── orchestrator/   NestJS + Temporal worker      :3002 (health only)
│   ├── frontend/       Next.js 16 (App Router)       :4200
│   ├── extension/      Chrome extension (Vite)
│   ├── sdk/            published client SDK
│   └── commands/       CLI (nestjs-command)
├── libraries/
│   ├── nestjs-libraries/   ~all business logic: database/, integrations/, temporal/,
│   │                       upload/, redis/, openai/, agent/, chat/, dtos/, emails/…
│   ├── helpers/            isomorphic utils: auth (JWT/bcrypt), custom fetch, decorators, swagger
│   └── react-shared-libraries/  form components, translation, toaster, sentry (client)
├── docker-compose.dev.yaml    Postgres, Redis, Temporal(+ES+PG), pgadmin, redisinsight
└── var/docker/                production image scripts
```

## Runtime topology

```
Browser ──(cookies: auth/showorg/impersonate)──▶ Next.js frontend :4200
   │                                                │ SSR + /api/uploads proxy
   └──────────── fetch (credentials) ──────────────▶ NestJS backend :3000
                                                     │  Prisma ▶ Postgres
                                                     │  ioredis ▶ Redis (OAuth state, throttle, caches)
                                                     │  Temporal client ▶ Temporal :7233
                                                     ▼
                                     Temporal server ⇄ orchestrator worker
                                                     │  activities: Prisma, provider APIs, webhooks, emails
                                                     ▼
                                     30+ social provider APIs · Stripe · R2/local storage · SMTP/Resend
```

## Ownership map

| Concern | Home | Notes |
| --- | --- | --- |
| HTTP routing/authz | [apps/backend/src/api/routes/](../../../apps/backend/src/api/routes) (27 controllers) | Thin by rule ([CLAUDE.md L31-L37](../../../CLAUDE.md#L31-L37)) |
| Auth boundary | [api.module.ts](../../../apps/backend/src/api/api.module.ts#L48-L110) + [auth.middleware.ts](../../../apps/backend/src/services/auth/auth.middleware.ts) | One array decides what's protected |
| Business logic | [libraries/nestjs-libraries/src/database/prisma/*/](../../../libraries/nestjs-libraries/src/database/prisma) service+repository pairs | Shared by backend, orchestrator, commands |
| Provider integrations | [libraries/nestjs-libraries/src/integrations/](../../../libraries/nestjs-libraries/src/integrations) | Registry + 30 providers + shared abstract |
| Background jobs | [apps/orchestrator/src/workflows|activities](../../../apps/orchestrator/src) | Temporal; task queues per provider |
| Schema | [schema.prisma](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma) (970 lines, ~40 models) | `db push`, no migrations dir |
| UI routes | [apps/frontend/src/app/(app)/(site)/](../../../apps/frontend/src/app) | launches (calendar), analytics, settings, media, billing… |
| UI components | [apps/frontend/src/components/](../../../apps/frontend/src/components) | `new-launch` = composer; `launches` = calendar |
| Public API | [apps/backend/src/public-api/](../../../apps/backend/src/public-api/routes/v1/public.integrations.controller.ts) | API-key auth, v1 |
| AI/agents | `libraries/nestjs-libraries/src/{agent,chat,openai}` + mastra_* tables | Not covered deeply here |

## Public interfaces vs private internals

**Public (contracts you must not break casually)**: `/public/v1/*` (API-key consumers), outgoing webhook payload shape ([getPostByForWebhookId](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L844-L868)), the SDK in `apps/sdk`, Stripe webhook expectations, provider OAuth redirect URLs, and — internally but critically — **Temporal workflow names/signatures** (running workflows pin old code; that's why [post-workflows/](../../../apps/orchestrator/src/workflows/post-workflows) keeps v1.0.1–v1.0.5).

**Private (refactor freely)**: everything in `libraries/` not exported to the SDK, controller internals, frontend components.

## What kind of project is this?

Commercial open-source (AGPL + paid cloud): code shows the dual nature everywhere — every billing check is conditional on Stripe env vars ([permissions.service.ts L50-L63](../../../apps/backend/src/services/auth/permissions/permissions.service.ts#L50-L63), [post.activity.ts L115](../../../apps/orchestrator/src/activities/post.activity.ts#L113-L124)). When you read a weird guard, ask "cloud or self-host?" first.

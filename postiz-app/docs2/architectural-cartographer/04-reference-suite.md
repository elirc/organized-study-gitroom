# Reference Suite

## Doc 1: Junior Onboarding Guide

Start with environment and scripts. Root package requires Node `>=22.12.0 <23.0.0`, pnpm `10.6.1`, and provides `dev`, `dev-backend`, `dev:docker`, Prisma, build, and test scripts (`package.json:9-40`). Copy `.env.example` to `.env` and fill required database, Redis, JWT, frontend, and backend URLs (`.env.example:3-12`). Run Docker dev services for Postgres/Redis/Temporal (`docker-compose.dev.yaml:5-30`, `docker-compose.dev.yaml:57-137`). Run Prisma generation/push through root scripts (`package.json:33-39`). Then run `pnpm run dev-backend` for frontend/backend only or `pnpm run dev` for all main apps (`package.json:14-15`).

Read order: route file (`apps/frontend/src/app/(app)/(site)/launches/page.tsx:1-11`), layout (`apps/frontend/src/components/new-layout/layout.component.tsx:52-149`), calendar context (`apps/frontend/src/components/launches/calendar.context.tsx:134-351`), composer modal (`apps/frontend/src/components/new-launch/manage.modal.tsx:202-447`), post controller (`apps/backend/src/api/routes/posts.controller.ts:111-229`), post service (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:243-292`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:750-808`), and post repository (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:488-630`).

## Doc 2: Mid-Level Architecture Guide

The system has three active lanes. The UI lane is Next App Router plus client components and SWR (`apps/frontend/src/app/(app)/layout.tsx:34-121`, `apps/frontend/src/components/launches/calendar.context.tsx:201-231`). The API lane is Nest controllers, middleware, guards, services, and DTO validation (`apps/backend/src/api/api.module.ts:48-109`, `apps/backend/src/services/auth/auth.middleware.ts:33-104`, `libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts:93-125`). The persistence/publishing lane is Prisma plus Temporal workflow start (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:393-440`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:488-630`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:791-798`).

Contracts to protect: `useFetch` request behavior (`libraries/helpers/src/utils/custom.fetch.func.ts:46-84`), `CreatePostDto` shape (`libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts:93-125`), `Post` schema (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:393-440`), and calendar response expansion/minification (`apps/frontend/src/components/launches/calendar.context.tsx:183-198`).

## Doc 3: Senior Ownership Guide

Technical debt register:

- `rawBody: any` in post creation weakens API clarity (`apps/backend/src/api/routes/posts.controller.ts:170-179`).
- Payload logging can leak draft content (`apps/backend/src/api/routes/posts.controller.ts:176-178`).
- Backend build strictness is relaxed (`apps/backend/tsconfig.build.json:16-20`).
- Large UI files mix interaction, rendering, and business decisions (`apps/frontend/src/components/launches/calendar.tsx:1-1318`, `apps/frontend/src/components/new-launch/manage.modal.tsx:1-712`).
- Tests are configured but no standard test files were found by inspection (`package.json:40`, `jest.config.ts:1-5`).

Upgrade path: add tests around post mapping/creation, extract composer payload builder, type API contracts, remove sensitive logs, and cap recurrence expansion.

## Doc 4: Code Review Guide

Review in this order:

1. Route/module registration. New authenticated backend routes must be in authenticated controller wiring (`apps/backend/src/api/api.module.ts:48-109`).
2. Auth/org isolation. Confirm `org.id` flows into service/repository calls (`apps/backend/src/api/routes/posts.controller.ts:111-179`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:503-507`).
3. DTO/runtime validation. Confirm user input passes through DTO or explicit validation (`libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts:93-125`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:280-291`).
4. UI fetch behavior. Confirm frontend uses `useFetch` for backend calls (`libraries/helpers/src/utils/custom.fetch.tsx:44-46`, `libraries/helpers/src/utils/custom.fetch.func.ts:46-84`).
5. Cache/update behavior. Confirm SWR mutate/reload is called after writes (`apps/frontend/src/components/new-launch/manage.modal.tsx:427-443`, `apps/frontend/src/components/launches/calendar.context.tsx:316-320`).
6. Persistence side effects. Confirm update paths handle existing groups and soft delete behavior (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:602-630`).

## Doc 5: Debugging Guide

For a missing post in the calendar, trace: `/launches` route (`apps/frontend/src/app/(app)/(site)/launches/page.tsx:1-11`), `CalendarWeekProvider` query params (`apps/frontend/src/components/launches/calendar.context.tsx:165-185`), controller `GET /posts` (`apps/backend/src/api/routes/posts.controller.ts:111-117`), repository date/customer filters (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:129-172`), and recurrence expansion (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:199-219`).

For a failed schedule, trace: composer validation (`apps/frontend/src/components/new-launch/manage.modal.tsx:248-334`), payload creation (`apps/frontend/src/components/new-launch/manage.modal.tsx:372-401`), `POST /posts` (`apps/frontend/src/components/new-launch/manage.modal.tsx:419-425`), controller policy and mapping (`apps/backend/src/api/routes/posts.controller.ts:170-179`), provider discriminator injection (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:256-275`), and repository upsert (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:488-630`).

## Doc 6: Change Playbook

To add a feature end-to-end:

1. Find the route or component that owns the screen. For scheduling, start at `/launches` (`apps/frontend/src/app/(app)/(site)/launches/page.tsx:1-11`).
2. Identify read state and write state. Calendar reads live in `CalendarWeekProvider`, composer writes in `useLaunchStore` (`apps/frontend/src/components/launches/calendar.context.tsx:134-351`, `apps/frontend/src/components/new-launch/store.ts:160-654`).
3. Add UI and serialization near the existing payload builder (`apps/frontend/src/components/new-launch/manage.modal.tsx:372-401`).
4. Add/adjust DTO validation (`libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts:93-125`).
5. Add controller/service/repository changes (`apps/backend/src/api/routes/posts.controller.ts:170-179`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:750-808`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:488-630`).
6. Update schema if persistence changes (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:393-440`).
7. Verify SWR reload/mutate after writes (`apps/frontend/src/components/launches/calendar.context.tsx:316-320`).

## Doc 7: Interview Walkthrough

"Postiz is a monorepo social scheduling app. The root package runs multiple apps with pnpm: frontend, backend, orchestrator, and extension (`package.json:14-31`). The frontend is a Next App Router application; `/launches` is the scheduling calendar route (`apps/frontend/src/app/(app)/(site)/launches/page.tsx:1-11`). The authenticated shell loads `/user/self` and provides user context, Copilot, billing, and navigation (`apps/frontend/src/components/new-layout/layout.component.tsx:52-149`). The calendar uses SWR and a fetch wrapper to load posts and integrations (`apps/frontend/src/components/launches/calendar.context.tsx:174-231`, `libraries/helpers/src/utils/custom.fetch.func.ts:46-84`). The backend is Nest; it applies auth middleware to authenticated controllers (`apps/backend/src/api/api.module.ts:48-109`). Post creation goes through `PostsController`, `PostsService`, and `PostsRepository`, then Prisma persists rows and Temporal publishes queued work (`apps/backend/src/api/routes/posts.controller.ts:170-179`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:750-808`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:488-630`)."

# Mission 25: Write the Docs That Don't Exist

## Doc 1: Junior Onboarding Checklist

- Install Node `>=22.12.0 <23.0.0` and pnpm `10.6.1` (`package.json:9-12`).
- Copy `.env.example` to `.env` and set required values (`.env.example:3-12`).
- Start dependencies with `pnpm run dev:docker` (`package.json:31`, `docker-compose.dev.yaml:5-30`, `docker-compose.dev.yaml:57-137`).
- Run `pnpm install`, then confirm Prisma generation via postinstall (`package.json:33-40`).
- Run `pnpm run prisma-db-push` before app testing (`package.json:33-36`).
- Start `pnpm run dev-backend` for frontend/backend work (`package.json:15`).
- First PR checklist: find route, state owner, API endpoint, DTO, service, repository, schema, and cache refresh.

## Doc 2: Architecture Guide For New Engineers

Navigate by request flow. For UI, start at route files under `apps/frontend/src/app`, then jump to feature components under `apps/frontend/src/components` (`apps/frontend/src/app/(app)/(site)/launches/page.tsx:1-11`). For API, start at controller routes in `apps/backend/src/api/routes`, then move to services/repositories in `libraries/nestjs-libraries/src/database/prisma` (`apps/backend/src/api/routes/posts.controller.ts:111-229`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:243-292`). For persistence, read `schema.prisma` models and indexes (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:393-440`).

## Doc 3: Code Review Checklist

- Does the change respect auth/org boundaries (`apps/backend/src/services/auth/auth.middleware.ts:78-99`)?
- Does the frontend use `useFetch` for backend calls (`libraries/helpers/src/utils/custom.fetch.tsx:44-46`)?
- Does the DTO validate any new client input (`libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts:93-125`)?
- Does the repository write and read the same fields (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:173-196`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:500-560`)?
- Does the UI mutate/reload after writes (`apps/frontend/src/components/new-launch/manage.modal.tsx:427-443`)?

## Doc 4: Debugging Playbook

For login/auth failures, inspect `customFetch` credentials and headers (`libraries/helpers/src/utils/custom.fetch.func.ts:46-74`), `LayoutContext` response handling (`apps/frontend/src/components/layout/layout.context.tsx:27-121`), `ApiModule` middleware application (`apps/backend/src/api/api.module.ts:48-109`), and `AuthMiddleware` token/org logic (`apps/backend/src/services/auth/auth.middleware.ts:33-104`).

For missing calendar data, inspect `CalendarWeekProvider` params and SWR keys (`apps/frontend/src/components/launches/calendar.context.tsx:165-231`), controller routes (`apps/backend/src/api/routes/posts.controller.ts:111-138`), and repository filters (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:129-260`).

## Doc 5: Change Playbook

Branch, inspect the existing feature path, make the smallest coherent change, and verify the route/API/read-model loop. For scheduling features, touch only the layers needed: composer UI (`apps/frontend/src/components/new-launch/manage.modal.tsx:372-425`), DTO (`libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts:93-125`), controller (`apps/backend/src/api/routes/posts.controller.ts:170-179`), service (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:243-292`), repository (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:488-630`), schema (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:393-440`), and read select (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:173-196`).

## Doc 6: Senior Ownership Notes

Monitor scheduling latency, failed workflows, post creation errors, auth failures, and calendar query volume. Improve `rawBody: any`, payload logging, large UI files, missing tests, and recurrence expansion (`apps/backend/src/api/routes/posts.controller.ts:170-179`, `apps/frontend/src/components/new-launch/manage.modal.tsx:1-712`, `apps/frontend/src/components/launches/calendar.tsx:1-1318`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:199-219`). Leave alone the core separation of route, context, controller, service, repository, and schema until a change has evidence.

## Doc 7: Interview Walkthrough

Practice answer: "I worked through a Postiz social scheduling flow. The Next `/launches` route renders the scheduling feature (`apps/frontend/src/app/(app)/(site)/launches/page.tsx:1-11`). The authenticated layout loads user state and wraps shared tools (`apps/frontend/src/components/new-layout/layout.component.tsx:52-149`). Calendar data loads through SWR and a custom fetch wrapper (`apps/frontend/src/components/launches/calendar.context.tsx:174-231`, `libraries/helpers/src/utils/custom.fetch.func.ts:46-84`). Creating a post serializes modal state into `POST /posts` (`apps/frontend/src/components/new-launch/manage.modal.tsx:372-425`). Nest middleware attaches org context, the controller delegates, the service validates/provider-normalizes, and the repository persists Prisma rows (`apps/backend/src/services/auth/auth.middleware.ts:33-104`, `apps/backend/src/api/routes/posts.controller.ts:170-179`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:243-292`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:488-630`)."

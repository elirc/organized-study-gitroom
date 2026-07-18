# Mission Learning Path Journal

## Mission Design Rationale

The missions are ordered from "find the app heartbeat" to "own the system under pressure." The first tier focuses on concrete navigation: scripts, folders, TypeScript contracts, small components, routes, props, data fetches, and navigation. The second tier follows real data and state: SWR, custom hooks, side effects, API contracts, middleware, full-stack scheduling, diff review, composition, and type inference. The third tier asks senior questions: why this architecture exists, where bugs hide, where performance and security risks live, what test is missing, and what history a codebase implies.

## Code Paths Chosen

The primary path is scheduling a post because it touches the actual product core: `/launches` route, calendar context, new-post modal, composer store, `POST /posts`, auth middleware, service mapping, repository upsert, Prisma post schema, and Temporal workflow start (`apps/frontend/src/app/(app)/(site)/launches/page.tsx:1-11`, `apps/frontend/src/components/launches/calendar.context.tsx:134-351`, `apps/frontend/src/components/new-launch/manage.modal.tsx:202-447`, `apps/backend/src/api/routes/posts.controller.ts:170-179`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:243-292`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:750-808`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:488-630`).

Secondary paths were chosen to build surrounding skill: `useFetch` teaches cross-cutting request behavior (`libraries/helpers/src/utils/custom.fetch.func.ts:46-84`), `AuthMiddleware` teaches request context (`apps/backend/src/services/auth/auth.middleware.ts:33-104`), `CreationMethodBadge` teaches small typed UI (`apps/frontend/src/components/launches/creation.method.badge.tsx:4-50`), and Prisma schema teaches persistence vocabulary (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:11-50`, `libraries/nestjs-libraries/src/database/prisma/schema.prisma:314-440`, `libraries/nestjs-libraries/src/database/prisma/schema.prisma:903-956`).

## Skill Developed By Tier

- Junior: locate entry points, identify ownership, explain a small file, and follow one HTTP call.
- Mid-Level: map contracts, compare UI state to API shape, reason about side effects, and review diffs for missing layers.
- Senior: evaluate tradeoffs, anticipate failure, design tests, and explain evolution from code evidence.

## What A Senior Engineer Would Do Differently

A senior would not memorize this repo. They would build a small set of invariants: every authenticated backend call needs org context; every scheduled post belongs to an integration and organization; every calendar write must trigger a read refresh; every provider-specific post must carry a discriminator; every schema change needs DTO, repository, read model, and UI review. Those invariants are visible in the cited files (`apps/backend/src/services/auth/auth.middleware.ts:78-99`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:503-507`, `apps/frontend/src/components/launches/calendar.context.tsx:316-320`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:268-275`).

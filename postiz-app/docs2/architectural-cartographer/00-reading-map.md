# Reading Map

## Mental Model

Postiz is a scheduling command center: the frontend calendar lets a user choose channels, compose social posts, and schedule them; the backend authenticates the organization context, validates and normalizes post payloads, persists posts through Prisma, and starts Temporal workflows that eventually publish through provider integrations. That model is grounded in the product description (`README.md:93-100`), stack list (`README.md:102-109`), frontend Launches route (`apps/frontend/src/app/(app)/(site)/launches/page.tsx:1-11`), authenticated API wiring (`apps/backend/src/api/api.module.ts:48-109`), post controller (`apps/backend/src/api/routes/posts.controller.ts:170-179`), post service (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:750-808`), and post schema (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:393-440`).

## Top 10 Files To Read In Order

1. `package.json`
   - Why now: establishes pnpm, Node version, dev/build/test scripts (`package.json:9-40`).
   - Understand first: this is a monorepo, not one app.
   - Explain after reading: which command starts all dev apps and which command pushes Prisma schema.

2. `apps/frontend/src/app/(app)/layout.tsx`
   - Why now: it is the root Next layout for the app shell (`apps/frontend/src/app/(app)/layout.tsx:34-121`).
   - Understand first: server components can read cookies and env variables.
   - Explain after reading: how `VariableContextComponent` and `LayoutContext` make environment and fetch behavior available.

3. `apps/frontend/src/components/layout/layout.context.tsx`
   - Why now: it wraps client requests with backend URL and response handling (`apps/frontend/src/components/layout/layout.context.tsx:24-129`).
   - Understand first: `useFetch` is context-driven.
   - Explain after reading: what response headers can update auth, org, impersonation, redirects, and billing behavior.

4. `apps/frontend/src/components/new-layout/layout.component.tsx`
   - Why now: it loads `/user/self` and builds the authenticated UI frame (`apps/frontend/src/components/new-layout/layout.component.tsx:52-83`, `apps/frontend/src/components/new-layout/layout.component.tsx:91-149`).
   - Understand first: SWR owns many read-model fetches.
   - Explain after reading: why children do not render until the user is loaded.

5. `apps/frontend/src/app/(app)/(site)/launches/page.tsx`
   - Why now: this is the route for the key scheduling feature (`apps/frontend/src/app/(app)/(site)/launches/page.tsx:1-11`).
   - Understand first: App Router pages can be tiny delegates.
   - Explain after reading: what component owns the feature.

6. `apps/frontend/src/components/launches/calendar.context.tsx`
   - Why now: it is the calendar read model and action context (`apps/frontend/src/components/launches/calendar.context.tsx:29-81`, `apps/frontend/src/components/launches/calendar.context.tsx:134-351`).
   - Understand first: calendar view and list view use different API endpoints.
   - Explain after reading: how filters become query params and SWR keys.

7. `apps/frontend/src/components/new-launch/manage.modal.tsx`
   - Why now: it serializes the post creation/update command (`apps/frontend/src/components/new-launch/manage.modal.tsx:202-447`).
   - Understand first: selected integrations and content live in Zustand plus editor refs.
   - Explain after reading: what conditions block a schedule attempt before the HTTP request.

8. `apps/backend/src/main.ts`
   - Why now: it shows middleware, CORS, validation, body limits, filters, Swagger, and port startup (`apps/backend/src/main.ts:22-80`).
   - Understand first: Nest startup is centralized here.
   - Explain after reading: why `/posts` receives a larger JSON body limit.

9. `apps/backend/src/api/routes/posts.controller.ts`
   - Why now: it is the authenticated HTTP contract for scheduling (`apps/backend/src/api/routes/posts.controller.ts:111-179`, `apps/backend/src/api/routes/posts.controller.ts:205-229`).
   - Understand first: `@GetOrgFromRequest` and `@GetUserFromRequest` read fields set by middleware.
   - Explain after reading: which controller methods map to calendar read, create, delete, and reschedule.

10. `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts`
    - Why now: it contains the business layer behind post creation (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:243-292`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:750-808`).
    - Understand first: service logic injects provider-specific settings and delegates persistence.
    - Explain after reading: how shortlinks, link stripping, creation method, and Temporal workflow start connect.

## Three Most Important Data Flows

1. Calendar read flow: `LaunchesComponent` loads integrations (`apps/frontend/src/components/launches/launches.component.tsx:352-365`), `CalendarWeekProvider` fetches `/posts` or `/posts/list` (`apps/frontend/src/components/launches/calendar.context.tsx:174-231`), `PostsController` delegates to `getPostsMinified` or `getPostsList` (`apps/backend/src/api/routes/posts.controller.ts:111-138`), and the repository queries Prisma posts by org/date/customer (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:129-220`).

2. Create post flow: `NewPost` finds a slot and opens `AddEditModal` (`apps/frontend/src/components/launches/new.post.tsx:17-75`), `ManageModal` validates and builds the payload (`apps/frontend/src/components/new-launch/manage.modal.tsx:248-401`), sends `POST /posts` (`apps/frontend/src/components/new-launch/manage.modal.tsx:419-425`), `PostsController.createPost` maps and delegates (`apps/backend/src/api/routes/posts.controller.ts:170-179`), `PostsService.createPost` normalizes links and starts workflow (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:750-808`), and `PostsRepository.createOrUpdatePost` upserts rows (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:488-630`).

3. Authenticated request flow: `LayoutContext` wraps children in `FetchWrapperComponent` (`apps/frontend/src/components/layout/layout.context.tsx:125-129`), `customFetch` sends credentials and headers (`libraries/helpers/src/utils/custom.fetch.func.ts:46-74`), `ApiModule` applies `AuthMiddleware` to authenticated controllers (`apps/backend/src/api/api.module.ts:48-109`), and `AuthMiddleware` verifies JWT, loads organization, and writes `req.user`/`req.org` (`apps/backend/src/services/auth/auth.middleware.ts:33-104`).

## Pre-Reading Checklist

1. Can I name every app in the root dev script (`package.json:14`)?
2. Can I explain why frontend dev runs on port 4200 (`apps/frontend/package.json:7`)?
3. Can I explain why backend dev points Nest to `apps/backend/src/main` (`apps/backend/package.json:6`)?
4. Can I find the route file for `/launches` (`apps/frontend/src/app/(app)/(site)/launches/page.tsx:1-11`)?
5. Can I name the component that owns the authenticated shell (`apps/frontend/src/components/new-layout/layout.component.tsx:52-149`)?
6. Can I say where `useFetch` gets its backend URL (`apps/frontend/src/components/layout/layout.context.tsx:24-129`)?
7. Can I explain where `req.org` comes from (`apps/backend/src/services/auth/auth.middleware.ts:78-99`)?
8. Can I name the Prisma model that stores scheduled posts (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:393-440`)?
9. Can I explain what the `State` enum allows (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:903-908`)?
10. Can I point to the code that starts publishing workflow after create (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:791-798`)?

## Red Flags Checklist

1. New authenticated routes must be added to the right controller/module path, because `AuthMiddleware` only applies to `authenticatedController` in `ApiModule` (`apps/backend/src/api/api.module.ts:48-109`).
2. Large post payload changes should consider the special `/posts` 50mb JSON limit (`apps/backend/src/main.ts:58-60`).
3. DTO validation is service-side for `POST /posts`, because the controller accepts `rawBody: any` (`apps/backend/src/api/routes/posts.controller.ts:170-179`).
4. Client requests should generally use `useFetch`, because it applies base URL, credentials, headers, and cache defaults (`libraries/helpers/src/utils/custom.fetch.func.ts:46-84`).
5. Calendar filters must update both React state and URL state (`apps/frontend/src/components/launches/calendar.context.tsx:258-281`).
6. Post thread changes must preserve parent/child behavior (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:500-560`).
7. Update behavior soft-deletes the previous group, so edits can affect historical debugging (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:602-630`).
8. Type strictness differs between root/frontend and backend build settings (`tsconfig.base.json:22-27`, `apps/backend/tsconfig.build.json:16-20`).
9. Tests are configured but no test files were found by the searched patterns (`package.json:40`, `jest.config.ts:1-5`).
10. Logs in `createPost` may capture full social post payloads (`apps/backend/src/api/routes/posts.controller.ts:176-178`).

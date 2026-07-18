# Architectural Cartographer Journal

## First-Pass Mental Model

Postiz is a pnpm monorepo: the root package declares pnpm 10.6.1, Node `>=22.12.0 <23.0.0`, and workspace scripts that run frontend, backend, orchestrator, and extension apps together (`package.json:9-15`, `package.json:31-40`). The README describes the product as a social media scheduling tool and names the stack as pnpm workspaces, NextJS, NestJS, Prisma/PostgreSQL, Temporal, and Resend (`README.md:93-109`). The source tree confirms the split: Next app routes live under `apps/frontend/src/app`, React feature components under `apps/frontend/src/components`, Nest controllers under `apps/backend/src/api/routes`, and Prisma-backed services under `libraries/nestjs-libraries/src/database/prisma`.

## Inspection Discoveries

- Frontend entry is the Next App Router. The `(app)` root layout imports global styles, reads the language cookie, injects environment variables into `VariableContextComponent`, and wraps children in `LayoutContext` (`apps/frontend/src/app/(app)/layout.tsx:1-27`, `apps/frontend/src/app/(app)/layout.tsx:34-121`).
- The authenticated shell is a client layout that loads `/user/self`, provides user context, mounts CopilotKit, tooltips, toaster, billing checks, and the left/top navigation (`apps/frontend/src/components/new-layout/layout.component.tsx:52-83`, `apps/frontend/src/components/new-layout/layout.component.tsx:91-149`).
- The backend entry is Nest. `main.ts` initializes Sentry, CORS, validation pipes, JSON body limit for `/posts`, cookie parsing, compression, filters, Swagger, and listens on `PORT || 3000` (`apps/backend/src/main.ts:1-22`, `apps/backend/src/main.ts:23-80`).
- Backend module wiring is conventional Nest but broad: `AppModule` imports database, API, public API, agent, third-party, video, chat, Temporal, and throttling modules, then installs global throttling and policy guards (`apps/backend/src/app.module.ts:21-66`).
- Authenticated controllers are grouped in `ApiModule`, and `AuthMiddleware` is applied to that group (`apps/backend/src/api/api.module.ts:48-81`, `apps/backend/src/api/api.module.ts:106-109`).
- Database persistence uses Prisma models for `Organization`, `User`, `Integration`, `Post`, `Tags`, `Media`, and enums such as `State`, `Provider`, and `CreationMethod` (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:11-50`, `libraries/nestjs-libraries/src/database/prisma/schema.prisma:79-123`, `libraries/nestjs-libraries/src/database/prisma/schema.prisma:314-356`, `libraries/nestjs-libraries/src/database/prisma/schema.prisma:393-440`, `libraries/nestjs-libraries/src/database/prisma/schema.prisma:903-956`).

## Why These Files Became Teaching Anchors

- `apps/frontend/src/app/(app)/(site)/launches/page.tsx` is a small route page that proves how route files delegate to feature components (`apps/frontend/src/app/(app)/(site)/launches/page.tsx:1-11`).
- `apps/frontend/src/components/launches/launches.component.tsx` is the calendar workspace: it loads integrations, groups channels, opens creation flows, and renders the calendar (`apps/frontend/src/components/launches/launches.component.tsx:352-604`).
- `apps/frontend/src/components/launches/calendar.context.tsx` is the read-model state center for the calendar: it fetches `/posts`, `/posts/list`, `/sets`, and `/signatures/default`, then exposes posts and actions through context (`apps/frontend/src/components/launches/calendar.context.tsx:134-185`, `apps/frontend/src/components/launches/calendar.context.tsx:201-256`, `apps/frontend/src/components/launches/calendar.context.tsx:325-351`).
- `apps/frontend/src/components/new-launch/manage.modal.tsx` is the write-path center for creating or updating posts: it validates editor refs, checks shortlink preference, builds the payload, and sends `POST /posts` (`apps/frontend/src/components/new-launch/manage.modal.tsx:202-334`, `apps/frontend/src/components/new-launch/manage.modal.tsx:336-447`).
- `apps/backend/src/api/routes/posts.controller.ts` is the backend HTTP surface for the same feature (`apps/backend/src/api/routes/posts.controller.ts:111-179`, `apps/backend/src/api/routes/posts.controller.ts:205-229`).
- `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts` contains business behavior: DTO mapping, provider settings injection, shortlink conversion, Temporal workflow start, and post creation (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:243-292`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:750-808`).
- `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts` contains the Prisma queries and upserts behind calendar reads and post writes (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:129-220`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:488-630`).

## Where A Junior Might Get Confused

- The word "launches" means the scheduling calendar, not a deployment launch. The route `/launches` returns `LaunchesComponent` (`apps/frontend/src/app/(app)/(site)/launches/page.tsx:1-11`).
- Client fetches are not raw `fetch` calls. `useFetch` comes from a context wrapper, and `customFetch` adds the backend base URL, credentials, headers, and no-store behavior (`libraries/helpers/src/utils/custom.fetch.tsx:27-46`, `libraries/helpers/src/utils/custom.fetch.func.ts:16-84`).
- Backend controller methods often look tiny because real logic lives in services and repositories. For example, `PostsController.createPost` logs, maps the raw body, then delegates creation (`apps/backend/src/api/routes/posts.controller.ts:170-179`).

## Where A Mid-Level Engineer Should Slow Down

- Calendar data is split between calendar view and list view. The context uses one SWR key for `/posts` when display is not `list`, and another key for `/posts/list` when display is `list` (`apps/frontend/src/components/launches/calendar.context.tsx:174-231`).
- The post creation payload is assembled from UI/editor state, not from a single form object. `ManageModal` pulls from Zustand state and editor refs before serializing integrations, settings, content, delay, media, tags, date, and shortlink choice (`apps/frontend/src/components/new-launch/manage.modal.tsx:69-101`, `apps/frontend/src/components/new-launch/manage.modal.tsx:248-401`).
- The repository writes thread-like posts by iterating `body.value`, connecting each new child to the previous post through `parentPost`, then optionally soft-deleting the old group on update (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:500-560`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:602-630`).

## Where A Senior Engineer Should Be Skeptical

- `PostsController.createPost` accepts `rawBody: any`, logs the whole body, and relies on service-side mapping for validation (`apps/backend/src/api/routes/posts.controller.ts:170-179`). That is flexible but weakens controller-level API contracts and may expose sensitive content in logs.
- The repo is configured as strict at the root and frontend level (`tsconfig.base.json:22-27`, `apps/frontend/tsconfig.json:3-19`), but backend build settings disable several strictness checks (`apps/backend/tsconfig.build.json:16-20`).
- Test discovery found no `*.spec.ts`, `*.test.ts`, `*.spec.tsx`, or `*.test.tsx` files, while the root `test` script still runs Jest with coverage (`package.json:40`, `jest.config.ts:1-5`). This suggests test infrastructure exists but feature tests are absent or outside the searched patterns.

## How To Use Checkpoints

Answer each checkpoint using file evidence. A strong answer should name the file, the line range, the local behavior, and the consequence. In this repo, an answer like "the frontend posts data to the backend" is too shallow; a stronger answer says "the modal sends `POST /posts` after validating `checkAllValid`, then the controller maps raw input through `PostsService.mapTypeToPost` before `createPost` persists it" with citations (`apps/frontend/src/components/new-launch/manage.modal.tsx:248-425`, `apps/backend/src/api/routes/posts.controller.ts:170-179`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:243-292`).

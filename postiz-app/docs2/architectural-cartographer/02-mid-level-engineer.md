# Mid-Level Engineer Guide

## Architecture Diagram

```text
Browser
  |
  | Next App Router route /launches
  v
apps/frontend/src/app/(app)/(site)/launches/page.tsx
  |
  v
LaunchesComponent -> CalendarWeekProvider -> Calendar / NewPost / AddEditModal
  |                       |
  |                       | useSWR + useFetch
  v                       v
customFetch with backendUrl, credentials, auth/showorg headers
  |
  v
Nest backend main.ts -> ApiModule -> AuthMiddleware -> PostsController
  |
  v
PostsService -> IntegrationService / ShortLinkService / IntegrationManager
  |
  v
PostsRepository -> PrismaService -> PostgreSQL
  |
  v
Temporal workflow start for queued publishing
```

The diagram follows the cited route, context, fetch wrapper, API module, middleware, controller, service, repository, and schema paths (`apps/frontend/src/app/(app)/(site)/launches/page.tsx:1-11`, `apps/frontend/src/components/launches/calendar.context.tsx:174-231`, `libraries/helpers/src/utils/custom.fetch.func.ts:46-84`, `apps/backend/src/api/api.module.ts:48-109`, `apps/backend/src/services/auth/auth.middleware.ts:33-104`, `apps/backend/src/api/routes/posts.controller.ts:170-179`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:750-808`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:488-630`).

## Type System Deep Dive

The frontend imports Prisma types into client code for read models, such as `Post`, `Integration`, and `Tags` in the calendar context (`apps/frontend/src/components/launches/calendar.context.tsx:14-25`, `apps/frontend/src/components/launches/calendar.context.tsx:29-81`). That gives quick alignment with database shape, but it can blur API response boundaries because minified/expanded post data is not identical to raw Prisma rows (`apps/frontend/src/components/launches/calendar.context.tsx:183-198`, `libraries/helpers/src/utils/posts.list.minify.ts` found by file path search).

The create command uses a stronger backend DTO. `PostContent` validates content, id, delay, and media; `Post` validates integration, values, group, and provider settings; `CreatePostDto` validates command type, shortlink flag, schedule date, tags, and posts array (`libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts:31-80`, `libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts:93-125`). The controller does not declare that DTO at the boundary; the service creates a `ValidationPipe` and transforms mapped data into `CreatePostDto` (`apps/backend/src/api/routes/posts.controller.ts:170-179`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:280-291`).

## State Management Deep Dive

Calendar read state has a home: `CalendarWeekProvider`. It owns filters, URL synchronization, calendar/list SWR fetches, sets, signature, internal optimistic post data, and reload actions (`apps/frontend/src/components/launches/calendar.context.tsx:134-351`). Composer write state has a separate home: `useLaunchStore`, a Zustand store holding selected integrations, global/internal post values, date, tags, repeater, editor type, locked flag, and setter/mutator functions (`apps/frontend/src/components/new-launch/store.ts:10-160`, `apps/frontend/src/components/new-launch/store.ts:160-654`). This split is useful because calendar state is page-level read state, while composer state is modal-level editing state.

Annotated store slice:

```ts
// apps/frontend/src/components/new-launch/store.ts:22-48
export interface SelectedIntegrations {
  settings: any; // Provider-specific settings are flexible here.
  integration: Integrations; // The chosen social channel.
  ref?: RefObject<any>; // The modal validates each provider/editor through refs.
}

interface StoreState {
  editor: undefined | 'none' | 'normal' | 'markdown' | 'html'; // Matches integration editor modes.
  date: dayjs.Dayjs; // Composer schedule time.
  tags: { label: string; value: string }[]; // Later serialized into CreatePostDto tags.
  selectedIntegrations: SelectedIntegrations[]; // Channels selected for the current command.
  global: Values[]; // Shared content before per-channel overrides.
  internal: Internal[]; // Per-integration content overrides.
}
```

## API Contract Map

- `GET /posts`: frontend calendar view (`apps/frontend/src/components/launches/calendar.context.tsx:174-185`) to controller (`apps/backend/src/api/routes/posts.controller.ts:111-117`) to repository date query (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:129-220`).
- `GET /posts/list`: frontend list view (`apps/frontend/src/components/launches/calendar.context.tsx:187-231`) to controller (`apps/backend/src/api/routes/posts.controller.ts:132-138`) to paginated repository query (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:222-260`).
- `GET /posts/find-slot`: new post button (`apps/frontend/src/components/launches/new.post.tsx:17-19`) to controller (`apps/backend/src/api/routes/posts.controller.ts:119-130`).
- `POST /posts/should-shortlink`: composer shortlink prompt (`apps/frontend/src/components/new-launch/manage.modal.tsx:336-370`) to controller (`apps/backend/src/api/routes/posts.controller.ts:66-69`).
- `POST /posts`: composer submit (`apps/frontend/src/components/new-launch/manage.modal.tsx:419-425`) to controller (`apps/backend/src/api/routes/posts.controller.ts:170-179`) to service/repository (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:750-808`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:488-630`).
- `DELETE /posts/:group`: calendar/modal delete (`apps/frontend/src/components/launches/calendar.tsx:197-220`, `apps/frontend/src/components/new-launch/manage.modal.tsx:180-200`) to controller (`apps/backend/src/api/routes/posts.controller.ts:205-211`).
- `PUT /posts/:id/date`: drag/drop reschedule (`apps/frontend/src/components/launches/calendar.tsx:727-735`) to controller (`apps/backend/src/api/routes/posts.controller.ts:213-221`) to service (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:843-875`).

## Component Interaction Map

`LaunchesComponent` loads integrations through `useIntegrationList`, computes sorted/grouped channel menu data, renders `NewPost`, wraps `Calendar` in `CalendarWeekProvider`, and hands integrations into context (`apps/frontend/src/components/launches/launches.component.tsx:352-422`, `apps/frontend/src/components/launches/launches.component.tsx:497-603`). `NewPost` calls `/posts/find-slot`, optionally asks for a set, and opens `AddEditModal` (`apps/frontend/src/components/launches/new.post.tsx:17-75`). `AddEditModal` initializes Zustand with date/integrations/set state and delegates to `ManageModal` (`apps/frontend/src/components/new-launch/add.edit.modal.tsx:36-107`, `apps/frontend/src/components/new-launch/add.edit.modal.tsx:109-225`). `ManageModal` controls validation, preview/settings panels, tags, repeat, date, draft/schedule/post-now actions, and Copilot helper (`apps/frontend/src/components/new-launch/manage.modal.tsx:202-447`, `apps/frontend/src/components/new-launch/manage.modal.tsx:449-698`).

## Full-Stack Feature Trace: Schedule A Post

1. User opens `/launches`; route returns `LaunchesComponent` (`apps/frontend/src/app/(app)/(site)/launches/page.tsx:1-11`).
2. Authenticated shell has already loaded `/user/self`, because `LayoutComponent` uses SWR and does not render children until `user` exists (`apps/frontend/src/components/new-layout/layout.component.tsx:59-73`).
3. `LaunchesComponent` loads connected channels from `/integrations/list` and shows `NewPost` only when integrations exist (`apps/frontend/src/components/launches/launches.component.tsx:352-365`, `apps/frontend/src/components/launches/launches.component.tsx:538-544`).
4. `NewPost` asks backend for a free slot, then opens the composer modal (`apps/frontend/src/components/launches/new.post.tsx:17-75`).
5. `AddEditModal` copies date and integrations into `useLaunchStore` (`apps/frontend/src/components/new-launch/add.edit.modal.tsx:36-60`).
6. `ManageModal` validates nonempty content/media, provider settings, max length, and shortlink preference (`apps/frontend/src/components/new-launch/manage.modal.tsx:248-370`).
7. `ManageModal` builds payload with type, tags, shortLink, date, integration ids, group, settings, content, delay, and media metadata (`apps/frontend/src/components/new-launch/manage.modal.tsx:372-401`).
8. `ManageModal` sends `POST /posts`, then mutates calendar data and closes the modal (`apps/frontend/src/components/new-launch/manage.modal.tsx:419-443`).
9. `customFetch` prefixes `backendUrl`, includes credentials, adds JSON headers, and sends auth/org/impersonate headers when applicable (`libraries/helpers/src/utils/custom.fetch.func.ts:46-74`).
10. `AuthMiddleware` verifies JWT, loads the org, and attaches `req.user`/`req.org` (`apps/backend/src/services/auth/auth.middleware.ts:33-104`).
11. `PostsController.createPost` checks policy, maps raw body through `PostsService.mapTypeToPost`, and calls `createPost` with creation method `WEB` (`apps/backend/src/api/routes/posts.controller.ts:170-179`).
12. `PostsService.mapTypeToPost` verifies every post has an integration id, loads each integration, injects provider `__type`, and runs a validation pipe against `CreatePostDto` (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:243-292`).
13. `PostsService.createPost` shortlinks or strips links, calls repository upsert, starts a Temporal workflow for non-update operations, and returns post ids (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:750-808`).
14. `PostsRepository.createOrUpdatePost` upserts each thread value, connects integration and organization, sets `DRAFT` or `QUEUE`, stores media/settings JSON, applies tags to the first post, and soft-deletes the previous group when updating (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:488-630`).
15. The UI reloads via SWR mutate, and `CalendarWeekProvider` refetches `/posts` or `/posts/list` (`apps/frontend/src/components/launches/calendar.context.tsx:316-320`).

## Diff Reading Exercise

Hypothetical change: "Add a new `approvalRequired` boolean to scheduled posts."

Read the diff in this order:
1. Schema: would `Post` gain a Boolean and index? Current `Post` fields live in `schema.prisma:393-440`.
2. DTO: would `CreatePostDto` validate the flag? Current create payload fields live in `create.post.dto.ts:93-125`.
3. UI state: would `useLaunchStore` need the field or can `ManageModal` derive it? Current modal payload lives in `manage.modal.tsx:372-401`.
4. Controller/service: would `mapTypeToPost` preserve it through validation? Current mapping is in `posts.service.ts:243-292`.
5. Repository: would `createOrUpdatePost` write it on create/update? Current upsert data is in `posts.repository.ts:500-560`.
6. Read model: would calendar queries select it? Current calendar select is in `posts.repository.ts:173-196`.

Strong review comment: "The schema and write path were updated, but `getPosts` does not select `approvalRequired`, so the calendar cannot show pending approval state after reload."

## Non-Obvious Patterns

- Route files are intentionally thin. `/launches` delegates immediately to a feature component (`apps/frontend/src/app/(app)/(site)/launches/page.tsx:1-11`).
- The app uses response headers as UI control signals. `LayoutContext` watches `auth`, `showorg`, `impersonate`, `logout`, `reload`, `onboarding`, 401, 402, and 406 (`apps/frontend/src/components/layout/layout.context.tsx:27-121`).
- The calendar has a read/write split: `CalendarWeekProvider` reads posts and sets state, while `useLaunchStore` holds modal edit state (`apps/frontend/src/components/launches/calendar.context.tsx:134-351`, `apps/frontend/src/components/new-launch/store.ts:160-654`).
- Provider-specific settings use a discriminator injected from integration provider identifier (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:268-275`, `libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts:71-80`).
- Thread-like posts are modeled as parent-child rows within one group, not as one row with an array of messages (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:404-430`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:500-560`).

## Mid-Level Socratic Checkpoint

1. Why does the calendar have both `posts` and `listPosts`?
2. What does `useFetch` add that a raw `fetch` would not?
3. Where does provider-specific post validation get its discriminator?
4. How does the backend ensure an integration belongs to the current organization during write?
5. What happens to old rows when a grouped post is updated?
6. Which line starts the publishing workflow after a scheduled post is created?
7. Why is `rawBody: any` risky even though service validation exists?
8. What would break if `/posts/list` returned raw Prisma rows instead of minified/expanded data?

## How To Self-Grade

Strong answers cite `calendar.context.tsx:187-231`, `custom.fetch.func.ts:46-84`, `posts.service.ts:268-291`, `posts.repository.ts:503-507`, `posts.repository.ts:617-628`, `posts.service.ts:791-798`, `posts.controller.ts:170-179`, and `calendar.context.tsx:196-231`. Good answers also explain consequences: stale UI, wrong org access, invalid provider settings, orphaned thread rows, or unexpected response shape.

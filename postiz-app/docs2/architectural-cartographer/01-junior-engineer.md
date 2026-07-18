# Junior Engineer Guide

## Local Setup

Use Node `>=22.12.0 <23.0.0` and pnpm `10.6.1`, because the root package declares those requirements (`package.json:9-12`). Install with `pnpm install`; the root `postinstall` runs Prisma generation (`package.json:33-40`). Copy `.env.example` to `.env`; the required local values include `DATABASE_URL`, `REDIS_URL`, `JWT_SECRET`, `FRONTEND_URL`, `NEXT_PUBLIC_BACKEND_URL`, and `BACKEND_INTERNAL_URL` (`.env.example:3-12`). Start local infrastructure with `pnpm run dev:docker`, which uses `docker-compose.dev.yaml` (`package.json:31`) and starts PostgreSQL, Redis, pgAdmin, Redis Insight, and Temporal services (`docker-compose.dev.yaml:5-30`, `docker-compose.dev.yaml:31-56`, `docker-compose.dev.yaml:57-137`). Push the schema with `pnpm run prisma-db-push` (`package.json:33-36`). Start the app with `pnpm run dev` for extension, orchestrator, backend, and frontend together, or `pnpm run dev-backend` for only backend and frontend (`package.json:14-15`). The frontend runs on port 4200 (`apps/frontend/package.json:7`), and the backend defaults to port 3000 (`apps/backend/src/main.ts:69-77`).

No seed script was found in the root scripts; setup can create schema but not sample data by a named seed command (`package.json:13-40`). That means a new developer should expect to register a user through the app or insert test data manually after Prisma schema push.

## Folder Orientation

- `apps/frontend`: Next/React app. App Router files live in `apps/frontend/src/app`, and feature components live in `apps/frontend/src/components` as shown by the Launches route and component imports (`apps/frontend/src/app/(app)/(site)/launches/page.tsx:1-11`, `apps/frontend/src/components/launches/launches.component.tsx:1-29`).
- `apps/backend`: Nest API app. Entry point and module live at `apps/backend/src/main.ts` and `apps/backend/src/app.module.ts` (`apps/backend/src/main.ts:1-22`, `apps/backend/src/app.module.ts:21-66`).
- `apps/orchestrator`: Nest/Temporal worker app. The root dev script runs it beside backend/frontend (`package.json:14`, `apps/orchestrator/package.json:6-9`).
- `apps/extension`: browser extension app. Its package declares Vite build/dev scripts (`apps/extension/package.json:4-9`).
- `libraries/helpers`: shared helpers such as `customFetch` and post minify utilities (`libraries/helpers/src/utils/custom.fetch.func.ts:1-92`).
- `libraries/nestjs-libraries`: backend services, Prisma layer, integrations, upload, Temporal, chat, and DTOs. The database module imports and exports many services/repositories (`libraries/nestjs-libraries/src/database/prisma/database.module.ts:1-101`).
- `libraries/react-shared-libraries`: shared React helpers, forms, toaster, translations, and Sentry helpers, visible through imports like `@gitroom/react/toaster/toaster` and `@gitroom/react/form/button` (`apps/frontend/src/components/new-launch/manage.modal.tsx:25-47`).

## One Level Deeper: Frontend

1. `apps/frontend/src/app`: route tree. `(app)` is the main app layout (`apps/frontend/src/app/(app)/layout.tsx:34-121`); `(site)` wraps authenticated pages in `LayoutComponent` (`apps/frontend/src/app/(app)/(site)/layout.tsx:1-9`); `/launches` delegates to `LaunchesComponent` (`apps/frontend/src/app/(app)/(site)/launches/page.tsx:1-11`).
2. `apps/frontend/src/components/launches`: scheduling calendar workspace. It includes integration menu, filters, calendar views, drag/drop, new-post entry, and context (`apps/frontend/src/components/launches/launches.component.tsx:352-604`, `apps/frontend/src/components/launches/calendar.tsx:567-822`).
3. `apps/frontend/src/components/new-launch`: post composer. It has a Zustand store (`apps/frontend/src/components/new-launch/store.ts:10-160`), modal bootstrapping (`apps/frontend/src/components/new-launch/add.edit.modal.tsx:36-107`), and scheduling command serialization (`apps/frontend/src/components/new-launch/manage.modal.tsx:202-447`).

## One Level Deeper: Backend

1. `apps/backend/src/api/routes`: HTTP controllers. `PostsController` exposes `/posts`, `/posts/list`, `/posts/find-slot`, `/posts/:id/date`, and related operations (`apps/backend/src/api/routes/posts.controller.ts:111-229`).
2. `apps/backend/src/services/auth`: request auth. `AuthMiddleware` reads JWT from headers/cookies, loads organization, supports impersonation for super admins, and sets request fields (`apps/backend/src/services/auth/auth.middleware.ts:33-104`).
3. `libraries/nestjs-libraries/src/database/prisma`: persistence. `PrismaService` connects/disconnects Prisma (`libraries/nestjs-libraries/src/database/prisma/prisma.service.ts:4-23`), `DatabaseModule` registers services/repositories (`libraries/nestjs-libraries/src/database/prisma/database.module.ts:46-101`), and post service/repository hold scheduling logic (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:750-808`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:488-630`).

## Frontend Entry Point Walkthrough

```tsx
// apps/frontend/src/app/(app)/(site)/launches/page.tsx:1-11
export const dynamic = 'force-dynamic'; // This page opts out of static caching for user-specific scheduling data.
import { LaunchesComponent } from '@gitroom/frontend/components/launches/launches.component'; // The route delegates feature ownership.
import { Metadata } from 'next';
import { isGeneralServerSide } from '@gitroom/helpers/utils/is.general.server.side';

export const metadata: Metadata = {
  title: `${isGeneralServerSide() ? 'Postiz Calendar' : 'Gitroom Launches'}`, // Product naming changes by deployment mode.
  description: '',
};

export default async function Index() {
  return <LaunchesComponent />; // The page file stays thin; behavior is in the component tree.
}
```

The route depends on the site layout, which wraps children with `LayoutComponent` (`apps/frontend/src/app/(app)/(site)/layout.tsx:1-9`), and the root app layout wraps everything with environment and fetch providers (`apps/frontend/src/app/(app)/layout.tsx:57-117`).

## Backend Entry Point Walkthrough

```ts
// apps/backend/src/main.ts:22-80
async function start() {
  const app = await NestFactory.create(AppModule, {
    rawBody: true, // Stripe/webhooks may need the unparsed body.
    cors: {
      ...(!process.env.NOT_SECURED ? { credentials: true } : {}), // Cookie auth is enabled in secured mode.
      allowedHeaders: ['Content-Type', 'Authorization', 'auth', 'showorg', 'impersonate'],
      exposedHeaders: ['reload', 'onboarding', 'activate'], // Frontend afterRequest reacts to these headers.
      origin: [process.env.FRONTEND_URL, 'http://localhost:6274', ...(process.env.MAIN_URL ? [process.env.MAIN_URL] : [])],
    },
  });

  app.useGlobalPipes(new ValidationPipe({ transform: true })); // DTO decorators can transform query/body data.
  app.use(['/copilot/{*splat}', '/posts'], (req, res, next) => {
    json({ limit: '50mb' })(req, res, next); // Composer payloads can include rich content and media metadata.
  });

  app.use(cookieParser()); // AuthMiddleware can read req.cookies.auth.
  app.use(compression());
  app.useGlobalFilters(new SubscriptionExceptionFilter());
  app.useGlobalFilters(new HttpExceptionFilter());

  const port = process.env.PORT || 3000;
  await app.listen(port); // Backend defaults to localhost:3000.
}
```

## TypeScript Orientation

- Path aliases let code import by domain instead of relative paths. For example, `@gitroom/frontend/*`, `@gitroom/helpers/*`, and `@gitroom/nestjs-libraries/*` are declared in `tsconfig.base.json:27-37`.
- Frontend strictness is enabled in `apps/frontend/tsconfig.json:3-19`.
- Backend build settings are looser for null checks, implicit any, bind/call/apply, casing, and switch fallthrough (`apps/backend/tsconfig.build.json:16-20`).
- DTO decorators define runtime validation for backend inputs. `CreatePostDto` requires `type`, `shortLink`, `date`, `tags`, and at least one `posts` entry (`libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts:93-125`).

## React Component Anatomy 1: Simple Badge

```tsx
// apps/frontend/src/components/launches/creation.method.badge.tsx:4-23
type CreationMethod = 'UNKNOWN' | 'WEB' | 'API' | 'MCP' | 'AUTOPOST' | 'CLI'; // UI accepts the same creation concepts stored in Prisma.

interface Props {
  creationMethod?: CreationMethod | string | null; // Defensive: data can arrive as typed enum or plain string.
  size?: 'xs' | 'sm' | 'md';
  className?: string;
  ringColor?: string;
}

export const CreationMethodBadge: FC<Props> = ({ creationMethod, size = 'xs', className, ringColor }) => {
  if (!creationMethod || creationMethod === 'UNKNOWN') return null; // No visible badge for unknown origin.
```

The matching Prisma enum defines `UNKNOWN`, `WEB`, `MCP`, `API`, `AUTOPOST`, and `CLI` (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:943-950`).

## React Component Anatomy 2: Custom Hook

```tsx
// apps/frontend/src/components/launches/helpers/use.integration.list.tsx:7-23
export const useIntegrationList = () => {
  const fetch = useFetch(); // Uses the app fetch wrapper, not global fetch.

  const load = useCallback(async (path: string) => {
    return (await (await fetch(path)).json()).integrations; // Backend response shape is { integrations }.
  }, []);

  return useSWR('/integrations/list', load, {
    revalidateOnFocus: false, // Calendar channel list should not jitter when tab focus changes.
    fallbackData: [], // Components can map integrations before the request finishes.
  });
};
```

The backend route for `/integrations/list` is on `IntegrationsController` (`apps/backend/src/api/routes/integrations.controller.ts:37-88`).

## React Component Anatomy 3: Complex Composer Submit

```tsx
// apps/frontend/src/components/new-launch/manage.modal.tsx:336-425
const shortlinkPreference = shortlinkPreferenceData?.shortlink || 'ASK'; // User-level setting comes from useShortlinkPreference.
let shortLink = false;

if (!dummy && shortlinkPreference !== 'NO') {
  const shortLinkUrl = await (await fetch('/posts/should-shortlink', {
    method: 'POST',
    body: JSON.stringify({
      messages: checkAllValid
        .filter((p: any) => !p?.integration?.stripLinks) // Do not ask to shortlink platforms that remove links.
        .flatMap((p: any) => p.values.flatMap((a: any) => a.content)),
    }),
  })).json();
  if (shortLinkUrl.ask) shortLink = shortlinkPreference === 'YES' || await deleteDialog(...);
}

const data = {
  type,
  tags,
  shortLink,
  date: date.utc().format('YYYY-MM-DDTHH:mm:ss'), // Backend receives UTC-like schedule time.
  posts: checkAllValid.map((post: any) => ({
    integration: { id: post.integration.id }, // Backend validates that this integration belongs to the org.
    group,
    settings: { ...(post.settings || {}) },
    value: post.values.map((value: any) => ({
      content: value.content,
      delay: value.delay || 0,
      image: (value?.media || []).map(({ id, path, alt, thumbnail, thumbnailTimestamp }: any) => ({ id, path, alt, thumbnail, thumbnailTimestamp })),
    })),
  })),
};

await fetch('/posts', { method: 'POST', body: JSON.stringify(data) }); // Command crosses to Nest here.
```

## Backend Anatomy 1: Simple Controller

```ts
// apps/backend/src/api/routes/posts.controller.ts:119-130
@Get('/find-slot')
async findSlot(@GetOrgFromRequest() org: Organization) {
  return { date: await this._postsService.findFreeDateTime(org.id) }; // Small route: org-scoped service call.
}

@Get('/find-slot/:id')
async findSlotIntegration(@GetOrgFromRequest() org: Organization, @Param('id') id?: string) {
  return { date: await this._postsService.findFreeDateTime(org.id, id) }; // Optional integration narrows slot search.
}
```

`NewPost` calls `/posts/find-slot` before opening the composer (`apps/frontend/src/components/launches/new.post.tsx:17-19`).

## Backend Anatomy 2: Create Post Controller

```ts
// apps/backend/src/api/routes/posts.controller.ts:170-179
@Post('/')
@CheckPolicies([AuthorizationActions.Create, Sections.POSTS_PER_MONTH]) // Subscription/permission gate.
async createPost(@GetOrgFromRequest() org: Organization, @Body() rawBody: any) {
  console.log(JSON.stringify(rawBody, null, 2)); // Risk: social post content is logged.
  const body = await this._postsService.mapTypeToPost(rawBody, org.id); // Runtime DTO validation happens here.
  return this._postsService.createPost(org.id, body, 'WEB'); // Creation method is persisted as WEB.
}
```

The service maps provider settings before DTO validation (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:243-292`).

## Backend Anatomy 3: Repository Upsert

```ts
// libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:500-560
for (const value of body.value) {
  const updateData = (type: 'create' | 'update') => ({
    publishDate: dayjs(date).toDate(),
    integration: { connect: { id: body.integration.id, organizationId: orgId } }, // Guards the relation by org.
    ...(posts?.[posts.length - 1]?.id ? { parentPost: { connect: { id: posts[posts.length - 1]?.id } } } : {}),
    content: value.content,
    delay: value.delay || 0,
    group: uuid,
    intervalInDays: inter ? +inter : null,
    state: state === 'draft' ? 'DRAFT' : 'QUEUE',
    image: JSON.stringify(value.image),
    settings: JSON.stringify(body.settings),
    organization: { connect: { id: orgId } },
  });

  posts.push(await this._post.model.post.upsert({
    where: { id: value.id || uuidv4() }, // Existing value id means edit; missing id means create.
    create: { ...updateData('create') },
    update: { ...updateData('update'), lastMessage: { disconnect: true }, submittedForOrder: { disconnect: true } },
  }));
}
```

## Domain Glossary

- Post: one scheduled content unit stored in `Post`, with state, publish date, integration, content, group, optional parent post, media JSON, tags, and creation method (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:393-440`).
- Group: a shared id tying one or more posts/thread items together; repository uses `group` for updates/deletes (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:526-530`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:602-630`).
- Integration/channel: a connected social account with provider identifier, token, disabled flag, customer grouping, posting times, and posts relation (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:314-356`).
- Customer: grouping for integrations, used by calendar filters and channel menu grouping (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:301-312`, `apps/frontend/src/components/launches/launches.component.tsx:405-422`).
- Tag: organization-scoped label/color that can attach to posts through `TagsPosts` (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:52-77`).
- Media: stored uploaded asset metadata for local or cloud storage (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:209-231`).
- State: post lifecycle enum with `QUEUE`, `PUBLISHED`, `ERROR`, and `DRAFT` (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:903-908`).
- Shortlink preference: organization setting with `ASK`, `YES`, or `NO`, used by composer shortlink behavior (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:952-956`, `apps/frontend/src/components/settings/shortlink-preference.component.tsx:10-31`).
- Launches: the scheduling calendar feature route, not deployments (`apps/frontend/src/app/(app)/(site)/launches/page.tsx:1-11`).

## Junior Socratic Checkpoint

1. Which command starts frontend, backend, orchestrator, and extension together?
2. Where does the frontend get the backend URL used by `useFetch`?
3. Which file is the route entry for `/launches`?
4. Which backend code sets `req.org`?
5. Why does `POST /posts` not receive a typed DTO directly in the controller?
6. Which Prisma model stores a scheduled social post?
7. Where does the UI build the JSON body sent to `POST /posts`?

## How To Self-Grade

Strong answers cite `package.json:14`, `apps/frontend/src/components/layout/layout.context.tsx:24-129`, `apps/frontend/src/app/(app)/(site)/launches/page.tsx:1-11`, `apps/backend/src/services/auth/auth.middleware.ts:78-99`, `apps/backend/src/api/routes/posts.controller.ts:170-179`, `libraries/nestjs-libraries/src/database/prisma/schema.prisma:393-440`, and `apps/frontend/src/components/new-launch/manage.modal.tsx:372-425`. A developing answer names the right concept but cannot point to files yet. A weak answer guesses from framework knowledge instead of evidence.

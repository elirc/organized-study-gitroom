# Tier 1 Junior Missions

### Mission 1: The App's Heartbeat
**Tier:** Junior  
**Time Estimate:** 30 minutes  
**Goal:** Identify how the repo starts, builds, tests, and talks to local infrastructure.  
**The Concept:** Scripts are the pulse of the app; like a scheduling platform needs clocks and queues, a repo needs commands that start its moving parts.  
**Design Intent Before You Read the Code:** The root package owns cross-app commands, while each app package owns its local dev/build command. Poor script design makes onboarding ambiguous.  
**Find It In The Code:** Open `package.json:9-40`, `apps/frontend/package.json:6-13`, `apps/backend/package.json:5-9`, `.env.example:3-12`, and `docker-compose.dev.yaml:5-30`.

```json
// package.json:14-15
"dev": "pnpm run --filter ./apps/extension --filter ./apps/orchestrator --filter ./apps/backend --filter ./apps/frontend --parallel dev", // starts all active dev apps
"dev-backend": "pnpm run --filter ./apps/backend --filter ./apps/frontend --parallel dev", // lighter loop for API + UI work

// package.json:31-40
"dev:docker": "docker compose -f ./docker-compose.dev.yaml up -d", // starts local dependencies
"prisma-db-push": "pnpm dlx prisma@6.5.0 db push --accept-data-loss --schema ./libraries/nestjs-libraries/src/database/prisma/schema.prisma", // syncs schema
"test": "jest --coverage --detectOpenHandles --reporters=default --reporters=jest-junit" // test runner exists even though no test files were found
```

**The Aha Moment:** The repo is not one server; it is a coordinated workspace with separate UI, API, worker, and extension commands.  
**Socratic Checkpoint:** What Node version is expected? Which script starts all apps? Which script starts only frontend/backend? Which command creates local infrastructure? Which command pushes Prisma schema?  
**How to self-grade:** Strong answers cite `package.json:9-40`, `.env.example:3-12`, and `docker-compose.dev.yaml:5-30`.  
**Connects To:** Mission 2 because scripts only make sense once you know the folders they target.

### Mission 2: The Folder Mental Map
**Tier:** Junior  
**Time Estimate:** 35 minutes  
**Goal:** Build a mental map of frontend, backend, shared helpers, and Prisma libraries.  
**The Concept:** The repo is a social scheduling office: frontend is the dashboard, backend is the operations desk, Prisma is the filing system, and orchestrator is dispatch.  
**Design Intent Before You Read the Code:** Folders should reflect ownership. If a feature's files scatter without pattern, changes become expensive.  
**Find It In The Code:** Open `tsconfig.base.json:27-37`, `apps/frontend/src/app/(app)/(site)/launches/page.tsx:1-11`, `apps/backend/src/api/api.module.ts:48-109`, and `libraries/nestjs-libraries/src/database/prisma/database.module.ts:46-101`.

```ts
// tsconfig.base.json:27-37
"paths": {
  "@gitroom/backend/*": ["apps/backend/src/*"], // backend app code
  "@gitroom/frontend/*": ["apps/frontend/src/*"], // frontend app code
  "@gitroom/helpers/*": ["libraries/helpers/src/*"], // shared non-React helpers
  "@gitroom/nestjs-libraries/*": ["libraries/nestjs-libraries/src/*"], // backend shared services
  "@gitroom/react/*": ["libraries/react-shared-libraries/src/*"] // shared React helpers/components
}
```

**The Aha Moment:** Aliases tell you architectural boundaries faster than folder names alone.  
**Socratic Checkpoint:** Which alias maps to frontend? Which maps to backend services? Why do components import from `@gitroom/react`? What file wires authenticated controllers? Where are Prisma services exported?  
**How to self-grade:** Strong answers cite `tsconfig.base.json:27-37`, `apps/backend/src/api/api.module.ts:48-109`, and `libraries/nestjs-libraries/src/database/prisma/database.module.ts:46-101`.  
**Connects To:** Mission 3 because TypeScript aliases and contracts are the next map layer.

### Mission 3: TypeScript Is a Contract
**Tier:** Junior  
**Time Estimate:** 40 minutes  
**Goal:** See TypeScript as a promise between files, not decoration.  
**The Concept:** A scheduled post is like a delivery label: it must include destination, date, content, and handling rules.  
**Design Intent Before You Read the Code:** DTOs should say what backend accepts; props and context types should say what UI needs.  
**Find It In The Code:** Open `libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts:31-80`, `libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts:93-125`, and `apps/frontend/src/components/new-launch/store.ts:10-48`.

```ts
// libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts:93-125
export class CreatePostDto {
  @IsDefined()
  @IsIn(['draft', 'schedule', 'now', 'update'])
  type: 'draft' | 'schedule' | 'now' | 'update'; // command mode

  @IsDefined()
  @IsBoolean()
  shortLink: boolean; // URL handling choice

  @IsDefined()
  @IsDateString()
  date: string; // schedule time

  @ArrayMinSize(1)
  posts: Post[]; // at least one channel/content bundle
}
```

**The Aha Moment:** Types are not only compile-time hints here; class-validator decorators make them runtime gates.  
**Socratic Checkpoint:** Which fields are required? Which decorator limits allowed `type` values? Where is media nested? Which frontend type resembles selected channels? What would happen if `posts` were empty?  
**How to self-grade:** Strong answers cite DTO lines and explain runtime validation through `ValidationPipe` in `posts.service.ts:280-291`.  
**Connects To:** Mission 6 because props are the frontend version of contracts.

### Mission 4: Your First React Component
**Tier:** Junior  
**Time Estimate:** 25 minutes  
**Goal:** Read a small component from inputs to output.  
**The Concept:** A creation-method badge is a tiny label on a scheduled delivery: web, API, MCP, CLI, or autopost.  
**Design Intent Before You Read the Code:** Small components should guard invalid input and render predictable UI.  
**Find It In The Code:** Open `apps/frontend/src/components/launches/creation.method.badge.tsx:4-50`.

```tsx
// apps/frontend/src/components/launches/creation.method.badge.tsx:16-23
export const CreationMethodBadge: FC<Props> = ({ creationMethod, size = 'xs', className, ringColor }) => {
  if (!creationMethod || creationMethod === 'UNKNOWN') return null; // nothing to show

  const sizeClasses = size === 'xs'
    ? 'h-[12px] px-[4px] text-[7px]' // compact calendar badge
    : size === 'md'
    ? 'h-[22px] px-[10px] text-[12px]'
    : 'h-[18px] px-[8px] text-[10px]';
```

**The Aha Moment:** A component's first responsibility is often deciding when not to render.  
**Socratic Checkpoint:** What prop controls visibility? What are the size options? How is tooltip content chosen? Which schema enum matches these values? Why is `UNKNOWN` hidden?  
**How to self-grade:** Strong answers cite `creation.method.badge.tsx:4-50` and `schema.prisma:943-950`.  
**Connects To:** Mission 16 because this is composition in miniature.

### Mission 5: Your First Node.js Route
**Tier:** Junior  
**Time Estimate:** 35 minutes  
**Goal:** Read a Nest controller method and trace its service call.  
**The Concept:** A route is the front desk for a request; it checks identity, reads input, and hands the work to the right department.  
**Design Intent Before You Read the Code:** Controllers should be thin and org-scoped.  
**Find It In The Code:** Open `apps/backend/src/api/routes/posts.controller.ts:111-130`.

```ts
// apps/backend/src/api/routes/posts.controller.ts:111-130
@Get('/')
async getPosts(@GetOrgFromRequest() org: Organization, @Query() query: GetPostsDto) {
  return this._postsService.getPostsMinified(org.id, query); // read calendar posts for current org
}

@Get('/find-slot')
async findSlot(@GetOrgFromRequest() org: Organization) {
  return { date: await this._postsService.findFreeDateTime(org.id) }; // suggest next schedule slot
}
```

**The Aha Moment:** The controller does not know scheduling math; it supplies org context and delegates.  
**Socratic Checkpoint:** Which decorator reads org? Which DTO reads query params? Which method returns a `date` object? Why is `org.id` passed instead of trusting a client org id? What service owns the logic?  
**How to self-grade:** Strong answers cite `posts.controller.ts:111-130` and `org.from.request.ts:1-8`.  
**Connects To:** Mission 13 because middleware explains where org context comes from.

### Mission 6: Props Are a Typed Contract
**Tier:** Junior  
**Time Estimate:** 30 minutes  
**Goal:** Understand a modal prop interface and how it shapes child behavior.  
**The Concept:** Props are instructions on a work order: date, channels, existing content, and callback signatures.  
**Design Intent Before You Read the Code:** Modal props should tell you whether the modal creates, edits, duplicates, or outputs dummy code.  
**Find It In The Code:** Open `apps/frontend/src/components/new-launch/add.edit.modal.tsx:13-34`, `apps/frontend/src/components/new-launch/add.edit.modal.tsx:36-60`.

```tsx
// apps/frontend/src/components/new-launch/add.edit.modal.tsx:13-34
export interface AddEditModalProps {
  dummy?: boolean; // output code instead of sending request
  date: dayjs.Dayjs; // initial schedule date
  integrations: Integrations[]; // visible/eligible channels
  allIntegrations?: Integrations[]; // store seed for modal
  addEditSets?: (data: any) => void; // alternate save target for sets
  reopenModal: () => void; // callback after modal closes/reopens
  mutate: () => void; // refresh caller after write
}
```

**The Aha Moment:** You can predict the modal's behavior from its prop names before reading JSX.  
**Socratic Checkpoint:** Which prop changes behavior from real submit to dummy output? Which prop refreshes parent data? Which prop supplies date? Which prop suggests this modal can edit sets? Which type describes integrations?  
**How to self-grade:** Strong answers cite `add.edit.modal.tsx:13-60`.  
**Connects To:** Mission 14 because these props are part of the full create-post trace.

### Mission 7: Following Data Into the App
**Tier:** Junior  
**Time Estimate:** 45 minutes  
**Goal:** Follow calendar data from HTTP to context to rendered components.  
**The Concept:** Calendar data is the schedule board; SWR is the clerk who keeps it fresh.  
**Design Intent Before You Read the Code:** A read model should centralize fetching, loading state, and reload behavior.  
**Find It In The Code:** Open `apps/frontend/src/components/launches/calendar.context.tsx:174-231`, `apps/frontend/src/components/launches/calendar.context.tsx:285-351`.

```tsx
// apps/frontend/src/components/launches/calendar.context.tsx:174-185
const loadData = useCallback(async () => {
  const modifiedParams = new URLSearchParams({
    display: filters.display,
    customer: filters?.customer?.toString() || '',
    startDate: newDayjs(filters.startDate).startOf('day').utc().format(),
    endDate: newDayjs(filters.endDate).endOf('day').utc().format(),
  }).toString();
  const data = await (await fetch(`/posts?${modifiedParams}`)).json(); // API read
  return expandPosts(data); // convert compact response into UI shape
}, [filters, params]);
```

**The Aha Moment:** The UI does not render raw API data; it expands a transport shape into a calendar shape.  
**Socratic Checkpoint:** Which endpoint loads calendar posts? Which endpoint loads list posts? What controls the SWR key? Where does reload call mutate? Why does date conversion use UTC?  
**How to self-grade:** Strong answers cite `calendar.context.tsx:174-231` and `calendar.context.tsx:316-351`.  
**Connects To:** Mission 9 because state location becomes more important as features grow.

### Mission 8: Navigation Is the App's Skeleton
**Tier:** Junior  
**Time Estimate:** 30 minutes  
**Goal:** Understand how a route becomes a screen inside the authenticated layout.  
**The Concept:** Navigation is the building hallway: the route points to a room, and layout supplies the walls, doors, and user identity.  
**Design Intent Before You Read the Code:** Routes should be thin; layout should supply cross-cutting UI.  
**Find It In The Code:** Open `apps/frontend/src/app/(app)/(site)/layout.tsx:1-9`, `apps/frontend/src/components/new-layout/layout.component.tsx:52-149`, and `apps/frontend/src/components/layout/title.tsx:1-14`.

```tsx
// apps/frontend/src/app/(app)/(site)/layout.tsx:1-9
import { LayoutComponent } from '@gitroom/frontend/components/new-layout/layout.component';

export default async function Layout({ children }: { children: React.ReactNode }) {
  return <LayoutComponent>{children}</LayoutComponent>; // every site page gets the authenticated shell
}

// apps/frontend/src/components/layout/title.tsx:6-13
export const Title = () => {
  const path = usePathname(); // current route
  const { all: menuItems } = useMenuItem(); // menu model
  const currentTitle = useMemo(() => menuItems.find((item) => path.indexOf(item.path) > -1)?.name, [path]);
  return <h1>{currentTitle}</h1>; // title follows route
};
```

**The Aha Moment:** Page files choose content; layouts provide identity, navigation, and shared tools.  
**Socratic Checkpoint:** Which layout wraps site pages? Where is `/user/self` loaded? Where is the page title derived? Why can `/launches/page.tsx` stay tiny? What happens if user data is missing?  
**How to self-grade:** Strong answers cite `layout.component.tsx:59-73` and `layout.component.tsx:91-149`.  
**Connects To:** Mission 12 because route skeleton connects to API contract behavior.

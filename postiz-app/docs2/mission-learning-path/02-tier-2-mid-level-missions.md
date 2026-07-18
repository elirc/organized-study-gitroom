# Tier 2 Mid-Level Missions

### Mission 9: State Has a Home and a Reason
**Tier:** Mid-Level  
**Time Estimate:** 45 minutes  
**Goal:** Separate calendar read state from composer write state.  
**The Concept:** The calendar board and the draft desk are separate work surfaces.  
**Design Intent Before You Read the Code:** Page-level state should not be mixed with modal editing state.  
**Find It In The Code:** Open `apps/frontend/src/components/launches/calendar.context.tsx:134-351` and `apps/frontend/src/components/new-launch/store.ts:138-654`.

```tsx
// apps/frontend/src/components/launches/calendar.context.tsx:325-345
<CalendarContext.Provider value={{
  reloadCalendarView, // page read refresh
  ...filters,
  posts: calendarIsLoading ? [] : internalData, // calendar grid data
  integrations,
  setFilters: setFiltersWrapper, // URL + state sync
  changeDate, // optimistic UI change
  listPosts,
  setListPage,
}}>
```

**The Aha Moment:** State location explains who owns a bug.  
**Socratic Checkpoint:** Which state owns filters? Which state owns selected integrations? Which state owns tags? Which state owns reload? Which state owns modal lock?  
**How to self-grade:** Strong answers cite `calendar.context.tsx:158-172`, `calendar.context.tsx:316-345`, and `store.ts:138-160`.  
**Connects To:** Mission 10 because hooks expose state homes.

### Mission 10: The Custom Hook Ecosystem
**Tier:** Mid-Level  
**Time Estimate:** 45 minutes  
**Goal:** Understand repo-specific hooks and what they hide.  
**The Concept:** Hooks are reusable control panels for common operations.  
**Design Intent Before You Read the Code:** A hook should hide repeat mechanics without hiding ownership.  
**Find It In The Code:** Open `apps/frontend/src/components/launches/helpers/use.integration.list.tsx:1-23`, `apps/frontend/src/components/settings/shortlink-preference.component.tsx:16-31`, and `apps/frontend/src/components/launches/helpers/use.existing.data.tsx:1-19`.

```tsx
// apps/frontend/src/components/settings/shortlink-preference.component.tsx:16-31
export const useShortlinkPreference = () => {
  const fetch = useFetch(); // app-level fetch wrapper
  const load = useCallback(async () => (await fetch('/settings/shortlink')).json(), []);
  return useSWR<ShortlinkPreferenceResponse>('shortlink-preference', load, {
    revalidateOnFocus: false, // stable settings read
    revalidateOnMount: true,
  });
};
```

**The Aha Moment:** Hooks in this repo mostly wrap SWR, context, or shared fetch behavior.  
**Socratic Checkpoint:** Which hook loads integrations? Which hook loads shortlink preference? Which hook provides existing edit data? Why do these hooks use `useFetch`? What cache keys are used?  
**How to self-grade:** Strong answers cite all three files and describe cache keys.  
**Connects To:** Mission 11 because hooks often wrap side effects.

### Mission 11: Side Effects Are Promises to the System
**Tier:** Mid-Level  
**Time Estimate:** 50 minutes  
**Goal:** Identify side effects and their obligations.  
**The Concept:** Scheduling is a promise: if the UI says saved, the backend, database, and workflow must agree.  
**Design Intent Before You Read the Code:** Every side effect needs a follow-up: update cache, close modal, redirect, or start workflow.  
**Find It In The Code:** Open `apps/frontend/src/components/new-launch/manage.modal.tsx:419-443`, `apps/frontend/src/components/layout/layout.context.tsx:27-121`, and `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:791-808`.

```tsx
// apps/frontend/src/components/new-launch/manage.modal.tsx:419-443
await fetch('/posts', { method: 'POST', body: JSON.stringify(data) }); // side effect: write
mutate(); // obligation: refresh caller/calendar
toaster.show(t('added_successfully', 'Added successfully')); // obligation: user feedback
modal.closeAll(); // obligation: close command surface
```

**The Aha Moment:** The code after a side effect is as important as the side effect itself.  
**Socratic Checkpoint:** What happens after post creation? Which headers can redirect or reload the app? Where does backend start workflow? What happens if mutate is skipped? What happens if afterRequest returns false?  
**How to self-grade:** Strong answers cite `manage.modal.tsx:419-443`, `layout.context.tsx:45-121`, and `posts.service.ts:791-808`.  
**Connects To:** Mission 14 because the full trace is mostly side effects in sequence.

### Mission 12: The Full API Contract
**Tier:** Mid-Level  
**Time Estimate:** 60 minutes  
**Goal:** Map frontend calls to backend controllers and DTOs.  
**The Concept:** API contracts are the shipping forms for scheduled posts.  
**Design Intent Before You Read the Code:** A contract includes URL, method, auth, payload, validation, response, and cache update.  
**Find It In The Code:** Open `apps/frontend/src/components/launches/calendar.context.tsx:174-231`, `apps/frontend/src/components/new-launch/manage.modal.tsx:336-425`, `apps/backend/src/api/routes/posts.controller.ts:66-179`, and `libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts:93-125`.

```ts
// apps/backend/src/api/routes/posts.controller.ts:170-179
@Post('/')
@CheckPolicies([AuthorizationActions.Create, Sections.POSTS_PER_MONTH]) // authorization contract
async createPost(@GetOrgFromRequest() org: Organization, @Body() rawBody: any) {
  const body = await this._postsService.mapTypeToPost(rawBody, org.id); // validation/normalization contract
  return this._postsService.createPost(org.id, body, 'WEB'); // response is post ids/integration ids
}
```

**The Aha Moment:** An endpoint is not just a URL; it is a chain of assumptions.  
**Socratic Checkpoint:** Which endpoints are read-only? Which endpoint mutates? Which endpoint is policy-gated? Which DTO validates schedule payloads? Which calls must mutate SWR?  
**How to self-grade:** Strong answers create a mini table using the cited files.  
**Connects To:** Mission 13 because contracts depend on middleware.

### Mission 13: The Middleware Chain
**Tier:** Mid-Level  
**Time Estimate:** 50 minutes  
**Goal:** Trace request identity from browser to controller decorators.  
**The Concept:** Middleware is the badge check before entering the scheduling control room.  
**Design Intent Before You Read the Code:** Controllers should not manually decode auth; middleware should attach trusted context.  
**Find It In The Code:** Open `libraries/helpers/src/utils/custom.fetch.func.ts:46-74`, `apps/backend/src/api/api.module.ts:48-109`, `apps/backend/src/services/auth/auth.middleware.ts:33-104`, `libraries/nestjs-libraries/src/user/org.from.request.ts:1-8`.

```ts
// apps/backend/src/services/auth/auth.middleware.ts:33-44
const auth = req.headers.auth || req.cookies.auth; // token source
if (!auth) throw new HttpForbiddenException(); // no identity, no controller
let user = AuthService.verifyJWT(auth) as User | null; // decode signed token
const orgHeader = req.cookies.showorg || req.headers.showorg; // selected org
if (!user || !user.activated) throw new HttpForbiddenException(); // active user only
```

**The Aha Moment:** `@GetOrgFromRequest` works because middleware writes `req.org` first.  
**Socratic Checkpoint:** Where can auth token come from? Which controller group gets middleware? Where is `req.org` assigned? How can super admins impersonate? Why should client org ids not be trusted?  
**How to self-grade:** Strong answers cite `api.module.ts:48-109`, `auth.middleware.ts:33-104`, and `org.from.request.ts:1-8`.  
**Connects To:** Mission 22 because auth context is a security boundary.

### Mission 14: End-to-End Feature Trace
**Tier:** Mid-Level  
**Time Estimate:** 90 minutes  
**Goal:** Trace schedule-post from click to database and UI refresh.  
**The Concept:** Scheduling a post is a dispatch ticket moving from calendar desk to operations to storage to queue.  
**Design Intent Before You Read the Code:** End-to-end tracing should reveal the contract at every handoff.  
**Find It In The Code:** Open `apps/frontend/src/components/launches/new.post.tsx:17-75`, `apps/frontend/src/components/new-launch/manage.modal.tsx:248-425`, `apps/backend/src/api/routes/posts.controller.ts:170-179`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:243-292`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:750-808`, and `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:488-630`.

```ts
// libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:750-808
for (const post of body.posts) { // each selected integration
  const provider = this._integrationManager.getSocialIntegration((post.settings as any)?.__type); // provider behavior
  const removeLinks = !!provider?.stripLinks?.(); // platform-specific rule
  const updateContent = !body.shortLink || removeLinks ? messages : await this._shortLinkService.convertTextToShortLinks(orgId, messages); // URL rule
  const { posts } = await this._postRepository.createOrUpdatePost(...); // database write
  if (body.type !== 'update') this.startWorkflow(...).catch((err) => {}); // dispatch publish workflow
}
```

**The Aha Moment:** The feature is understandable when you follow one payload, not when you scan all files.  
**Socratic Checkpoint:** Where is the slot chosen? Where is payload built? Where is provider `__type` injected? Where are rows upserted? Where does workflow start?  
**How to self-grade:** Strong answers narrate all six cited files in order.  
**Connects To:** Mission 15 because diff reading is tracing a hypothetical change through the same path.

### Mission 15: Read the Diff Like an Engineer
**Tier:** Mid-Level  
**Time Estimate:** 60 minutes  
**Goal:** Practice reviewing a realistic change across layers.  
**The Concept:** A diff is a proposed route change in the scheduling dispatch system; check every station it passes.  
**Design Intent Before You Read the Code:** Feature changes should update UI, DTO, service, repository, schema, read model, and tests when relevant.  
**Find It In The Code:** Use the schedule-post trace: `manage.modal.tsx:372-425`, `create.post.dto.ts:93-125`, `posts.controller.ts:170-179`, `posts.service.ts:243-292`, `posts.repository.ts:488-630`, `schema.prisma:393-440`.

```text
Review drill:
1. If UI sends a new field, find it in manage.modal.tsx:372-401.
2. If backend accepts it, find validation in create.post.dto.ts:93-125.
3. If service transforms it, find preservation in posts.service.ts:243-292.
4. If database stores it, find write in posts.repository.ts:500-560 and schema in schema.prisma:393-440.
5. If UI later reads it, find select in posts.repository.ts:173-196.
```

**The Aha Moment:** The best review question is "where else must this be true?"  
**Socratic Checkpoint:** What files must change for a persisted post field? What files must change for UI-only display? Where would you add validation? Where would stale calendar data appear? What missing test would you request?  
**How to self-grade:** Strong answers name all layers and one failure mode per layer.  
**Connects To:** Mission 19 because finding future bugs uses the same habit.

### Mission 16: Composition Over Inheritance
**Tier:** Mid-Level  
**Time Estimate:** 45 minutes  
**Goal:** See how the UI composes providers, contexts, and modals.  
**The Concept:** The app assembles a scheduling cockpit from small instruments instead of one giant inheritance tree.  
**Design Intent Before You Read the Code:** Composition should make it possible to wrap children with shared capabilities.  
**Find It In The Code:** Open `apps/frontend/src/app/(app)/layout.tsx:57-117`, `apps/frontend/src/components/new-layout/layout.component.tsx:72-149`, and `apps/frontend/src/components/launches/launches.component.tsx:497-603`.

```tsx
// apps/frontend/src/components/launches/launches.component.tsx:497-603
<DNDProvider> // drag/drop capability
  <Onboarding /> // cross-screen onboarding
  <CalendarWeekProvider integrations={sortedIntegrations}> // calendar data capability
    <div>{/* channel sidebar */}</div>
    <div>
      <Filters />
      <Calendar /> // reads context, does not fetch directly for its base data
    </div>
  </CalendarWeekProvider>
</DNDProvider>
```

**The Aha Moment:** Context providers are how this app gives features shared powers.  
**Socratic Checkpoint:** Which provider supplies fetch? Which provider supplies user? Which provider supplies calendar? Which provider supplies drag/drop? Why is this better than passing all props manually?  
**How to self-grade:** Strong answers cite root layout, new layout, and Launches component wrappers.  
**Connects To:** Mission 18 because composition reveals architectural decisions.

### Mission 17: TypeScript's Hidden Work
**Tier:** Mid-Level  
**Time Estimate:** 50 minutes  
**Goal:** Identify where TypeScript helps and where runtime validation still matters.  
**The Concept:** TypeScript is a route plan; runtime validation checks the actual cargo.  
**Design Intent Before You Read the Code:** Anything crossing HTTP needs runtime validation, not just TypeScript.  
**Find It In The Code:** Open `tsconfig.base.json:22-27`, `apps/backend/tsconfig.build.json:16-20`, `libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts:31-125`, and `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:280-291`.

```ts
// libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:280-291
const validationPipe = new ValidationPipe({
  skipMissingProperties: false, // require missing fields to fail
  transform: true, // convert plain request object into DTO class
  transformOptions: { enableImplicitConversion: true },
});

return await validationPipe.transform(mappedValues, {
  type: 'body',
  metatype: CreatePostDto, // runtime validation target
});
```

**The Aha Moment:** TypeScript disappears at runtime; class-validator is the real request gate.  
**Socratic Checkpoint:** Which config enables strict mode? Which backend build checks are disabled? Which DTO validates post content? Where is the validation pipe created? Why is `any` still dangerous?  
**How to self-grade:** Strong answers cite all four files and distinguish compile-time from runtime.  
**Connects To:** Mission 23 because good tests target runtime behavior.

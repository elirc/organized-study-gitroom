# Tier 3 Senior Missions

### Mission 18: Reverse-Engineer the Architecture Decisions
**Tier:** Senior  
**Time Estimate:** 75 minutes  
**Goal:** Infer why the system is split across apps, libraries, services, and workers.  
**The Concept:** A social scheduler separates dashboard, dispatch, storage, and integrations so one slow delivery does not freeze the desk.  
**Design Intent Before You Read the Code:** Architecture should isolate UI, API, persistence, and publishing side effects.  
**Find It In The Code:** Open `package.json:14-31`, `apps/backend/src/app.module.ts:21-66`, `libraries/nestjs-libraries/src/database/prisma/database.module.ts:46-101`, and `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:791-798`.

```ts
// apps/backend/src/app.module.ts:23-43
imports: [
  DatabaseModule, // persistence services
  ApiModule, // HTTP API
  PublicApiModule, // public/API-key paths
  AgentModule,
  ThirdPartyModule,
  VideoModule,
  ChatModule,
  getTemporalModule(false), // workflow integration
  ThrottlerModule.forRoot({ storage: new ThrottlerStorageRedisService(ioRedis) }), // rate limiting via Redis
]
```

**The Aha Moment:** The module graph says which concerns the app treats as first-class.  
**Socratic Checkpoint:** Why is Temporal imported? Why is database global? Why are public and authenticated APIs separate? Why does root dev run orchestrator? What does Redis support besides app data?  
**How to self-grade:** Strong answers cite module imports and script evidence.  
**Connects To:** Mission 21 because architecture affects performance.

### Mission 19: Find the Bugs Before They Happen
**Tier:** Senior  
**Time Estimate:** 70 minutes  
**Goal:** Predict failure modes from code shape.  
**The Concept:** A senior engineer looks for missed handoffs in the scheduling relay.  
**Design Intent Before You Read the Code:** Bugs often hide where UI state, API validation, and database shape disagree.  
**Find It In The Code:** Open `apps/frontend/src/components/new-launch/manage.modal.tsx:248-425`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:243-292`, and `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:488-630`.

```ts
// libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:617-628
if (body.group) {
  await this._post.model.post.updateMany({
    where: { group: body.group, deletedAt: null },
    data: { parentPostId: null, deletedAt: new Date() }, // old group is soft-deleted after new upserts
  });
}
```

**The Aha Moment:** Update behavior is not just "save"; it creates new rows and retires old rows.  
**Socratic Checkpoint:** What bugs could come from soft-deleting old groups? What bugs could come from missing integration ids? What bugs could come from media JSON shape? What bugs could come from skipped SWR mutate? What bugs could come from provider `__type` mismatch?  
**How to self-grade:** Strong answers name symptoms and cite relevant lines.  
**Connects To:** Mission 20 because bug injection turns predictions into tests.

### Mission 20: The Bug Injection Challenge
**Tier:** Senior  
**Time Estimate:** 60 minutes  
**Goal:** Design bug scenarios without changing production code.  
**The Concept:** You are simulating dispatch failures: wrong time, wrong channel, missing package, duplicate delivery, or no delivery.  
**Design Intent Before You Read the Code:** A useful bug exercise names a user-visible symptom and a reproducible test scenario.  
**Find It In The Code:** Use `calendar.tsx:727-735`, `manage.modal.tsx:248-334`, `posts.service.ts:750-808`, and `posts.repository.ts:129-220`.

```text
Bug drill:
- Symptom: list view misses recurring posts.
- Evidence: calendar query expands interval posts in posts.repository.ts:199-219.
- Counter-evidence: list query filters intervalInDays to null in posts.repository.ts:242-245.
- Test: create recurring post, switch from week to list, compare visibility.
```

**The Aha Moment:** A great bug report ties a symptom to two conflicting code paths.  
**Socratic Checkpoint:** Which bug would affect only list view? Which bug would affect only draft posts? Which bug would affect only link-heavy posts? Which bug would affect only super admins? Which bug would affect only large date ranges?  
**How to self-grade:** Strong answers include symptom, code evidence, and test scenario for each.  
**Connects To:** Mission 23 because every bug scenario can become a test.

### Mission 21: Performance X-Ray
**Tier:** Senior  
**Time Estimate:** 75 minutes  
**Goal:** Spot performance risk in data shape, revalidation, and rendering.  
**The Concept:** A scheduling board can get slow from too many deliveries, too many refreshes, or too many widgets per delivery.  
**Design Intent Before You Read the Code:** Performance review starts at loops, queries, cache keys, and large components.  
**Find It In The Code:** Open `calendar.context.tsx:201-231`, `posts.repository.ts:129-220`, and `calendar.tsx:973-1178`.

```ts
// libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:199-219
return list.reduce((all, post) => {
  if (!post.intervalInDays) return [...all, post]; // non-recurring post
  const addMorePosts = [];
  let startingDate = dayjs.utc(post.publishDate);
  while (dayjs.utc(endDate).isSameOrAfter(startingDate)) { // can expand many occurrences
    addMorePosts.push({ ...post, publishDate: startingDate.toDate(), actualDate: post.publishDate });
    startingDate = startingDate.add(post.intervalInDays, 'days');
  }
  return [...all, ...addMorePosts];
}, []);
```

**The Aha Moment:** Performance bugs often come from reasonable logic meeting unreasonable data volume.  
**Socratic Checkpoint:** Which SWR options reduce refresh churn? Which loop can grow by date range? Which component repeats per post? What index supports date queries? What cap would you add first?  
**How to self-grade:** Strong answers cite SWR options, recurrence loop, `CalendarItem`, and `schema.prisma:431-440`.  
**Connects To:** Mission 18 because architecture determines scaling levers.

### Mission 22: The Security Audit
**Tier:** Senior  
**Time Estimate:** 75 minutes  
**Goal:** Review auth, org isolation, logging, cookies, and public/private boundaries.  
**The Concept:** A scheduler protects unpublished posts like embargoed announcements.  
**Design Intent Before You Read the Code:** Security review asks who can do what, in which org, with what data exposed.  
**Find It In The Code:** Open `auth.middleware.ts:33-104`, `custom.fetch.func.ts:22-74`, `posts.controller.ts:170-179`, and `posts.repository.ts:503-507`.

```ts
// apps/backend/src/api/routes/posts.controller.ts:170-179
@Post('/')
@CheckPolicies([AuthorizationActions.Create, Sections.POSTS_PER_MONTH]) // authorization
async createPost(@GetOrgFromRequest() org: Organization, @Body() rawBody: any) {
  console.log(JSON.stringify(rawBody, null, 2)); // security concern: draft content in logs
  const body = await this._postsService.mapTypeToPost(rawBody, org.id);
  return this._postsService.createPost(org.id, body, 'WEB');
}
```

**The Aha Moment:** The same line can be operationally helpful and security risky.  
**Socratic Checkpoint:** Where is org isolation enforced? Which line logs sensitive data? Which mode sends auth headers from cookies? Which controller group is authenticated? What should production avoid setting?  
**How to self-grade:** Strong answers cite auth middleware, fetch wrapper, controller log, and repository connect.  
**Connects To:** Mission 23 because security findings need regression tests.

### Mission 23: Write the Test That Doesn't Exist
**Tier:** Senior  
**Time Estimate:** 90 minutes  
**Goal:** Design a focused test for risky behavior.  
**The Concept:** A test is a saved inspection checklist that runs every time.  
**Design Intent Before You Read the Code:** Test behavior at the smallest layer that proves the risk.  
**Find It In The Code:** Open `posts.service.ts:243-292`, `posts.service.ts:750-808`, `posts.repository.ts:488-630`, `jest.config.ts:1-5`, and `package.json:40`.

```ts
// libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:248-266
if (!body?.posts?.every((p) => p?.integration?.id)) {
  throw new BadRequestException('All posts must have an integration id'); // perfect unit-test target
}
const integration = await this._integrationService.getIntegrationById(organization, post.integration.id);
if (!integration) throw new BadRequestException(`Integration with id ${post.integration.id} not found`);
```

**The Aha Moment:** Start tests where the behavior is deterministic and the mocks are small.  
**Socratic Checkpoint:** Which behavior is easiest to unit test? Which dependencies must be mocked? Which test would need a database? Which test protects provider `__type` injection? Which script runs tests?  
**How to self-grade:** Strong answers distinguish unit tests for service mapping from integration tests for repository upsert.  
**Connects To:** Mission 24 because test gaps are part of codebase history.

### Mission 24: The Git History Tells a Story
**Tier:** Senior  
**Time Estimate:** 60 minutes  
**Goal:** Infer evolution from current code structure and realistic commit messages.  
**The Concept:** A codebase is an archive of product pressure: public API, MCP, autopost, billing, and calendar needs left fossils.  
**Design Intent Before You Read the Code:** History reading helps you avoid undoing why something exists.  
**Find It In The Code:** Open `schema.prisma:943-950`, `public.integrations.controller.ts:146-202`, `calendar.context.tsx:187-231`, and `creation.method.badge.tsx:4-50`.

```text
Realistic history clues:
- CreationMethod enum includes WEB, MCP, API, AUTOPOST, CLI in schema.prisma:943-950.
- Public API has /public/v1/posts endpoints in public.integrations.controller.ts:146-202.
- UI exposes creation method badges in creation.method.badge.tsx:4-50.
```

**The Aha Moment:** When the schema has an enum, the product has a story.  
**Socratic Checkpoint:** What feature likely introduced `MCP`? What feature likely introduced list view? Why does public API need its own module? Why display creation method only to impersonating users? What commit would you search for first?  
**How to self-grade:** Strong answers tie each guess to a file and line range, not just product intuition.  
**Connects To:** Mission 25 because documenting history is part of ownership.

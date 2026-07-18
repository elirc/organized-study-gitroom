# Senior Engineer Guide

## Architectural Critique Scores

- Scalability: 3/5. The app separates frontend, backend, orchestrator, and extension at the script/app level (`package.json:14-31`), and publishing work is handed to Temporal (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:791-798`). The broad global `DatabaseModule` provider list is convenient but becomes harder to reason about as domains grow (`libraries/nestjs-libraries/src/database/prisma/database.module.ts:46-101`).
- TypeScript discipline: 2/5. Root and frontend strictness are enabled (`tsconfig.base.json:22-27`, `apps/frontend/tsconfig.json:3-19`), but backend build disables important checks (`apps/backend/tsconfig.build.json:16-20`) and key frontend/backend paths use `any` or `@ts-ignore` (`apps/backend/src/api/routes/posts.controller.ts:170-179`, `apps/frontend/src/components/new-launch/manage.modal.tsx:248-401`).
- Separation of concerns: 3/5. Controllers delegate well (`apps/backend/src/api/routes/posts.controller.ts:111-179`), but `PostsService.createPost` combines provider lookup, shortlinking, persistence, workflow start, and metrics (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:750-808`).
- Testability: 2/5. Jest is configured (`jest.config.ts:1-5`, `package.json:40`), but no test files were found by `*.spec.*` or `*.test.*` search. The create-post path has enough branching to deserve tests (`apps/frontend/src/components/new-launch/manage.modal.tsx:248-425`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:750-808`).
- Maintainability: 3/5. Feature folders are readable, but large files like `calendar.tsx` and `manage.modal.tsx` contain many concerns (`apps/frontend/src/components/launches/calendar.tsx:1-1318`, `apps/frontend/src/components/new-launch/manage.modal.tsx:1-712`).
- Security posture: 3/5. JWT/org middleware exists (`apps/backend/src/services/auth/auth.middleware.ts:33-104`) and policies protect post creation (`apps/backend/src/api/routes/posts.controller.ts:170-172`), but full payload logging and non-secured cookie/header paths deserve scrutiny (`apps/backend/src/api/routes/posts.controller.ts:176-178`, `libraries/helpers/src/utils/custom.fetch.func.ts:22-44`).
- Performance: 3/5. SWR disables noisy focus revalidation for major calendar reads (`apps/frontend/src/components/launches/calendar.context.tsx:201-231`), but some queries can grow with date range and recurring posts (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:129-220`).

## Performance Audit

Finding 1: recurring posts are expanded in memory after fetching any post with `intervalInDays` in the date window query (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:129-220`). If a user has many recurring posts, this loop can create large arrays per calendar request.

Corrected direction:

```ts
// Instead of unbounded recurrence expansion inside getPosts, cap generated occurrences.
// Existing expansion location: libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:199-219
const maxOccurrencesPerRecurringPost = 366; // one year of daily repeats
while (
  dayjs.utc(endDate).isSameOrAfter(startingDate) &&
  addMorePosts.length < maxOccurrencesPerRecurringPost
) {
  // preserve existing push behavior, but prevent pathological ranges
}
```

Finding 2: `CalendarItem` creates several hover-only controls and uses drag state per post (`apps/frontend/src/components/launches/calendar.tsx:973-1178`). This is reasonable for modest calendars, but large calendar views can pay component cost. Keep `memo` and avoid passing freshly-created callbacks where possible; the file already uses `memo` at the component declaration (`apps/frontend/src/components/launches/calendar.tsx:992-1031`).

## Security Audit

Finding 1: `createPost` logs the full raw body, which can include unpublished social content and media metadata (`apps/backend/src/api/routes/posts.controller.ts:170-179`).

Corrected code:

```ts
// apps/backend/src/api/routes/posts.controller.ts:170-179
async createPost(@GetOrgFromRequest() org: Organization, @Body() rawBody: any) {
  // Log shape, not content. Useful during incidents without storing drafts.
  Logger.debug({
    postCount: rawBody?.posts?.length ?? 0,
    type: rawBody?.type,
    orgId: org.id,
  }, 'Creating post');
  const body = await this._postsService.mapTypeToPost(rawBody, org.id);
  return this._postsService.createPost(org.id, body, 'WEB');
}
```

Finding 2: non-secured mode reads auth-related values from document cookies and sends them as headers (`libraries/helpers/src/utils/custom.fetch.func.ts:22-44`, `libraries/helpers/src/utils/custom.fetch.func.ts:59-68`). This supports local/self-hosted scenarios, but production should keep `NOT_SECURED` unset so cookies use secure/httpOnly/sameSite behavior in `AuthMiddleware.removeAuth` (`apps/backend/src/services/auth/auth.middleware.ts:11-24`).

## TypeScript Discipline Review

High-value fixes:

- Replace `rawBody: any` with a narrow pre-DTO type or pipe at controller level (`apps/backend/src/api/routes/posts.controller.ts:170-179`).
- Replace `settings: any` in composer store with a provider settings union or `unknown` plus guards (`apps/frontend/src/components/new-launch/store.ts:22-26`).
- Remove `@ts-ignore` in layout/user and store flows after modeling response types (`apps/frontend/src/components/new-layout/layout.component.tsx:52-73`, `apps/frontend/src/components/new-launch/store.ts:160-654`).

## Custom Abstractions Inventory

- `customFetch` and `FetchWrapperComponent`: centralize base URL, credentials, headers, cache policy, and after-request hooks (`libraries/helpers/src/utils/custom.fetch.func.ts:1-92`, `libraries/helpers/src/utils/custom.fetch.tsx:27-46`).
- `CalendarWeekProvider` and `useCalendar`: page-level read model and mutation triggers (`apps/frontend/src/components/launches/calendar.context.tsx:29-81`, `apps/frontend/src/components/launches/calendar.context.tsx:134-351`).
- `useLaunchStore`: modal-level write state (`apps/frontend/src/components/new-launch/store.ts:160-654`).
- `GetOrgFromRequest` and `GetUserFromRequest`: Nest decorators for request context (`libraries/nestjs-libraries/src/user/org.from.request.ts:1-8`, `libraries/nestjs-libraries/src/user/user.from.request.ts:1-8`).
- `PrismaRepository<T>`: thin typed holder around `PrismaService` (`libraries/nestjs-libraries/src/database/prisma/prisma.service.ts:25-31`).

## Testing Assessment

Risky untested behavior: `PostsService.mapTypeToPost` must reject missing integration ids and inject provider `__type` before DTO validation (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:243-292`). A complete runnable test would require Nest testing utilities already installed in root dev dependencies (`package.json` dependencies include `@nestjs/testing` in the devDependency section from inspection) and a mocked `IntegrationService`.

Proposed test file:

```ts
// libraries/nestjs-libraries/src/database/prisma/posts/posts.service.spec.ts
import { BadRequestException } from '@nestjs/common';
import { PostsService } from './posts.service';

describe('PostsService.mapTypeToPost', () => {
  it('rejects posts without integration ids', async () => {
    const service = new PostsService({} as any, {} as any, {} as any, {} as any, {} as any, {} as any, {} as any, {} as any, {} as any);
    await expect(service.mapTypeToPost({ type: 'schedule', posts: [{}] } as any, 'org_1')).rejects.toBeInstanceOf(BadRequestException);
  });
});
```

This is intentionally not added to production code here; the suite is documentation-only.

## Bug Injection Exercise

1. Symptom: dragging a draft post to a new date appears to move it, but after refresh it returns to draft state. Test scenario: drag a draft in `CalendarColumn` and inspect `PUT /posts/:id/date` action value (`apps/frontend/src/components/launches/calendar.tsx:727-735`, `apps/backend/src/api/routes/posts.controller.ts:213-221`).
2. Symptom: scheduled thread loses child order after deleting a middle child. Test scenario: inspect store delete behavior preserving ids (`apps/frontend/src/components/new-launch/store.ts:258-300`) and repository parent chain creation (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:500-560`).
3. Symptom: shortlinks are created for a platform that strips links. Test scenario: compare UI filter for `stripLinks` (`apps/frontend/src/components/new-launch/manage.modal.tsx:340-350`) with service provider strip behavior (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:756-775`).
4. Symptom: list view misses recurring posts that calendar view shows. Test scenario: compare `/posts` recurrence expansion (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:199-219`) with `/posts/list` filter excluding `intervalInDays` (`libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:222-260`).
5. Symptom: user gets logged out after a harmless failed billing request. Test scenario: inspect `afterRequest` 401/logout handling and 402 handling (`apps/frontend/src/components/layout/layout.context.tsx:83-120`).

## Git History Learning Exercise

1. `feat(calendar): add list view backed by paginated posts endpoint`
   - Implies `calendar.context.tsx` grew a second SWR key and `PostsController` gained `/posts/list` (`apps/frontend/src/components/launches/calendar.context.tsx:187-231`, `apps/backend/src/api/routes/posts.controller.ts:132-138`).
2. `feat(posts): track creation method for web/api/mcp/autopost/cli`
   - Implies schema enum and UI badge arrived together (`libraries/nestjs-libraries/src/database/prisma/schema.prisma:943-950`, `apps/frontend/src/components/launches/creation.method.badge.tsx:4-50`).
3. `fix(auth): preserve org selection through non-secured local mode`
   - Implies `customFetch` and `AuthMiddleware` both handle `showorg` headers/cookies (`libraries/helpers/src/utils/custom.fetch.func.ts:30-54`, `apps/backend/src/services/auth/auth.middleware.ts:40-83`).
4. `perf(calendar): minify posts response for calendar grid`
   - Implies service returns `minifyPosts` for `/posts` (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:324-337`).
5. `feat(posts): start temporal workflow after schedule`
   - Implies create service starts workflow after repository persistence (`libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:777-798`).

## If I Owned This Codebase

| Refactor | Effort | Impact | Evidence |
|---|---:|---:|---|
| Move `POST /posts` DTO validation to controller and remove `rawBody: any` | M | High | `apps/backend/src/api/routes/posts.controller.ts:170-179`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:280-291` |
| Split `ManageModal` into validation, payload builder, and view components | L | High | `apps/frontend/src/components/new-launch/manage.modal.tsx:202-698` |
| Add unit tests for map/create/update post behavior | M | High | `libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts:243-292`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:488-630` |
| Replace stringified media/settings JSON with typed Prisma JSON fields if migration is acceptable | L | Medium | `libraries/nestjs-libraries/src/database/prisma/schema.prisma:407-408`, `libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts:536-537` |
| Create explicit API response types for calendar posts instead of importing Prisma client types in UI | M | Medium | `apps/frontend/src/components/launches/calendar.context.tsx:17-25`, `apps/frontend/src/components/launches/calendar.context.tsx:44-75` |

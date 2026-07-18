# Writing Tests Here

Eight recipes. Commands are `inferred` (jest config is currently broken — recipe 0 fixes it). All example test code is **illustrative and labeled**; adapt to the real signatures by opening the anchor.

## Recipe 0: make the runner work (do first)

Create `libraries/nestjs-libraries/jest.config.ts` (illustrative):
```ts
// Illustrative fake code: not from this repo
export default {
  displayName: 'nestjs-libraries',
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src'],
  moduleNameMapper: { '^@gitroom/(.*)$': '<rootDir>/../../libraries/$1/src' }, // match tsconfig paths
};
```
Run: `pnpm --filter ./libraries/nestjs-libraries exec jest` (`inferred`). See ticket J17 for the full wiring.

## Recipe 1: happy path — a pure codec round-trip
Anchor: [posts.list.minify.ts](../../../libraries/helpers/src/utils/posts.list.minify.ts).
```ts
// Illustrative fake code: not from this repo
it('expand(minify(x)) preserves fields the UI reads', () => {
  const posts = [{ id:'1', content:'hi', publishDate:new Date('2026-01-01'), state:'QUEUE', integration:{id:'i1'} }];
  expect(expandPosts(minifyPosts({ posts }))).toMatchObject({ /* the fields calendar uses */ });
});
```
Teaches: pure-function testing, no infra.

## Recipe 2: validation failure — DTO rejects bad input
Anchor: [CreatePostDto](../../../libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts#L93-L125), [mapTypeToPost](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L243-L292).
```ts
// Illustrative fake code: not from this repo
it('rejects a post with no integration id', async () => {
  await expect(service.mapTypeToPost({ posts:[{ value:[{content:'x'}] }] } as any, 'org1'))
    .rejects.toThrow('All posts must have an integration id');   // real message L249
});
```
Teaches: asserting on the manual-validation path; mock `_integrationService.getIntegrationById`.

## Recipe 3: permission failure — quota gate denies
Anchor: [PermissionsService.check](../../../apps/backend/src/services/auth/permissions/permissions.service.ts#L39-L172).
```ts
// Illustrative fake code: not from this repo
it('denies POSTS_PER_MONTH when count >= plan', async () => {
  subs.getSubscription.mockResolvedValue({ createdAt:new Date(), subscriptionTier:'STANDARD' });
  posts.countPostsFromDay.mockResolvedValue(400);   // STANDARD cap
  const ability = await svc.check('org1', new Date(), 'USER', [[Create, POSTS_PER_MONTH]]);
  expect(ability.can(Create, POSTS_PER_MONTH)).toBe(false);
});
```
Teaches: `jest-mock-extended` for the injected services; and set `STRIPE_PUBLISHABLE_KEY` in the test env or the allow-all short-circuit ([L50-L63](../../../apps/backend/src/services/auth/permissions/permissions.service.ts#L50-L63)) makes every assertion pass — a subtle test-env gotcha worth learning.

## Recipe 4: cross-tenant rejection (the bug-fix test)
Anchor: [editTag](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L804-L814) — write this test **red**, then fix editTag (J2) to make it green.
```ts
// Illustrative fake code: not from this repo
it('does not rename a tag from another org', async () => {
  const tagA = await repo.createTag('orgA', { name:'x', color:'#000' });
  await expect(repo.editTag(tagA.id, 'orgB', { name:'hacked', color:'#fff' })).rejects.toThrow(); // or assert no-op
  expect((await repo.getTags('orgA'))[0].name).toBe('x');
});
```
Teaches: integration test against a real (test) DB; the red-green-with-fix workflow.

## Recipe 5: async side effect — workflow start is invoked
Anchor: [createPost → startWorkflow](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L750-L808).
```ts
// Illustrative fake code: not from this repo
it('starts a workflow for scheduled posts but not for updates', async () => {
  const spy = jest.spyOn(service, 'startWorkflow').mockResolvedValue();
  await service.createPost('org1', { type:'schedule', /*…*/ } as any, 'WEB');
  expect(spy).toHaveBeenCalled();
  spy.mockClear();
  await service.createPost('org1', { type:'update', /*…*/ } as any, 'WEB');
  expect(spy).not.toHaveBeenCalled();   // update branch skips it (L791)
});
```
Teaches: verifying orchestration decisions without real Temporal; the update-skips-workflow invariant.

## Recipe 6: cache invalidation — analytics cached by date
Anchor: [checkPostAnalytics Redis set/get](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L205-L228).
```ts
// Illustrative fake code: not from this repo
it('caches analytics under a date-scoped key', async () => {
  provider.postAnalytics.mockResolvedValue([{ label:'likes', data:[] }]);
  await service.checkPostAnalytics('org1','post1', 7);
  expect(redis.set).toHaveBeenCalledWith(expect.stringContaining(':post1:7'), expect.any(String), 'EX', 3600);
});
```
Teaches: asserting cache keys encode all inputs (ties to fake-code contrast #4); mock `ioRedis`.

## Recipe 7: workflow logic — refresh-then-retry
Anchor: [postWorkflowV105 refresh branch](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L219-L237), using `@temporalio/testing`.
```ts
// Illustrative fake code: not from this repo
it('refreshes token then retries postSocial', async () => {
  const acts = { postSocial: jest.fn()
      .mockRejectedValueOnce(refreshTokenActivityFailure())
      .mockResolvedValueOnce([{ postId:'1', releaseURL:'u' }]),
    refreshTokenWithCause: jest.fn().mockResolvedValue({ accessToken:'new' }),
    /* …other activities stubbed… */ };
  const result = await testEnv.executeWorkflow(postWorkflowV105, { activities: acts, args:[/*…*/] });
  expect(acts.refreshTokenWithCause).toHaveBeenCalled();
  expect(acts.postSocial).toHaveBeenCalledTimes(2);
});
```
Teaches: the Temporal test framework, mocking activities, asserting retry/refresh control flow — the highest-value test in the repo.

## Recipe 8: UI state — composer resets on unmount
Anchor: [add.edit.modal reset on unmount](../../../apps/frontend/src/components/new-launch/add.edit.modal.tsx#L209-L212), [store.ts](../../../apps/frontend/src/components/new-launch/store.ts).
```ts
// Illustrative fake code: not from this repo
it('clears composer state when the modal unmounts', () => {
  const { unmount } = render(<AddEditModal {...props} />);   // @testing-library/react
  act(() => useLaunchStore.getState().addGlobalValue(0, [{ content:'x', id:'1', media:[] }]));
  unmount();
  expect(useLaunchStore.getState().global).toHaveLength(0);
});
```
Teaches: testing zustand + React with Testing Library (both are deps); guards against state bleed between modal opens.

---

Command summary (all `inferred`):
- Run one package: `pnpm --filter ./libraries/nestjs-libraries exec jest` 
- Watch: `... exec jest --watch`
- One file: `... exec jest posts.list.minify`
- Coverage: `... exec jest --coverage`

Progression: recipes 1 → 2 → 3 need no DB (do in an hour). 4 needs test Postgres. 7 needs the Temporal test env. 8 needs jsdom (`jest-environment-jsdom`, already a dep). Start with 1–3; each later recipe buys down a specific risk from the [risk register](../09-reference/risk-register.md).

# Fake-Code Contrasts

Ten bad-vs-better pairs. **All code blocks are illustrative and labeled — they are not from this repo** — but each is tied to a real pattern here so you learn to spot the shape. Read the "novice instinct" and "why better," then find the real anchor.

---

## 1. Coupling UI shape to DB shape

```ts
// Illustrative fake code: not from this repo
// Bad — controller returns the raw Prisma row; frontend now depends on column names
@Get('/:id') getPost(@Param('id') id) { return this.repo.findUnique({ where: { id } }); }
```
```ts
// Illustrative fake code: not from this repo
// Better — map to an explicit response shape the client owns
return toPostResponse(await this.repo.getPost(orgId, id)); // stable contract
```
Novice instinct: "just return the row." Why better: the DB is free to change without breaking clients. Real anchor: this repo *does* shape responses — [minifyPosts](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L328-L338) and the hand-built object in [getPost L529-L544](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L527-L547) — a good habit to emulate.

## 2. Missing permission/org filter (IDOR)

```ts
// Illustrative fake code: not from this repo
// Bad — id comes from the URL, no tenant scope
editTag(id, orgId, body) { return this.tags.update({ where: { id }, data: body }); }
```
```ts
// Illustrative fake code: not from this repo
// Better — scope by org (use updateMany or findFirst+update for non-unique compound)
async editTag(id, orgId, body) {
  const tag = await this.tags.findFirst({ where: { id, orgId } });
  if (!tag) throw new ForbiddenException();
  return this.tags.update({ where: { id }, data: body });
}
```
Real anchor: the bad version is essentially [editTag L804-L814](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L804-L814); the good pattern is its sibling [deleteTag L816-L826](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L816-L826). Ticket J2.

## 3. N+1 / per-item async in a loop

```ts
// Illustrative fake code: not from this repo
// Bad — one query per channel, serially
for (const ch of channels) ch.meta = await getProviderMeta(ch.identifier);
```
```ts
// Illustrative fake code: not from this repo
// Better — batch, or parallel with a concurrency cap
await Promise.all(channels.map(ch => limit(() => getProviderMeta(ch.identifier))));
```
Real anchor: [getIntegrationList](../../../apps/backend/src/api/routes/integrations.controller.ts#L88-L124) maps every channel through async metadata (parallel, but uncapped); the recursive [getPostsRecursively](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L294-L322) is one-query-per-thread-item. `p-limit` is already a dependency.

## 4. Stale cache key

```ts
// Illustrative fake code: not from this repo
// Bad — key ignores the customer filter; switching customers serves the wrong data
useSWR('posts', () => fetchPosts(customer));
```
```ts
// Illustrative fake code: not from this repo
// Better — every input that changes the result is in the key
useSWR(['posts', startDate, endDate, customer], () => fetchPosts(customer, startDate, endDate));
```
Real anchor: the calendar's SWR keys must encode date range + customer ([calendar.context.tsx](../../../apps/frontend/src/components/launches/calendar.context.tsx#L1-L50)); the analytics cache key *does* include date ([checkPostAnalytics Redis key L212-L213](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L205-L219)) — good example.

## 5. Side effect in an unreliable place (retried unit)

```ts
// Illustrative fake code: not from this repo
// Bad — fetch inside a Temporal activity that Temporal will retry → duplicate deliveries
async sendWebhook(url, body) { await fetch(url, { method:'POST', body }); }
```
```ts
// Illustrative fake code: not from this repo
// Better — include a delivery id; receivers dedupe; record the attempt
await fetch(url, { method:'POST', headers:{ 'X-Delivery-Id': id, 'X-Signature': sign(body) }, body });
await this.deliveries.record(id, res.status);
```
Real anchor: [sendWebhooks L321-L335](../../../apps/orchestrator/src/activities/post.activity.ts#L310-L336) — no id, no signature, no record; runs inside a retried activity.

## 6. Swallowed error

```ts
// Illustrative fake code: not from this repo
// Bad — any failure returns the un-processed input; the real error vanishes
try { return await process(x); } catch { return x; }
```
```ts
// Illustrative fake code: not from this repo
// Better — narrow what you tolerate; surface the rest
try { return await process(x); }
catch (e) { if (isRecoverable(e)) { logger.warn(e); return x; } throw e; }
```
Real anchor: [updateMedia catch L434-L436](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L434-L436) returns the unresolved list on *any* error; the workflow-start `.catch(() => {})` ([posts.service L797](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L790-L798)) — at least that one has a compensating sweep.

## 7. `any` at the boundary

```ts
// Illustrative fake code: not from this repo
// Bad — any disables checking; validation must be re-done by hand elsewhere
@Post('/') create(@Body() body: any) { return this.svc.create(body); }
```
```ts
// Illustrative fake code: not from this repo
// Better — a DTO the global ValidationPipe can enforce at the door
@Post('/') create(@Body() body: CreatePostDto) { return this.svc.create(body); }
```
Real anchor: [posts.controller createPost `@Body() rawBody: any`](../../../apps/backend/src/api/routes/posts.controller.ts#L170-L179) → forced manual validation in [mapTypeToPost](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L280-L291).

## 8. Casual public-contract change

```ts
// Illustrative fake code: not from this repo
// Bad — renaming an activity method silently breaks running Temporal workflows
@ActivityMethod() async publishPost(...) {}   // was postSocial
```
```ts
// Illustrative fake code: not from this repo
// Better — keep the old name as an alias, or version the workflow that calls it
@ActivityMethod() async postSocial(...) {}     // running workflows pin this name
```
Real anchor: the [proxyActivities<PostActivity>](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L31-L46) contract; the five kept [workflow versions](../../../apps/orchestrator/src/workflows/post-workflows) exist precisely because you can't casually change what running executions depend on.

## 9. Non-idempotent write under retry

```ts
// Illustrative fake code: not from this repo
// Bad — retry re-posts; provider has no dedupe
await provider.post(content);           // attempt 1 tweeted, then timed out → attempt 2 tweets again
```
```ts
// Illustrative fake code: not from this repo
// Better — record the remote id the instant it returns; check on retry entry
if (post.releaseId) return post.releaseId;      // already published
const id = await provider.post(content);
await repo.updatePost(post.id, id);
```
Real anchor: [workflow retry loop L164-L200](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L164-L200) posts then `updatePost`; the gap between them is where duplicates are born (debugging scenario 2).

## 10. Missing transaction on multi-row invariant

```ts
// Illustrative fake code: not from this repo
// Bad — a crash mid-loop leaves half a thread and a half-deleted group
for (const item of items) await repo.upsert(item);
await repo.softDeleteOldGroup(group);
```
```ts
// Illustrative fake code: not from this repo
// Better — one transaction; all-or-nothing
await prisma.$transaction(async tx => {
  for (const item of items) await tx.upsert(item);
  await tx.softDeleteOldGroup(group);
});
```
Real anchor: [createOrUpdatePost L488-L631](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L488-L631) — the exact bad shape; senior project S2 fixes it.

---

Self-grade — Strong: for ≥8 pairs you found the real anchor unaided *and* can state the one-sentence rule (e.g., "no side effects in retried units without an idempotency key"). Those rules are your review checklist for [04-review-katas.md](04-review-katas.md).

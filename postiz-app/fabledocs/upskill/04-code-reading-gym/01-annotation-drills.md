# Annotation Drills

Ten real excerpts. For each: open the file, and annotate **inputs, outputs, dependencies, invariants, side effects, failure modes** before reading the "what to notice" notes. Grade with the rubric at the bottom.

---

## Drill 1: `AuthMiddleware.use` — [auth.middleware.ts L33-L104](../../../apps/backend/src/services/auth/auth.middleware.ts#L33-L104)

Annotate first. Then check yourself:
- Inputs: `auth` cookie **or header** (two transports); `showorg`, `impersonate` cookies/headers.
- Invariant it must uphold: `req.org` is an org the user actually belongs to and isn't disabled in. Verify: does the impersonation branch uphold it? (It swaps in the *target's* org — by design.)
- Side effects: may write an org apiKey (L89-L91) — a **write inside auth middleware**, on the read path.
- Failure modes: empty org list → `organization[0]` is undefined → `.apiKey` throws → caught → 403 with no cause; the `!organization` check (L85) can never fire (arrays are truthy).
- Depth check: why `delete user.password` twice (L59, L78)?

## Drill 2: `createOrUpdatePost` — [posts.repository.ts L488-L631](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L488-L631)

- Inputs: `state` (4 modes), org, date, `body.value[]` (thread items), tags, optional `body.group`.
- Invariants *intended*: one `group` uuid per thread; each item's `parentPost` = previous item; tags attach to the first item only (check: the code attaches to `posts[posts.length - 1]` inside the `posts.length === 1` branch — first item, both expressions equal — why does it read strangely?).
- Side effects: N upserts + tag delete/create + old-group soft-delete — **no transaction**.
- Failure modes: crash mid-loop → half a thread, old group already/not-yet deleted; `value.id || uuidv4()` in the upsert `where` means a *random* uuid that can't match → always creates (idempotency accident that works).
- Depth: why does the update path `disconnect` `lastMessage` and `submittedForOrder`?

## Drill 3: The retry core — [post.workflow.v1.0.5.ts L164-L269](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L164-L269)

- Inputs: `postsList` (slimmed posts), `post.integration` (mutable — token gets swapped at L235).
- Invariant: `postsResults[i]` corresponds to `postsList[i]` — check the comment-chain indexing at L183-L191 (`postsResults[i-1].postId` as reply target).
- Failure modes: if `postSocial` succeeds but `updatePost` throws, the catch marks the whole post ERROR *after it published* (published-but-ERROR state); the `iterate` retry re-posts on non-typed errors → duplicate posts on the platform.
- Depth: `before` counter (L165, L265) — what exactly does it detect?

## Drill 4: `SocialAbstract.fetch` — [social.abstract.ts L101-L173](../../../libraries/nestjs-libraries/src/integrations/social.abstract.ts#L101-L173)

- Inputs: url/options/identifier/totalRetries.
- Outputs: `Response` only on 200/201 — a 204 or 302 becomes an error (invariant assumption about provider APIs).
- Failure taxonomy: 429/rate-strings/500-without-mapping → recursive retry (max 3, fixed 5s); 401 or mapped refresh → `RefreshToken`; else `BadBody`.
- Trap: `handleErrors(json, status)` is called with the *response body*; but in `runInConcurrent` (L74-L99) it's called with a stringified *exception* and status literal 200 — same function, two meanings.
- Depth: why is the retry not exponential? What load does fixed-5s create during a provider outage across thousands of workflows?

## Drill 5: `PermissionsService.check` — [permissions.service.ts L39-L172](../../../apps/backend/src/services/auth/permissions/permissions.service.ts#L39-L172)

- Inputs: orgId, org createdAt, role, requested `[action, section]` pairs, optional refresh channel.
- Dependencies: subscription, integrations list, webhook count, post count — **up to 4 queries per gated request**.
- Invariant: self-hosted (no Stripe key) = everything allowed.
- Failure modes: `POSTS_PER_MONTH` window anchors on subscription createdAt drifting monthly (L106-L122) — off-by-one at month boundaries? Work an example.
- Depth: `channel: -10` for paid tiers (L34) — find what consumes a negative channel count and why.

## Drill 6: `connectSocialMedia` — [no.auth.integrations.controller.ts L42-L307](../../../apps/backend/src/api/routes/no.auth.integrations.controller.ts#L42-L307)

- Inputs: provider name (path), `{state, code, refresh, timezone}` body — **no auth cookie**.
- Trust chain: state → Redis → orgId. Annotate every `ioRedis.get/del` and whether the key is single-use.
- Invariants: refresh must reconnect the *same* account (L186-L190); trial orgs can't reconnect previously-seen channels (L201-L210).
- Side effects: integration upsert, refresh workflow start, optional webhook POST to a stored URL, extension JWT minting.
- Failure modes: what if two callbacks race with the same state? (First deletes the keys; second fails at which line?)

## Drill 7: `getPosts` + interval expansion — [posts.repository.ts L129-L220](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L129-L220)

- Inputs: org, `startDate/endDate` (`customer` optional).
- Output shape: rows *plus synthetic rows* for recurring posts (`actualDate` marks the real row).
- Failure modes: a recurring post with `publishDate` far in the past generates occurrences from that date forward in a `while` loop — how many iterations for a 2-year-old daily post? Where's the cap? (There isn't one.)
- Depth: the `AND/OR` where-clause — why is `organizationId` inside an OR of one element? (Archaeology: something was removed. What might it have been?)

## Drill 8: `updateMedia` — [posts.service.ts L340-L437](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L340-L437)

- Inputs: post id, `imagesList` (possibly ids-only), convertToJPEG flag.
- Pipeline: resolve id→Media, prefix path/url, optional PNG→JPEG via sharp + re-upload.
- Side effects: uploads a *new* file; writes the resolved list back onto the post when anything changed.
- Failure modes: whole function wrapped in `try { … } catch { return imagesList }` — any error returns the **unresolved input**, deferring the explosion to the provider call with a worse error message.
- Depth: `process.env.FRONTEND_URL + '/' + NEXT_PUBLIC_UPLOAD_STATIC_DIRECTORY + m.path` — track how many env vars must agree for media URLs to work.

## Drill 9: The composer store initialization — [add.edit.modal.tsx L36-L107](../../../apps/frontend/src/components/new-launch/add.edit.modal.tsx#L36-L107)

- Inputs: props (`date`, `integrations`, `set`, `selectedChannels`, `onlyValues`…), zustand store, `useExistingData()` (URL-driven edit state).
- Invariant: three nested components each render only when their store slice is ready (`integrations.length`, `selectedIntegrations`, `global/internal`) — a **staged initialization gate**.
- Failure modes: all init `useEffect`s have `[]` deps — prop changes after mount are ignored; a `set` prop arriving late does nothing.
- Depth: why three components instead of one with conditions? (Each stage's hooks must not run until the previous stage committed — the rules of hooks shape the architecture.)

## Drill 10: `changeState` + error record — [posts.repository.ts L393-L428](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L393-L428)

- Inputs: post id (unscoped — trusted internal caller), state, optional err/body.
- Side effects: state write, `Errors` row (best-effort, its own try/catch).
- Invariant *not* enforced: legal state transitions — nothing stops `PUBLISHED → QUEUE`.
- Depth: `error: typeof err === 'string' ? err : JSON.stringify(err)` — what does `JSON.stringify(new Error('x'))` produce? (`{}` — Error props aren't enumerable. Where else does this bite? Check the workflow's `changeState(id, 'ERROR', err, …)` calls.)

---

## Self-grading rubric (per drill)

- **Basic**: you correctly listed inputs/outputs and one side effect. You can say what the code does.
- **Solid**: you identified the invariant the code protects (or fails to), traced one failure path to its user-visible symptom, and named every external dependency.
- **Strong**: you found something not listed in the notes above, *and* can state the smallest change that removes the sharpest failure mode, *and* can say what test would lock it in. Strong on 7+ drills ≈ ready for mid-level code-reading rounds.

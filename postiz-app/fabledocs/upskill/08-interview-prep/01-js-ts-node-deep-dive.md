# JS / TS / Node Deep-Dive Question Cards

Fourteen cards. Format: what interviewers ask, what they're actually testing, where this repo exercises it, and the junior→mid→senior answer ladder. Practice: answer aloud in ~90 seconds, *leading with the repo example*.

---

## Q1: "What does `await` actually do? Walk me through this loop."

Round: JS/TS deep-dive
What it's really testing: event-loop model — microtasks, suspension points, serial vs parallel.
Repo anchor: [posts.service.ts createPost L755-L806](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L750-L808) — a `for...of` with `await` per post (serial, order-preserving), vs [updateMedia L343-L424](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L340-L437) using `Promise.all` over maps (parallel).
Junior answer sounds like: "await pauses until the promise resolves."
Mid-level adds: it suspends *this function*, yielding to the event loop; a `for await` loop serializes I/O — good when each iteration depends on the previous DB row (the thread-chaining upsert at [posts.repository.ts L500-L561](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L500-L561) *must* be serial because each row connects to the previous one's id), wasteful when iterations are independent.
Senior includes: unbounded `Promise.all` is its own failure mode (N provider calls at once → rate limits); mention `p-limit` (it's in this repo's deps) and backpressure.
Likely follow-ups: "microtask vs macrotask?", "what happens to the loop if one await rejects?"
Practice drill: point at the two anchors and say which you'd flip, and why neither is "always right".

## Q2: "Why is there a `new Promise(async (res) => …)` here, and what's wrong with it?"

Round: JS/TS deep-dive / code review
What it's really testing: promise-executor semantics, error propagation.
Repo anchor: [no.auth.integrations.controller.ts L106-L176](../../../apps/backend/src/api/routes/no.auth.integrations.controller.ts#L96-L176) — eslint-disabled `no-async-promise-executor`.
Junior: "it wraps the auth call."
Mid: an async executor's rejection is **not** captured by the promise — a throw before the first `await` inside would be an unhandled rejection; here every path is try/caught so it works, but the pattern hides that guarantee in discipline rather than structure. The honest shape is an `async function normalize(): Promise<AuthTokenDetails>` with try/catch returning the error object.
Senior: also notes it resolves-with-error-object instead of rejecting — converting exceptions into data at the boundary is *fine* (it's a taxonomy decision), but do it with a named function, not a promise constructor.
Follow-ups: "when is `new Promise` legitimate?" (wrapping callback APIs — only then).
Drill: refactor it on paper in 5 minutes.

## Q3: "This fetch wrapper returns `new Promise((res) => {})`. What does that do to callers?"

Round: JS/TS deep-dive
What it's really testing: do you understand promise lifetimes and what "never settles" means downstream.
Repo anchor: [custom.fetch.func.ts L76-L85](../../../libraries/helpers/src/utils/custom.fetch.func.ts#L76-L85) — when `afterRequest` returns false (e.g. a 401 triggering a redirect), the wrapper returns a never-resolving promise.
Junior: "it returns an empty promise."
Mid: callers `await` forever — deliberately, so no error UI flashes during the redirect; but any `finally`-based cleanup, loading-state reset, or SWR retry attached to that promise never runs; memory held until page unload.
Senior: names the alternative — throw a typed `RedirectingError` and catch it at one boundary — and the trade-off (every caller must not toast on it). Also spots that during SSR (`typeof window === 'undefined'`) the cookie parsing is skipped ([L17-L44](../../../libraries/helpers/src/utils/custom.fetch.func.ts#L16-L44)) — same function, two runtimes.
Drill: describe the exact user-visible symptom if `afterRequest` returns false on a non-redirect path.

## Q4: "JWT with no expiry — what are the consequences?"

Round: JS/TS deep-dive / security
What it's really testing: stateless-auth trade-offs.
Repo anchor: [signJWT](../../../libraries/helpers/src/auth/auth.service.ts#L42-L44) — `sign(value, secret)`, no `expiresIn`; payload is the whole user row ([jwt() L314-L319](../../../apps/backend/src/services/auth/auth.service.ts#L314-L319)).
Junior: "the token doesn't expire."
Mid: no revocation without a denylist; stale claims — `isSuperAdmin` or `activated` changes don't propagate until re-login (the middleware re-reads *orgs* from DB each request ([auth.middleware.ts L79-L83](../../../apps/backend/src/services/auth/auth.middleware.ts#L79-L91)) but trusts the user claims from the token).
Senior: design answer — short-lived access + refresh rotation, or session ids; and the *reset-password contrast in the same file*: the forgot-token carries a hand-rolled `expires` claim checked manually ([auth.service.ts L223-L245](../../../apps/backend/src/services/auth/auth.service.ts#L217-L245)) instead of JWT's standard `exp` — and isn't single-use.
Drill: list the three claims you'd keep in the session JWT (sub, orgId?, iat/exp) and why not the rest.

## Q5: "How does TypeScript know which settings class to validate here?"

Round: JS/TS deep-dive
What it's really testing: runtime vs compile-time types; discriminated unions; decorators.
Repo anchor: [create.post.dto.ts L71-L80](../../../libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts#L71-L80) — class-transformer discriminator on `settings.__type`, populated server-side in [mapTypeToPost L268-L275](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L252-L292).
Junior: "the decorator picks the right class."
Mid: TS types are erased at runtime; class-validator/class-transformer rebuild runtime knowledge via decorator metadata (`reflect-metadata`); the `__type` field is a **discriminant** — the same idea as a TS discriminated union, but enforced at runtime. Note the server stamps `__type` itself from the DB row rather than trusting the client.
Senior: compares with zod/valibot (schema-first, single source for both runtime and static types via `z.infer`), and notes `ValidateIf((o) => o.type !== 'draft')` means **drafts skip settings validation** — an intentional funnel-friction decision with a data-quality cost when the draft is later scheduled ([mapTypeToPost](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L243-L292) revalidates with `replaceDraft`).
Drill: sketch the zod equivalent of this discriminator in 10 lines.

## Q6: "`@Body() rawBody: any` — what does `any` cost here?"

Round: JS/TS deep-dive / code review
What it's really testing: `any` vs `unknown`, where type safety must be re-established.
Repo anchor: [posts.controller.ts L170-L179](../../../apps/backend/src/api/routes/posts.controller.ts#L170-L179); safety restored manually in [mapTypeToPost](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L243-L292).
Junior: "any disables type checking."
Mid: `any` is *contagious* — everything derived from it is `any`; `unknown` would force narrowing. Here the global `ValidationPipe` ([main.ts L52-L56](../../../apps/backend/src/main.ts#L52-L56)) can't help because the parameter type is `any`, so validation had to be re-invoked by hand — the type escape created a *second* validation pathway to maintain.
Senior: why it's probably `any` — the endpoint accepts several shapes (see [getPostGroupDebugExport's export format](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L439-L484)) and mutation happens before validation; the fix is a union DTO or a small "pre-map" type, not heroics.
Drill: write the 6-line `unknown`-based version of the controller signature and the first narrowing check.

## Q7: "Why does this monorepo import server code into the frontend without breaking?"

Round: JS/TS deep-dive / tooling
What it's really testing: module resolution, workspace layout, tree-shaking boundaries.
Repo anchor: `@gitroom/*` path aliases (see imports like `@gitroom/helpers/utils/custom.fetch` in [launches.component.tsx L9](../../../apps/frontend/src/components/launches/launches.component.tsx#L1-L15)); shared codec [posts.list.minify.ts](../../../libraries/helpers/src/utils/posts.list.minify.ts) used by both backend service and browser calendar; even the *frontend* importing from `@gitroom/nestjs-libraries` ([add.edit.modal.tsx L6](../../../apps/frontend/src/components/new-launch/add.edit.modal.tsx#L1-L11) imports `makeId`).
Junior: "path aliases in tsconfig."
Mid: aliases resolve at build time per-app (Next bundles what it imports); sharing works only if the imported file's transitive deps are isomorphic — `makeId` is pure, so it's safe; importing anything that touches `fs`/Prisma would explode the client bundle or crash at runtime. The boundary is **unenforced** — nothing stops a client file importing a server-only module.
Senior: names enforcement options (eslint import zones, `server-only` package marker, separate packages with exports maps) and the cost of the root-level single `package.json` (every app shares one dependency version universe — [package.json](../../../package.json#L42-L256)).
Drill: find one import in `apps/frontend` from `@gitroom/nestjs-libraries` and decide whether it's isomorphic-safe.

## Q8: "What's a memory-safe way to log unknown objects? This one handles circles."

Round: JS/TS deep-dive
What it's really testing: JSON.stringify pitfalls (circular refs, BigInt, getters).
Repo anchor: [safeStringify with WeakSet](../../../libraries/nestjs-libraries/src/integrations/social.abstract.ts#L35-L47).
Junior: "JSON.stringify fails on circular objects, this fixes it."
Mid: the replacer walks the object; `WeakSet` marks visited objects without preventing GC; returns `'[Circular]'` sentinel. Still loses functions/undefined/symbols, and shared-but-not-circular subtrees serialize twice.
Senior: when to reach for `util.inspect`/structuredClone instead; and why this exists *here* — provider errors get stringified into Temporal `ApplicationFailure` details ([L82-L98](../../../libraries/nestjs-libraries/src/integrations/social.abstract.ts#L74-L99)) which must be serializable across process boundaries.
Drill: name two object types this still serializes badly.

## Q9: "Explain this recursive retry and when it becomes a stack overflow / infinite loop."

Round: JS/TS deep-dive
What it's really testing: recursion vs iteration for retries, termination conditions.
Repo anchor: [checkPostAnalytics retries itself once on RefreshToken with forceRefresh=true](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L148-L229); [SocialAbstract.fetch recurses on 429/500 with totalRetries counter](../../../libraries/nestjs-libraries/src/integrations/social.abstract.ts#L101-L152); [findTime() random-date recursion](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L892-L910); [findFreeDateTimeRecursive walks days forward](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L995-L1018).
Junior: traces one call.
Mid: async recursion doesn't grow the JS stack per await (each continuation is a fresh microtask), so "stack overflow" is the wrong fear — **non-termination** is the right one. `fetch` terminates via `totalRetries > 2`; `checkPostAnalytics` terminates because `forceRefresh=true` on the retry path skips the refresh branch… unless `postAnalytics` throws `RefreshToken` again — then it re-enters with the same args → potential infinite loop (investigate).
Senior: `findFreeDateTimeRecursive` has no upper bound — if `getPostsCountsByDates` always returns empty (e.g. an org with zero posting times), it walks days forever, one DB query per day. Names the fix: iteration cap + explicit failure.
Drill: for each of the four anchors, write down the termination condition or "none".

## Q10: "Node one-process-per-concern: why three Nest apps instead of one?"

Round: JS/TS deep-dive / architecture
What it's really testing: process model, deployability, worker isolation.
Repo anchor: backend HTTP ([apps/backend/src/main.ts](../../../apps/backend/src/main.ts#L22-L81)), orchestrator Temporal worker + tiny health server ([apps/orchestrator/src/main.ts](../../../apps/orchestrator/src/main.ts#L13-L22)), frontend Next server; all sharing `libraries/` services.
Junior: "separation of concerns."
Mid: the worker's load profile (long-lived activities, provider I/O, sharp image transcodes) must not steal event-loop time from request latency; separate processes = independent scaling and deploys; shared DI modules mean the *same* PostsService runs in both ([post.activity.ts L57-L66](../../../apps/orchestrator/src/activities/post.activity.ts#L54-L66)).
Senior: notes the coupling cost — a schema change redeploys everything (single dependency universe), and the orchestrator importing HTTP-oriented services can drag in surprising deps; would consider extracting a leaner "domain" package.
Drill: name one service used by both processes and one that must never be (hint: anything reading `req`).

## Q11: "Where would `process.env.TZ = 'UTC'` bite you, and why does this app set it?"

Round: JS/TS deep-dive
What it's really testing: dates/timezones in Node, UTC discipline.
Repo anchor: [main.ts L10](../../../apps/backend/src/main.ts#L10); dayjs UTC plugin usage everywhere ([posts.repository.ts L131-L132](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L129-L133)); user timezone stored as an int offset ([schema User.timezone L91](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L79-L123)).
Junior: "so times are consistent."
Mid: forcing process TZ makes `new Date()` formatting deterministic across dev machines and servers; all business times normalize to UTC at the boundary and convert for display client-side. `publishDate` comparisons ([workflow L96-L102](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L96-L102)) would be wrong if the API and worker disagreed on TZ.
Senior: storing timezone as a fixed **offset int** loses DST — a user at UTC+2 in summer is UTC+1 in winter; IANA zone strings are the correct type. Check how `findFreeDateTime` uses `dayjs.utc().startOf('day')` ([posts.service.ts L974-L984](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L974-L984)) — whose midnight?
Drill: describe the DST bug a fixed offset causes for a post scheduled "9am every Monday".

## Q12: "Decorators: what actually happens when Nest sees `@Injectable()` / `@ActivityMethod()`?"

Round: JS/TS deep-dive
What it's really testing: metadata reflection, DI containers.
Repo anchor: [PostActivity](../../../apps/orchestrator/src/activities/post.activity.ts#L54-L68) (`@Activity()` + `@ActivityMethod()`); custom metadata via `Reflect.getMetadata('custom:plug', …)` in [integration.manager.ts L132-L152](../../../libraries/nestjs-libraries/src/integrations/integration.manager.ts#L132-L152); the `@Plug` decorator declaring scheduled automations *as metadata on provider methods* ([x.provider.ts L110](../../../libraries/nestjs-libraries/src/integrations/social/x.provider.ts#L110)).
Junior: "decorators register the class."
Mid: decorators run at class-definition time and write key/value metadata via `reflect-metadata`; Nest's container later reads constructor param types (`emitDecoratorMetadata`) to inject; the plug system is a home-grown version of the same trick — behavior discovered by scanning metadata, not by registration lists.
Senior: costs — decorator metadata breaks with `import type`, is invisible to tree-shaking, and couples you to experimental decorators; contrasts with explicit registration (the provider *array* at [integration.manager.ts L40-L75](../../../libraries/nestjs-libraries/src/integrations/integration.manager.ts#L40-L75) — both styles in one file!).
Drill: explain why `getAllPlugs` stringifies field `validation` RegExps ([L142-L147](../../../libraries/nestjs-libraries/src/integrations/integration.manager.ts#L132-L152)). (It crosses a JSON boundary to the frontend.)

## Q13: "What does `satisfies`/narrowing buy you that a type annotation doesn't?" (conceptual — thin repo anchor)

Round: JS/TS deep-dive
What it's really testing: TS inference vs widening.
Repo anchor: mostly *counter-examples* — pervasive `@ts-ignore` ([auth.middleware.ts L61-L72](../../../apps/backend/src/services/auth/auth.middleware.ts#L50-L76), [custom.fetch.tsx L16-L37](../../../libraries/helpers/src/utils/custom.fetch.tsx#L14-L42)) where proper typing of `req.user`/`req.org` (module augmentation of Express's `Request`) would remove every ignore at once.
Junior: defines `satisfies`.
Mid: annotation *widens* (checks assignability then forgets), `satisfies` checks *without* widening so literal keys/discriminants survive; narrowing via `in`/`typeof`/discriminant checks replaces casts.
Senior: connects to this repo — the `PostWithConditionals` type ([posts.service.ts L53-L56](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L53-L56)) papering over Prisma include-dependent result shapes; mentions `Prisma.PostGetPayload<{include: …}>` as the real fix.
Drill: write the Express `Request` augmentation that would delete the `@ts-expect-error`s in the middleware.

## Q14: "How do you keep a hot path from serializing huge objects repeatedly?"

Round: JS/TS deep-dive / performance
What it's really testing: cost model for JSON, payload discipline.
Repo anchor: [slimPost](../../../apps/orchestrator/src/activities/post.activity.ts#L27-L52) — strips `error` (grows per retry), `childrenPost`, and other unread fields before returning posts into workflow history; the comment on L27 states the rationale; also [updateTags round-tripping the entire posts array through JSON.stringify/parse to do string replacement](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L553-L575).
Junior: "remove unused fields."
Mid: Temporal persists every activity result in workflow history — payload size is a *durability* cost, not just bandwidth; slimming at the boundary is the cheapest fix. Meanwhile `updateTags` treats serialization as a regex substrate — clever, O(payload) per call, fragile against content containing the pattern.
Senior: quantifies (history size limits, replay cost), proposes explicit DTOs for activity results, and a targeted replace over `post.content` fields instead of whole-array stringification.
Drill: list the fields `slimPost` keeps that the workflow actually reads (cross-check against [workflow usage](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L104-L158)).

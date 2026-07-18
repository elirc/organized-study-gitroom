# Pattern Catalog

Sixteen patterns you can point at in this repo. The goal is **recognition** — seeing the shape in any codebase and knowing when it's the wrong shape — not name-dropping.

---

## Pattern 1: Controller → Service → Repository (layered CRUD)

Problem it solves: keeps HTTP concerns, business rules, and SQL from tangling; each layer is replaceable and (in principle) testable alone.
General shape: controller parses/authorizes → service decides → repository touches the DB. The repo mandates it in [CLAUDE.md](../../../CLAUDE.md#L31-L37).
Real example: [posts.controller.ts L165-L168](../../../apps/backend/src/api/routes/posts.controller.ts#L165-L168) → [posts.service.ts getPost L527-L547](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L527-L547) → [posts.repository.ts getPost L339-L365](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L339-L365).
Second example: tags CRUD, controller [L81-L109](../../../apps/backend/src/api/routes/posts.controller.ts#L81-L109) → service L1024-L1038 → repository L785-L826.
Why this implementation works: services live in `libraries/` so backend *and* orchestrator share them — the layering doubles as code reuse across processes.
Failure modes: "pass-through" services (most of [posts.service.ts L1020-L1047](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L1020-L1047) is one-line delegation — ceremony without value) and business logic leaking *up* into controllers ([deleteChannel](../../../apps/backend/src/api/routes/integrations.controller.ts#L402-L418) loops and deletes posts in the controller).
Use it when: multiple entry points share rules. Avoid when: a 3-line CRUD app — layers have a carrying cost.
Interview angle: "How do you structure a Node API?" — describe this, then volunteer the failure modes; that's the mid-level differentiator.
Drill: find one more controller that does business logic it shouldn't. (Candidate: [media.controller.ts generateImageFromText L71-L85](../../../apps/backend/src/api/routes/media.controller.ts#L71-L85).)

## Pattern 2: Provider abstraction over N external APIs (strategy + registry)

Problem it solves: 30+ social networks, one posting pipeline.
General shape: an abstract base ([SocialAbstract](../../../libraries/nestjs-libraries/src/integrations/social.abstract.ts#L49-L194)) supplies shared fetch/retry/error mapping; each provider implements a common interface; a registry array + lookup ([integration.manager.ts L40-L75, L170-L172](../../../libraries/nestjs-libraries/src/integrations/integration.manager.ts#L40-L75)) resolves by string identifier.
Real example: [x.provider.ts](../../../libraries/nestjs-libraries/src/integrations/social/x.provider.ts#L26-L45) (`identifier = 'x'`, `post()` L427, `handleErrors` L47-L108).
Second example: any of the 30+ files in [integrations/social/](../../../libraries/nestjs-libraries/src/integrations/social).
Why it works: the workflow calls `getIntegration.post(...)` without knowing the platform; capability differences are expressed as optional members (`comment?`, `postAnalytics?`, `isBetweenSteps`) checked at runtime ([isCommentable](../../../apps/orchestrator/src/activities/post.activity.ts#L149-L156)).
Failure modes: the registry lookup ends with `!` — an unknown identifier returns `undefined` and explodes later ([L170-L172](../../../libraries/nestjs-libraries/src/integrations/integration.manager.ts#L170-L172)); optional-member sprawl makes the interface a "maybe-bag" over time.
Use it when: many interchangeable backends. Avoid when: two variants — an `if` is cheaper.
Interview angle: classic "design an integration layer" question; also the OOP-vs-composition talking point.
Drill: add a mental "NullProvider" — what would `getSocialIntegration` return for identifier `"myspace"` and where does it first crash?

## Pattern 3: Durable timers instead of cron (Temporal workflow-per-entity)

Problem it solves: "run this at time T, exactly once-ish, surviving restarts and deploys."
General shape: one workflow per post, deterministic id `post_{postId}`, `sleep(until)` then act; the DB row and the workflow reference each other via search attributes.
Real example: [startWorkflow](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L690-L748); [the sleep](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L96-L102).
Second example: streak workflow started per-org with `TERMINATE_EXISTING` ([post.activity.ts L248-L261](../../../apps/orchestrator/src/activities/post.activity.ts#L248-L261)).
Why it works: rescheduling = terminate + restart ([L697-L716](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L697-L716)); dedup = `workflowIdConflictPolicy`.
Failure modes: the DB and Temporal are **two sources of truth** that can diverge (silent catch on start, [L747](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L747)); versioning pain — hence five workflow files v1.0.1…v1.0.5 in [post-workflows/](../../../apps/orchestrator/src/workflows/post-workflows) because running workflows can't be mutated.
Use it when: long delays, human-scale timing, retries with state. Avoid when: sub-second jobs at huge volume (queue is cheaper) or when you can't run the extra infra.
Interview angle: "cron vs queue vs workflow engine" — you have concrete trade-offs and a versioning war story from just reading the folder.
Drill: explain why the old workflow files can't be deleted yet, and what would let you delete them.

## Pattern 4: Workflow/Activity split (determinism boundary)

Problem it solves: Temporal replays workflow code to reconstruct state; anything nondeterministic must live in activities.
General shape: workflow = pure orchestration (sleeps, branching on activity results); activities = I/O, wrapped with timeouts/retry policy ([proxyActivities](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L19-L46)).
Real example: the workflow never touches Prisma or fetch; [PostActivity](../../../apps/orchestrator/src/activities/post.activity.ts#L54-L66) injects Nest services for all I/O.
Second example: dynamic task-queue proxy for per-provider concurrency ([proxyTaskQueue L19-L29, used L64-L73](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L19-L29)).
Failure modes: fat activity payloads replayed forever — which is why [slimPost](../../../apps/orchestrator/src/activities/post.activity.ts#L27-L52) strips heavy fields (the comment explains `error` grows per retry); accidental nondeterminism (e.g. `new Date()` in workflow code — spot the one at [L80](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L80)! Temporal patches `Date` inside workflow context, but that's worth saying aloud in review).
Interview angle: "What makes code replay-safe?" — determinism, idempotent activities, versioning.
Drill: classify each of the 8 destructured activities at [L31-L46](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L31-L46) as read, write, or external-call.

## Pattern 5: Typed failure taxonomy across a boundary

Problem it solves: 30 providers fail in 30 dialects; the workflow needs exactly three verdicts: *refresh the token*, *give up (bad body)*, or *retry*.
General shape: provider `handleErrors` maps raw response text → `{type, value}`; shared `fetch` throws `RefreshToken` / `BadBody` (both `ApplicationFailure` subclasses so the type survives Temporal serialization); workflow catches and switches on `err.cause.type`.
Real example: [social.abstract.ts L101-L173](../../../libraries/nestjs-libraries/src/integrations/social.abstract.ts#L101-L173) → [x.provider.ts handleErrors L47-L108](../../../libraries/nestjs-libraries/src/integrations/social/x.provider.ts#L47-L108) → [workflow catch L219-L261](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L219-L261).
Second example: same catch pattern re-implemented for plugs ([L330-L356](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L326-L358)).
Why it works: **the error type is the contract**; string matching stays quarantined inside each provider.
Failure modes: substring matching on error bodies is brittle (`body.includes('maximum of 4 items')`); unknown errors default to `BadBody` = permanent failure, so a transient outage can kill a post.
Interview angle: "How do you handle errors across service boundaries?" — typed errors, retryable vs terminal, and *who decides*.
Drill: write the `handleErrors` entry you'd add for an HTTP 503 with body `"Service Unavailable"` — which of the three verdicts?

## Pattern 6: State parameter as short-lived capability (OAuth via Redis)

Problem it solves: linking an unauthenticated OAuth callback to the org that initiated it, across browsers/devices.
General shape: authed leg writes `state → context` into Redis with TTL; callback leg redeems + deletes.
Real example: write [integrations.controller.ts L229-L248](../../../apps/backend/src/api/routes/integrations.controller.ts#L229-L248); redeem [no.auth.integrations.controller.ts L60-L94](../../../apps/backend/src/api/routes/no.auth.integrations.controller.ts#L60-L94).
Second example: `webhookUrl:{state}` for third-party app installs ([L269-L284](../../../apps/backend/src/api/routes/no.auth.integrations.controller.ts#L269-L284)).
Failure modes: Redis restart mid-dance orphans every pending connect; forgetting single-use deletion enables replay.
Interview angle: capability tokens, CSRF-like `state` semantics, why TTL matters.
Drill: enumerate the six Redis key prefixes in this flow and their lifecycle.

## Pattern 7: Org-scoped repository queries (multi-tenancy by convention)

Problem it solves: tenant isolation without RLS or a query builder wrapper.
General shape: every repository method takes `orgId` and includes it in the where-clause.
Real example: [getPostsByGroup](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L321-L337); [changeDate L439-L464](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L439-L464) (id **and** orgId in `where`).
Second example (the counter-example that proves the risk): [editTag L804-L814](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L804-L814) takes `orgId` and **doesn't use it** — while sibling [deleteTag L816-L826](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L816-L826) does.
Why it works (when it works): cheap, obvious, greppable.
Failure modes: it's *convention* — nothing fails closed. One missed filter = IDOR. Alternatives: Postgres RLS, Prisma client extensions injecting the filter.
Interview angle: "How do you enforce tenant isolation?" — layers of defense; this repo gives you a live counter-example to cite (respectfully).
Drill: grep `posts.repository.ts` for `update(` and classify each `where` as org-scoped or not.

## Pattern 8: Soft delete everywhere

Problem it solves: recoverability and referential safety on user-facing data.
General shape: `deletedAt DateTime?` + every read filters `deletedAt: null`.
Real example: [deletePost sets deletedAt](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L298-L319); reads filter it ([getPosts L163](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L129-L172)).
Second example: schema-wide — `Tags`, `Media`, `Subscription`, `Webhooks`, `Integration` all carry `deletedAt` + index ([schema](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L52-L65)).
Failure modes: forget the filter once and ghosts return; unique constraints must include `deletedAt` (see `Customer @@unique([orgId, name, deletedAt])` [schema L311](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L301-L312) — and why that's still imperfect: multiple deleted rows with same name collide on the `null`… investigate Postgres NULL-distinct semantics — actually NULLs are distinct, so deleted dupes are fine but *two actives + one deleted* re-create is allowed. Think it through).
Interview angle: soft vs hard delete, GDPR tension, partial unique indexes.
Drill: find one query that *forgets* the `deletedAt` filter. (Candidate: [getPostUrls L114-L127](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L114-L127).)

## Pattern 9: JSON-in-string columns (schemaless corners of a relational schema)

Problem it solves: per-provider settings and media lists vary wildly; a column per field is impossible.
General shape: `String` columns holding JSON (`Post.settings`, `Post.image`, `Integration.postingTimes` with a JSON default — [schema L332](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L332)), parsed at each use site.
Real example: `JSON.parse(post.settings || '{}')` in [posts.service.ts L505](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L486-L507) and again in the activity ([post.activity.ts L237](../../../apps/orchestrator/src/activities/post.activity.ts#L226-L246)).
Why it (sort of) works: DTO discriminator validates the JSON *on the way in* ([create.post.dto.ts L71-L80](../../../libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts#L71-L80)), so the string is validated-at-birth.
Failure modes: no DB-level integrity, no queryability, parse scattered everywhere (no single deserialization boundary), and Prisma `Json` type was available but unused — ask why (portability? habit?).
Interview angle: "JSONB vs columns" — when denormalized blobs are fine and what you give up.
Drill: count `JSON.parse(` occurrences in `posts.service.ts` (grep) and propose the single choke-point you'd introduce.

## Pattern 10: Upsert-loop thread building (linked list in SQL)

Problem it solves: a post thread (X thread, IG carousel comments) is an ordered chain of rows.
General shape: iterate values; each upsert `connect`s `parentPost` to the previous row's id; a shared `group` uuid marks the set.
Real example: [createOrUpdatePost L488-L631](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L488-L631); read back recursively ([getPostsRecursively](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L294-L322)) or by group + in-memory re-chain ([arrangePostsByGroup L509-L525](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L509-L525)).
Failure modes: no transaction (partial thread on crash); recursion loads one row per query (N+1 by design — fine at thread length 3, bad at 300); `childrenPost: true` include side-loads every child at every level ([repo L339-L365](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L339-L365)).
Interview angle: modeling ordered hierarchies (adjacency list vs position column vs closure table).
Drill: rewrite the recursive read as a single `findMany({where:{group}})` + in-memory sort — the repo already has both; when does each win?

## Pattern 11: Guard + decorator authorization (CASL policies)

Problem it solves: plan limits (channels, posts/month, webhooks) enforced declaratively per route.
General shape: `@CheckPolicies([action, section])` metadata → global [PoliciesGuard](../../../apps/backend/src/services/auth/permissions/permissions.guard.ts#L22-L64) → [PermissionsService.check](../../../apps/backend/src/services/auth/permissions/permissions.service.ts#L39-L172) builds a CASL ability from live counts vs [pricing](../../../libraries/nestjs-libraries/src/database/prisma/subscriptions/pricing.ts) → throws `SubscriptionException` → filter maps it to a payment-required response.
Real example: [posts.controller.ts L170-L171](../../../apps/backend/src/api/routes/posts.controller.ts#L170-L179).
Second example: channel connect ([integrations.controller.ts L46-L47](../../../apps/backend/src/api/routes/integrations.controller.ts#L46-L54)).
Why it works: quota logic in one service; self-hosted mode short-circuits to allow-all ([L50-L63](../../../apps/backend/src/services/auth/permissions/permissions.service.ts#L50-L63)).
Failure modes: path-string bypasses in the guard ([L24-L31](../../../apps/backend/src/services/auth/permissions/permissions.guard.ts#L24-L31), including a duplicated `/auth` condition — copy-paste tell); counting queries on every gated request (N channels fetched just to count — [L83-L94](../../../apps/backend/src/services/auth/permissions/permissions.service.ts#L83-L94)).
Interview angle: RBAC vs ABAC vs "ability" models; where quota enforcement belongs.
Drill: follow one `SubscriptionException` from throw to HTTP response ([subscription.exception.ts](../../../apps/backend/src/services/auth/permissions/subscription.exception.ts)).

## Pattern 12: Compensating sweep (reconciliation cron)

Problem it solves: fire-and-forget writes eventually drift; a periodic scan repairs the drift.
Real example: [missingPostWorkflow](../../../apps/orchestrator/src/workflows/missing.post.workflow.ts#L13-L19) + [searchForMissingThreeHoursPosts](../../../apps/orchestrator/src/activities/post.activity.ts#L73-L106) + idempotent `signalWithStart`.
Second example: token refresh workflow started on connect ([refresh.integration.service.ts](../../../libraries/nestjs-libraries/src/integrations/refresh.integration.service.ts)).
Why it works: repair is idempotent, so running it hourly is safe.
Failure modes: gated behind `RUN_CRON` env ([infinite.workflow.register.ts L9](../../../libraries/nestjs-libraries/src/temporal/infinite.workflow.register.ts#L8-L19)) — an invisible deployment invariant; the 2-day window bounds repair.
Interview angle: reconciliation loops are the grown-up answer to "what if the message is lost?"
Drill: name two other writes in Flow 1 that have **no** compensating sweep (webhook delivery; short-link creation).

## Pattern 13: Storage strategy behind a factory

Problem it solves: local disk for self-hosters, Cloudflare R2 for cloud, one call site.
Real example: [UploadFactory.createStorage()](../../../libraries/nestjs-libraries/src/upload/upload.factory.ts) selected by env; used as a field initializer in [media.controller.ts L32](../../../apps/backend/src/api/routes/media.controller.ts#L29-L36) and [posts.service.ts L60](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L58-L70).
Second example: implementations [local.storage.ts](../../../libraries/nestjs-libraries/src/upload/local.storage.ts) / [cloudflare.storage.ts](../../../libraries/nestjs-libraries/src/upload/cloudflare.storage.ts) behind [upload.interface.ts](../../../libraries/nestjs-libraries/src/upload/upload.interface.ts).
Failure modes: chosen once at module load — config changes need restarts; not injected, so tests can't swap it (compare: it *could* be a Nest provider).
Interview angle: strategy + factory; and "why is `new`-in-field-initializer hostile to testing?"
Drill: sketch how you'd make this injectable without changing call sites (Nest custom provider + token).

## Pattern 14: SWR + context-provided fetch (frontend data layer)

Problem it solves: consistent auth headers/base URL and cache-keyed server state in React.
General shape: a configured `fetch` lives in React context ([custom.fetch.tsx L14-L46](../../../libraries/helpers/src/utils/custom.fetch.tsx#L14-L46)); components call `useFetch()` and wrap requests in `useSWR` hooks — one hook per resource, per [CLAUDE.md's rule](../../../CLAUDE.md#L43-L58).
Real example: [calendar.context.tsx L1-L50](../../../apps/frontend/src/components/launches/calendar.context.tsx#L1-L50) (SWR-fed calendar context), [launches.component.tsx L353-L364](../../../apps/frontend/src/components/launches/launches.component.tsx#L353-L364) (`useFetch` + `mutate`).
Why it works: SWR gives cache, dedupe, revalidation; `mutate()` after writes keeps the calendar honest.
Failure modes: cache keys must encode all inputs or you serve stale cross-filter data; the custom fetch returns a **never-resolving Promise** when `afterRequest` fails ([custom.fetch.func.ts L83-L85](../../../libraries/helpers/src/utils/custom.fetch.func.ts#L76-L85)) — intentional "halt the world" on 401 redirects, but a footgun anywhere else.
Interview angle: "server state vs client state" (SWR/React Query vs zustand — this repo uses **both**, correctly split: see Pattern 15).
Drill: find the SWR key for the integrations list and predict when it revalidates.

## Pattern 15: Zustand for composer state (client state machine)

Problem it solves: the post composer is a multi-channel, multi-tab editor — too much interlocked state for prop drilling, none of it server-owned.
Real example: [new-launch/store.ts L28-L60](../../../apps/frontend/src/components/new-launch/store.ts#L28-L60) (one store: selected integrations, per-channel values, global values, tags, editor mode); consumed with `useShallow` selectors in [add.edit.modal.tsx L37-L47](../../../apps/frontend/src/components/new-launch/add.edit.modal.tsx#L36-L60).
Why it works: selector-based subscriptions avoid re-rendering the whole composer per keystroke; `reset()` on unmount ([L209-L211](../../../apps/frontend/src/components/new-launch/add.edit.modal.tsx#L209-L212)) prevents state bleed between modal opens.
Failure modes: module-level store is a **singleton** — two composers can't coexist; empty-deps `useEffect` initialization ([L48-L53](../../../apps/frontend/src/components/new-launch/add.edit.modal.tsx#L48-L53)) silently ignores prop changes after mount.
Interview angle: "when zustand/Redux vs SWR vs useState" — this repo is a clean worked example of the three-way split.
Drill: list every field in `StoreState` and label it: server-derived, user-draft, or UI-only.

## Pattern 16: Minify/expand wire-format pairing

Problem it solves: calendar fetches whole months of posts; full rows are heavy.
General shape: server minifies to a compact array shape; client expands back — a matched encoder/decoder pair in a **shared** library so they can't drift independently.
Real example: [minifyPosts / minifyPostsList](../../../libraries/helpers/src/utils/posts.list.minify.ts) used in [posts.service.ts L328-L338](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L324-L338); expanded client-side via `expandPosts` ([calendar.context.tsx L25](../../../apps/frontend/src/components/launches/calendar.context.tsx#L1-L50)).
Why it works: the codec lives in `libraries/helpers`, imported by both sides — the monorepo advantage in one line.
Failure modes: hand-rolled codecs need versioning discipline; adding a field means touching both directions (a test would catch drift — none exists).
Interview angle: payload design, over-fetching, and why shared types beat duplicated interfaces.
Drill: open `posts.list.minify.ts` and write the round-trip property test you'd add (`expand(minify(x)) ⊇ x`'s used fields).

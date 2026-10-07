# Good First Tickets (Junior)

Eighteen tickets, spread across API, services, orchestrator, frontend, and docs. Each is small-blast-radius, follows an existing pattern, and states what would make a maintainer reject it. Template fields kept tight; expand your own design notes before coding.

**Repo reality check**: there is no test infrastructure (zero spec files). "Suggested checks" therefore means: a manual verification script *plus* the unit test you would write if you also pick up ticket J17 (bootstrap testing) — doing J17 first multiplies the value of every other ticket.

---

## Ticket J1: Remove full-body `console.log` from post creation
Difficulty: Easy — 30 min. Skills: log hygiene, PII awareness.
Story: As a self-hoster, I don't want every post's full content and media URLs dumped to stdout on every create.
Anchors: [posts.controller.ts L176](../../../apps/backend/src/api/routes/posts.controller.ts#L170-L179). Also sweep siblings: `console.log('hello')` in [media.controller.ts L48](../../../apps/backend/src/api/routes/media.controller.ts#L43-L50), `console.log('reconnect')` in [no.auth.integrations.controller.ts L131](../../../apps/backend/src/api/routes/no.auth.integrations.controller.ts#L130-L132).
Plan: delete or downgrade to a redacted debug log behind an env flag.
Rejection risk: maintainer may *want* a creation log — replace with `Logger.debug` of ids only, not deletion plus nothing.
Interview story potential: "found PII leaking into logs; shipped a redaction policy."

## Ticket J2: Fix `editTag` missing org scope (confirmed cross-tenant write)
Difficulty: Easy code, Medium significance — 1 h. Skills: IDOR reasoning, Prisma where-clauses.
Story: As an org member, I must not be able to rename another org's tag by guessing its uuid.
Anchors: [editTag L804-L814](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L804-L814) (no `orgId` in `where`); contrast [deleteTag L816-L826](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L816-L826); route [posts.controller.ts L94-L101](../../../apps/backend/src/api/routes/posts.controller.ts#L94-L101).
Plan: add `orgId` to the `where`; Prisma compound where on non-unique fields needs `updateMany` or a `findFirst`+`update` — match `deleteTag`'s shape.
What could go wrong: `update` on a compound non-unique where won't compile → use the same approach the sibling uses.
Checks: two orgs, tag id from org A, PUT from org B → expect 4xx/no-op.
Review questions: are there other `update({where: {id}})`-only methods with an unused orgId param? (Grep — that's ticket M2's audit.)
**Check (audit warm-up):** the repositories have more than a dozen id-only `update({ where: { id } })` calls. Examples are `posts.repository.ts` L103 `updateImages`, L367 `updatePost`, L393 `changeState`, L655 `updateMessage`, and `integration.repository.ts` L365 `updateNameAndUrl`. For each one, decide whether it is reachable with a user-supplied id (follow the callers into `apps/backend/src/api/routes/`) or only from workflows, and whether a caller already did an org-scoped read. `updateNameAndUrl` is the worked example: `setNickname` checks `getIntegrationById(org.id, id)` first (`integrations.controller.ts` L144–150). `editTag` is the one whose signature takes `orgId` and then ignores it.
Interview story potential: "found and fixed an IDOR with a one-line where-clause change and wrote the cross-tenant regression test."

## Ticket J3: Fix `createPopularPosts` inserting literal strings
Difficulty: Easy — 20 min. Skills: reading diffs carefully.
Anchors: [posts.repository.ts L728-L742](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L728-L742) — `data: { category: 'category', topic: 'topic', … }` ignores the argument.
Plan: use the destructured values. Find callers first (`createPopularPosts` in [posts.service.ts L986-L993](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L986-L993)) to confirm intent.
Rejection risk: if the feature is dead code, the maintainer may prefer deletion — ask in the PR description.
Interview story: "classic copy-paste bug; my fix included a caller audit."

## Ticket J4: `deletePost` returns `{ error: true }` on success
Difficulty: Easy — 45 min. Skills: API contracts.
Anchors: [posts.service.ts L651-L680](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L651-L680); controller [DELETE /posts/:group](../../../apps/backend/src/api/routes/posts.controller.ts#L205-L211).
Plan: return `{ id: post?.id }` or 204; check the frontend caller for what it reads before changing the shape (grep `deletePost` in `apps/frontend`).
What could go wrong: a client switch on `error` — this is a **contract change**; version carefully and say so in the PR.
Interview story: "even a wrong response shape is a contract; I changed it with a consumer audit."

## Ticket J5: Guard `changeDate` against missing post
Difficulty: Easy — 45 min. Skills: null-safety, error taxonomy.
Anchors: [posts.service.ts L843-L875](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L843-L875) — `getPostById` result dereferenced (`getPostById.state`) without a null check (sibling [changePostStatus L818-L841](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L818-L841) *does* check).
Plan: mirror the sibling's `BadRequestException('Post not found')`.
Checks: PUT `/posts/:id/date` with a foreign/deleted id → clean 400, not a 500 TypeError.
Interview story: "consistency between sibling code paths as a review lens."

## Ticket J6: Delete the vestigial `poke` flag in the post workflow
Difficulty: Easy — 30 min + careful thinking. Skills: Temporal versioning awareness.
Anchors: [post.workflow.v1.0.5.ts L48, L75-L78](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L48-L78) — signal handler sets `poked`, never read; the sweep still sends the signal ([post.activity.ts L79-L84](../../../apps/orchestrator/src/activities/post.activity.ts#L73-L106)).
Plan: **don't** remove the signal handler (running workflows + `signalWithStart` depend on the signal existing); remove only the dead variable, and document why the handler must stay.
Rejection risk: removing the handler breaks `signalWithStart` on live workflows — showing you know this is the whole point of the ticket.
Interview story: "dead code isn't always safe to delete: durable-execution constraints."

## Ticket J7: Duplicate `/auth` condition in PoliciesGuard
Difficulty: Easy — 15 min. Skills: attentive reading.
Anchors: [permissions.guard.ts L24-L31](../../../apps/backend/src/services/auth/permissions/permissions.guard.ts#L24-L31).
Plan: remove the duplicated condition; while there, extract the path list to a named constant.
Interview story: small, but pairs with J8 as "guard hygiene".

## Ticket J8: Replace path-substring bypass with route metadata
Difficulty: Medium — 2-3 h. Skills: Nest decorators, guards.
Story: `request.path.indexOf('/auth') > -1` also matches `/oauth/...` and any future route containing the substring — a fragile allow-list.
Anchors: [permissions.guard.ts L24-L31](../../../apps/backend/src/services/auth/permissions/permissions.guard.ts#L22-L64).
Plan: introduce `@SkipPolicies()` metadata + `Reflector` check, mirroring how `CheckPolicies` already works ([permissions.ability.ts](../../../apps/backend/src/services/auth/permissions/permissions.ability.ts#L1-L7)); annotate the three intended controllers.
What could go wrong: missing one bypassed route = lockout; enumerate current matches first (grep controllers for paths containing `auth`, `social-connect`, `provider`).
Interview story: "replaced string matching with declarative metadata — fail-closed authorization."

## Ticket J9: `getPostUrls` misses the soft-delete filter
Difficulty: Easy — 30 min. Skills: soft-delete discipline.
Anchors: [posts.repository.ts L114-L127](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L114-L127); used by [updateTags](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L553-L575) to resolve `(post:id)` references.
Plan: add `deletedAt: null`. Consider: is resolving a URL from a deleted-but-published post actually wrong? Write your reasoning in the PR (maybe it's *correct* to keep it!). This ticket's real skill is arguing both sides.
Interview story: "a 'bug fix' I talked myself out of / into, with evidence."

## Ticket J10: Type `req.user` / `req.org` via Express module augmentation
Difficulty: Medium — 2 h. Skills: TS declaration merging.
Anchors: the `@ts-expect-error` cluster in [auth.middleware.ts L61-L99](../../../apps/backend/src/services/auth/auth.middleware.ts#L50-L104); decorators [org.from.request.ts](../../../libraries/nestjs-libraries/src/user/org.from.request.ts#L1-L8).
Plan: `declare global { namespace Express { interface Request { user?: User; org?: Organization & {users: …} } } }` in a `.d.ts`; delete the ignores that become unnecessary.
Rejection risk: the org shape isn't `Organization` exactly (it carries `users[0].role` — see [permissions.guard.ts L50](../../../apps/backend/src/services/auth/permissions/permissions.guard.ts#L43-L51)); define the *real* shape or you've documented a lie.
Interview story: "typed an untyped seam and the compiler found the undocumented shape."

## Ticket J11: Extract duplicated refresh-token logic in PostsService
Difficulty: Medium — 2 h. Skills: refactoring, DRY judgment.
Anchors: [getMissingContent L100-L123](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L80-L138) and [checkPostAnalytics L173-L196](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L148-L229) — identical 20-line refresh/disconnect blocks.
Plan: private `withFreshToken(integration, fn)` helper; keep behavior identical (including the 10s `refreshWait`).
Interview story: "rule-of-three refactor with behavior-preserving evidence."

## Ticket J12: Fix stale-doc drift in CLAUDE.md
Difficulty: Easy — 30 min. Skills: docs as contracts.
Anchors: [CLAUDE.md L16](../../../CLAUDE.md#L14-L17) says frontend is "Vite ReactJS"; [apps/frontend/package.json](../../../apps/frontend/package.json) runs Next 16.
Plan: correct the framework, verify the other claims while you're there (the 3-layer rule, tailwind version — root deps list both tailwind 3.4.17 and `@tailwindcss/postcss` 4.x, [package.json L94, L236](../../../package.json#L94)).
Interview story: "docs drift detection — small PR, high trust yield."

## Ticket J13: Surface workflow-start failures in `createPost`
Difficulty: Medium — 2-3 h. Skills: failure visibility.
Anchors: [posts.service.ts L791-L798](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L790-L798) — `.catch((err) => {})`; and [startWorkflow's internal catch L747](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L722-L748).
Plan: log with Sentry (already wired: `Sentry.metrics.count` at [L800](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L800)) + increment a `workflow_start_failed` metric. Do **not** fail the request (the sweep will repair) — but say why in a comment.
Interview story: "made an intentional fire-and-forget observable without changing its semantics."

## Ticket J14: Add `alt` text support surfaced in the composer (frontend)
Difficulty: Medium — 3-4 h. Skills: React forms, existing model reuse.
Anchors: `Media.alt` exists in the schema ([L221](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L209-L231)) and save endpoint ([media.controller.ts /information L122-L128](../../../apps/backend/src/api/routes/media.controller.ts#L122-L128)); check where the composer renders media (new-launch editor components) and whether alt is editable.
Plan: investigate first — if an alt editor already exists, convert this ticket into "document it"; if not, add the field to the media modal following an existing input component in `apps/frontend/src/components/ui`.
Rejection risk: UI without provider support is cosmetic — check which providers accept alt text before promising it publishes.
Interview story: accessibility + full-stack slice.

## Ticket J15: Normalize email casing at login
Difficulty: Easy — 1 h. Skills: auth edge cases, data migration thinking.
Anchors: register lowercases ([auth.service.ts L46-L48](../../../apps/backend/src/services/auth/auth.service.ts#L42-L49)); login does not ([L49, L85](../../../apps/backend/src/services/auth/auth.service.ts#L49-L94)).
Plan: lowercase at login too. Danger: legacy mixed-case rows now unreachable? They already are for *new* registrations; check `getUserByEmail` implementation for case behavior first ([users.service.ts](../../../libraries/nestjs-libraries/src/database/prisma/users/users.service.ts)).
Interview story: "auth edge case + backfill reasoning."

## Ticket J16: Add missing `deletedAt` index audit note / composite indexes
Difficulty: Easy investigation — 2 h. Skills: reading query patterns vs schema indexes.
Anchors: hottest query [getPosts L134-L197](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L129-L220) filters `organizationId + publishDate + deletedAt + parentPostId`; schema has single-column indexes only ([Post indexes L431-L445](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L431-L445)).
Plan: propose `@@index([organizationId, publishDate])` with EXPLAIN evidence from seeded data; this is a proposal PR (schema + reasoning), maintainer decides.
Interview story: "index selection from real query shapes" — a guaranteed interview topic.

## Ticket J17: Bootstrap the test infrastructure (the multiplier)
Difficulty: Medium — half a day. Skills: jest config in a pnpm monorepo.
Story: `pnpm test` currently references `@nx/jest` ([jest.config.ts](../../../jest.config.ts)) but there's no nx workspace file — tests can't run at all.
Plan: minimal `jest.config.ts` using ts-jest against `libraries/nestjs-libraries/src/**/*.spec.ts`; first test target: pure functions with zero infra — [minify/expand codec](../../../libraries/helpers/src/utils/posts.list.minify.ts), [safeStringify](../../../libraries/nestjs-libraries/src/integrations/social.abstract.ts#L35-L47), [checkScopes](../../../libraries/nestjs-libraries/src/integrations/social.abstract.ts#L175-L193).
Rejection risk: maintainers may have CI opinions — keep it additive, no script changes beyond `test`.
Interview story: "introduced testing to an untested repo — chose seams with zero mocking first."

## Ticket J18: Document the `RUN_CRON` invariant
Difficulty: Easy — 1 h. Skills: operational docs.
Anchors: [infinite.workflow.register.ts L8-L19](../../../libraries/nestjs-libraries/src/temporal/infinite.workflow.register.ts#L8-L19) — without `RUN_CRON`, the missed-post sweep never runs; nothing in `.env.example`'s vicinity explains that one instance (and only one) should set it.
Plan: README/env docs entry: what it does, why exactly one instance, what breaks without it.
Interview story: "deployment invariants that live in env vars are documentation debt — I paid it."

---

Every ticket above: write the PR description using the template in [07-career-and-collaboration/02-writing-prs-and-rfcs.md](../07-career-and-collaboration/02-writing-prs-and-rfcs.md) (what/why/how-tested/risk/rollback), even if you never open the PR. The description *is* the exercise.

# Refactor & Design Katas

Nine senior katas. Each has a self-grading criterion. Do them on paper or in a branch you throw away — the thinking is the point.

## Kata 1: Fix a boundary leak
Target: [deleteChannel](../../../apps/backend/src/api/routes/integrations.controller.ts#L402-L418) loops posts and orchestrates deletes *in the controller*.
Task: move the orchestration into a service method; controller becomes a one-liner.
Self-grade — Strong: your service method is transactional-aware, testable without HTTP, and you identified that the current `.catch(() => {})` on each delete swallows failures.

## Kata 2: Design an eventing change (outbox)
Target: fire-and-forget workflow start ([posts.service L777-L798](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L777-L798)).
Task: write the RFC for a transactional outbox (project S3) — schema, dispatcher, migration-alongside-sweep, how you prove it works.
Self-grade — Strong: your migration keeps the sweep running and uses its repair-count dropping to zero as the success metric; you address dispatcher failure.

## Kata 3: Split a module
Target: [posts.service.ts](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts) is 1048 lines mixing scheduling, media, analytics, tags, plugs, generation.
Task: propose a split (e.g. PostSchedulingService, PostMediaService, PostAnalyticsService) with clear ownership; identify shared state that resists splitting.
Self-grade — Strong: your seams follow the flows (Flow 1 vs Flow 7 vs analytics), you keep the public method surface stable, and you note which methods are pure pass-throughs that should just be deleted.

## Kata 4: Remove duplication
Target: the refresh-token block duplicated in [getMissingContent L100-L123](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L80-L138) and [checkPostAnalytics L173-L196](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L148-L229); the copy-pasted retry/catch in the [workflow's plug loops](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L326-L413).
Task: extract `withFreshToken(integration, fn)` and a `retryWithRefresh` helper; preserve exact behavior including `refreshWait`.
Self-grade — Strong: behavior-identical (including the 10s wait and the disconnect-on-hard-fail), and you note the workflow helper must stay determinism-safe.

## Kata 5: Improve type safety
Target: the `@ts-ignore`/`@ts-expect-error` cluster around `req.user`/`req.org` ([auth.middleware](../../../apps/backend/src/services/auth/auth.middleware.ts#L50-L104), [permissions.guard L43-L50](../../../apps/backend/src/services/auth/permissions/permissions.guard.ts#L43-L51)) and `PostWithConditionals` ([posts.service L53-L56](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L53-L56)).
Task: Express `Request` module augmentation (with the *real* org shape incl. `users[0].role`); replace the conditional type with `Prisma.PostGetPayload`.
Self-grade — Strong: zero remaining ignores in those files, and the augmented `org` type matches what the guard actually reads.

## Kata 6: Design a migration
Target: the recurrence model ([intervalInDays expanded at read](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L199-L219)).
Task: design the expand-contract migration to materialized occurrence rows (enabling skip/edit of a single occurrence), across three independently-deployed processes and `db push` → `migrate`.
Self-grade — Strong: your steps never leave a deployed process reading a column another hasn't written; you sequence add→backfill→dual→switch→drop; you rehearse rollback.

## Kata 7: Reduce N+1
Target: [getPostsRecursively](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L294-L322) (one query per thread item) + [`childrenPost: true` include](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L339-L365).
Task: replace with the single-query `getPostsByGroup` + in-memory `arrangePostsByGroup` where thread length can be large; measure both.
Self-grade — Strong: you kept the recursive version where it's cheap, switched where it's not, and produced a before/after query count.

## Kata 8: Write an RFC
Target: structural tenant isolation (project M2).
Task: RFC comparing Prisma client extension vs Postgres RLS vs lint rule — recommend one with a rollout that survives the intentionally-cross-org internal queries.
Self-grade — Strong: you enumerate the safe-internal queries that must opt out and explain how each option handles them.

## Kata 9: Review a flawed PR
Target: use [review kata 8 (position column)](../04-code-reading-gym/04-review-katas.md) or write your own flawed diff for a teammate to review.
Task: produce the review *and* the author's ideal response to it — practice both chairs.
Self-grade — Strong: your review catches the blocking issue, offers the fix, and stays kind; your author-response accepts the substance without defensiveness.

---

Meta-skill these build: **changing code you didn't write, safely, at scale.** For each kata ask the three senior questions — what does this commit us to, who pays the cost, how do I reduce the blast radius — and write the answer before the diff. That habit, more than any single refactor, is what mid→senior looks like.

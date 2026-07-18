# Review Katas

Ten simulated PRs. Review each: list findings as **Blocking / Important / Optional**, then compare. All diffs are illustrative (labeled) but mirror real patterns and files here. Practice the *language* too — specific, kind, actionable. This is the closest thing to a real code-review interview round.

---

## Kata 1: "Add tag rename endpoint"
Author intent: let users rename tags.
Fake diff summary: new `PUT /posts/tags/:id` → `editTag(id, orgId, body)` where the repo method does `update({ where: { id }, data })`.
Files this resembles: [posts.repository.ts L804-L814](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L804-L814).
Expected findings — **Blocking**: no org scope → any user renames any org's tag (IDOR); sibling `deleteTag` shows the correct pattern. **Important**: no test for cross-org rejection. **Optional**: validate color format.
Good comment:
> Blocking: `where: { id }` lets a member of org B rename org A's tag by guessing the uuid. `deleteTag` just below scopes with `orgId` via `findFirst`+`update` — mirror that here, and add a two-org test asserting a 403/no-op. Happy to pair on the compound-where quirk.

## Kata 2: "Speed up channel list"
Author intent: fewer awaits.
Fake diff: replaces `await Promise.all(map(...))` in `getIntegrationList` with a plain `for` loop of awaits "to reduce memory."
Resembles: [integrations.controller.ts L88-L124](../../../apps/backend/src/api/routes/integrations.controller.ts#L88-L124).
Findings — **Important**: this makes it *slower* (serial vs parallel) with no memory benefit at these sizes; if memory is the real concern, cap concurrency with `p-limit`, don't serialize. **Optional**: measure before optimizing.
Good comment:
> This trades parallel for serial — latency goes up, not memory down. If you're worried about fan-out at large channel counts, `p-limit(10)` gives you a bound without giving up concurrency. Do we have a number for how many channels a big org has?

## Kata 3: "Log posts for debugging"
Fake diff: adds `console.log(JSON.stringify(body))` in `createPost`.
Resembles: the *existing* [posts.controller.ts L176](../../../apps/backend/src/api/routes/posts.controller.ts#L176).
Findings — **Blocking**: logs full post content + media URLs (PII) to stdout on every create; in prod that's a privacy and log-volume problem. **Important**: use `Logger.debug` with ids only, behind a flag.
Good comment:
> Let's not log full bodies — that's user content and media URLs on every request. `Logger.debug({ postId, org: org.id })` gives us the trail without the PII. (There's an existing one in this file we should clean up too — ticket J1.)

## Kata 4: "Cache the calendar"
Fake diff: `useSWR('calendar', fetchCalendar)` where `fetchCalendar` closes over `startDate/customer`.
Resembles: [calendar.context.tsx](../../../apps/frontend/src/components/launches/calendar.context.tsx#L1-L50).
Findings — **Blocking**: constant key ignores date range + customer → switching weeks or customers serves stale/wrong data. **Important**: key must be `['calendar', startDate, endDate, customer]`.
Good comment:
> The SWR key is constant, so navigating to next week reuses this week's cache. Put every input that changes the result into the key array — `['calendar', startDate, endDate, customer]` — and SWR handles the rest.

## Kata 5: "Retry webhook delivery"
Fake diff: wraps the webhook `fetch` in a 3× loop.
Resembles: [sendWebhooks L321-L335](../../../apps/orchestrator/src/activities/post.activity.ts#L310-L336).
Findings — **Important**: this runs inside a Temporal activity that *already* retries — you now have retries×retries; and still no idempotency id, so receivers get duplicates. **Blocking (design)**: add a delivery id + signature *before* adding retries, or receivers can't dedupe. **Optional**: exponential backoff.
Good comment:
> Careful — this activity is already retried by Temporal, so a manual loop multiplies attempts. Before we add retries, receivers need a stable `X-Delivery-Id` and a signature to dedupe/verify; otherwise we're amplifying duplicate deliveries. Can we split this into "sign + id" first, "retry policy" second?

## Kata 6: "Fix the null crash in changeDate"
Fake diff: adds `getPostById(...)!` (non-null assertion) to silence the TS error.
Resembles: [posts.service.ts L849](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L843-L875).
Findings — **Blocking**: `!` hides the bug, doesn't fix it — a missing/foreign id still crashes at runtime. **Important**: throw `BadRequestException('Post not found')` like the sibling `changePostStatus` does. **Optional**: note the two siblings should share a guard.
Good comment:
> The `!` tells the compiler to trust us, but the runtime still gets `undefined.state` for a foreign id. `changePostStatus` right above throws a clean 400 on missing post — let's do the same and maybe extract a shared `getOwnedPostOrThrow`.

## Kata 7: "Encrypt tokens" (a good PR to review generously)
Fake diff: wraps `Integration.token` writes in `fixedEncryption`, adds a decrypt on read, plus a backfill script behind `ENCRYPT_TOKENS_BACKFILL`.
Resembles: R1 in [architecture critique](../03-architecture-and-patterns/06-architecture-critique.md); pattern from [third-party.repository.ts encryption].
Findings — **Important**: is decryption applied at *every* read site (grep all `.token` uses across providers + refresh service)? Missed site = broken channel. **Important**: backfill must be idempotent (don't double-encrypt). **Optional**: key rotation story.
Good comment:
> Great direction. Two things before merge: (1) let's enumerate every read of `integration.token` — providers, refresh service, analytics — to be sure decrypt is applied uniformly; (2) can the backfill detect already-encrypted rows so a re-run is safe? Love that it's flag-gated.

## Kata 8: "Add position column to threads"
Fake diff: adds `Post.position Int` and sorts by it, keeping `parentPostId` too.
Resembles: [pattern 10](../03-architecture-and-patterns/05-pattern-catalog.md), [createOrUpdatePost](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L488-L631).
Findings — **Important**: two sources of order (parentPostId chain + position) that can disagree — pick one or keep them in sync in one place. **Blocking**: `db push` migration with no backfill for existing threads → all get `position: 0`. **Important**: no transaction still.
Good comment:
> If we add `position`, let's retire the `parentPostId` ordering in the same PR — two orderings will drift. And existing rows need a backfill (they'll default to 0), which means a real migration, not just `db push`. Want to pair on the expand-contract steps?

## Kata 9: "Make login case-insensitive"
Fake diff: lowercases email in `routeAuth` login branch.
Resembles: [auth.service.ts L42-L94](../../../apps/backend/src/services/auth/auth.service.ts#L42-L94).
Findings — **Important**: correct direction (register already lowercases), but check `getUserByEmail` — does the unique index or query already fold case? If not, existing mixed-case rows may become unreachable or ambiguous. **Optional**: a data audit query in the PR description.
Good comment:
> Matches register's behavior, good. Before merge, can we confirm `getUserByEmail`/the unique constraint aren't already case-sensitive in a way that'd strand legacy rows? A quick `SELECT lower(email), count(*) ... having count(*)>1` in the description would settle it.

## Kata 10: "Remove the unused poke signal"
Fake diff: deletes both the `poke` signal definition and its `setHandler`.
Resembles: [post.workflow.v1.0.5.ts L48-L78](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L48-L78) + [signalWithStart poke](../../../apps/orchestrator/src/activities/post.activity.ts#L79-L84).
Findings — **Blocking**: the sweep still calls `signalWithStart(..., signal: 'poke')`; removing the handler breaks signal delivery to running workflows. **Important**: remove only the dead `poked` variable; keep the handler; document why.
Good comment:
> The `poked` variable is dead, agreed — but the handler must stay: `searchForMissingThreeHoursPosts` sends `poke` via `signalWithStart`, and removing the receiver would break that path for in-flight workflows. Let's drop just the variable and add a comment explaining the handler's contract.

---

## Grading your reviews

- **Blocking** = correctness/security/data-loss; must not merge (IDOR, PII, breaking running workflows, missing backfill).
- **Important** = will bite soon; fix before or right after (perf regressions, missing tests, drift).
- **Optional** = taste/polish.
Score yourself: did you catch every Blocking? Was each comment *specific* (named the line/alternative) and *kind* (assumed good intent, offered help)? Missing a Blocking on 3+ katas → re-read [03-fake-code-contrasts.md](03-fake-code-contrasts.md) and the [security checklist](../05-quality-engineering/05-security-checklist.md). Timed versions in [08-interview-prep/05](../08-interview-prep/05-debugging-and-code-review-rounds.md).

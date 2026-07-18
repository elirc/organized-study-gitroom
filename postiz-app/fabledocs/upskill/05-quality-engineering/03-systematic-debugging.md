# Systematic Debugging

Method first, then five repo-specific scenarios. Each scenario is also interview material — the "interview version" line tells you how to narrate it in a debugging round.

## The method

1. **Reproduce** — smallest deterministic trigger; if you can't reproduce, instrument first.
2. **Narrow** — binary-search the pipeline: which *layer* last saw correct data?
3. **Hypothesize** — one falsifiable cause at a time.
4. **Test cheaply** — logs/queries/one-off scripts before code changes.
5. **Fix the root cause** — not the symptom.
6. **Add regression coverage** — here that often means *creating* the first test (see [01-testing-strategy.md](01-testing-strategy.md)).

The tools this repo actually gives you:
- **Temporal Web UI** (`:8080` via [docker-compose.dev.yaml](../../../docker-compose.dev.yaml)) — per-workflow event history, activity failures with stack traces, search by `postId`/`organizationId` search attributes ([temporal.search.attribute.ts](../../../libraries/nestjs-libraries/src/temporal/temporal.search.attribute.ts)). This is the single highest-value debugging surface.
- **`Errors` table** — publish failures persist message+platform+raw body ([changeState L413-L425](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L393-L428)); admin UI at `/admin/errors` ([frontend route](../../../apps/frontend/src/app/(app)/(site)/admin/errors)).
- **Debug export** — superadmin `GET /posts/group/:group/debug-export` bundles the whole post group + errors into a re-importable JSON ([posts.controller.ts L148-L158](../../../apps/backend/src/api/routes/posts.controller.ts#L148-L158), [service L439-L484](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L439-L484)) — a purpose-built repro capsule; there's even an import modal ([import-debug-post.modal.tsx](../../../apps/frontend/src/components/launches/import-debug-post.modal.tsx)).
- Sentry (backend, orchestrator, frontend), Redis (`redisinsight` at `:5540`), pgadmin (`:8081`), Prisma query logging (`DATABASE_URL` + prisma log config).

---

## Scenario 1: "My post says QUEUE but never published, and there's no error."

Reproduction: schedule a post; stop the orchestrator (or Temporal) at the wrong moment; or have workflow start silently fail.
First question: is the bug in the **DB→Temporal handoff** or in **Temporal execution**?
Narrowing path:
1. Temporal UI: search `postId="<id>"`. Workflow exists?
2. **No workflow** → the fire-and-forget start failed ([posts.service.ts L722-L747](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L690-L748) swallows errors). Check backend logs around creation time; check Temporal connectivity env (`TEMPORAL_ADDRESS`).
3. **Workflow exists, still running** → open its history: is it parked on the `sleep` (publishDate in the future? timezone bug?) or retrying an activity (which one, what error)?
4. **Workflow completed but post is QUEUE** → read the history for early returns: subscription missing ([getPost activity returns false](../../../apps/orchestrator/src/activities/post.activity.ts#L113-L124)), `parentPostId` set ([getPostsList L142-L144](../../../apps/orchestrator/src/activities/post.activity.ts#L127-L147)).
5. Ask: did the safety net run? `RUN_CRON` set anywhere? ([infinite.workflow.register.ts L9](../../../libraries/nestjs-libraries/src/temporal/infinite.workflow.register.ts#L8-L19)) — if yes, a workflow *should* reappear within the hour, but only if the post is <2 days old ([repo query L45-L48](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L36-L64)).
Likely root causes: Temporal unreachable at creation; `RUN_CRON` unset; publishDate older than the 2-day sweep window.
Regression coverage: a test on `searchForMissingThreeHoursPosts` window logic; an alert on `workflow_start_failed` (ticket J13).
Senior lesson: fire-and-forget + compensation is fine *only* while the compensation actually runs — verify the janitor exists before blaming the mess.
Interview version: narrate the two-sources-of-truth diagnosis: "DB said QUEUE, scheduler had no record; the write path swallows start failures, so I checked the reconciliation job next…"

## Scenario 2: "Duplicate tweet — the same post published twice."

Reproduction: hard; simulate by making `updatePost` throw after a successful `postSocial`.
First question: which retry layer duplicated — Temporal's activity retry, or the in-workflow `iterate` loop?
Narrowing path:
1. Temporal UI history: count `postSocial` **attempts** (activity retries show as one activity with N attempts; the manual loop shows as N separate activity invocations).
2. One activity, 2 attempts → attempt 1 posted then timed out/threw before returning (look at `startToCloseTimeout` 10 min vs slow video upload — [proxy config L19-L29](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L19-L29)).
3. Two invocations → the manual loop retried a non-typed error ([catch fall-through: `continue` is implicit via loop](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L164-L269)); find the first attempt's error type.
4. Check platform-side: X rejects exact duplicates ('duplicate-rules' mapping — [x.provider.ts L47-L108](../../../libraries/nestjs-libraries/src/integrations/social/x.provider.ts#L47-L108)) — which platforms don't?
Likely root causes: success-then-crash between provider call and `updatePost` ([workflow L170-L200](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L164-L200)); network timeout after the provider accepted.
Regression: idempotency — persist `releaseId` as soon as the provider returns, and check it on retry entry.
Senior lesson: retries are a *duplication engine* unless the retried unit is idempotent; the boundary between "call succeeded" and "success recorded" is where duplicates are born.
Interview version: this is the canonical at-least-once story — tell it with the two retry layers named.

## Scenario 3: "Calendar shows the post at the wrong time for a user in Manila."

Reproduction: user timezone +8; schedule 09:00 local; compare calendar cell vs DB `publishDate` vs actual publish moment.
First question: is the value wrong **at rest** (DB), or wrong **at render**?
Narrowing path:
1. DB: `publishDate` should be UTC (backend forces `TZ=UTC` — [main.ts L10](../../../apps/backend/src/main.ts#L10)). Is it 01:00Z? Then rest is fine.
2. Render: calendar uses `newDayjs`/timezone helpers ([set.timezone](../../../apps/frontend/src/components/layout/set.timezone.tsx), used in [calendar.context.tsx](../../../apps/frontend/src/components/launches/calendar.context.tsx#L1-L50)) — does it apply the *user profile* offset (stored as an **int**, [schema User.timezone L91](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L79-L123)) or browser zone?
3. Submission: what exactly did the composer send in `date` ([CreatePostDto @IsDateString](../../../libraries/nestjs-libraries/src/dtos/posts/create.post.dto.ts#L110-L112))? An offset-less string is interpreted in… whose zone? (`dayjs(date)` server-side under TZ=UTC → UTC. So the *client* must convert before sending.)
4. `type: 'now'` special case: server generates `dayjs().format('YYYY-MM-DDTHH:mm:00')` ([posts.service.ts L780](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L777-L785)) — no offset, server clock.
Likely root causes: int-offset vs DST drift; a layer parsing an offset-less string in the wrong zone.
Regression: round-trip test: compose 09:00+08 → DB 01:00Z → calendar 09:00 → publishes at 01:00Z.
Senior lesson: fix timezone bugs by writing down the zone at every hop *first*; the bug is always at the hop nobody wrote down.
Interview version: draw the four hops (composer → API → DB → workflow) and the zone at each; interviewers want the *method*, not the answer.

## Scenario 4: "Channel keeps flipping to 'needs refresh' every few days."

Reproduction: connect a channel whose provider issues short-lived refresh tokens (or revoke access at the provider).
First question: is refresh **failing** (provider says no) or **never attempted**?
Narrowing path:
1. Find the refresh workflow started at connect time ([no.auth… L242-L246](../../../apps/backend/src/api/routes/no.auth.integrations.controller.ts#L242-L246), [refresh.token.workflow.ts](../../../apps/orchestrator/src/workflows/refresh.token.workflow.ts), [refresh.integration.service.ts](../../../libraries/nestjs-libraries/src/integrations/refresh.integration.service.ts)) in the Temporal UI.
2. Failing: activity error shows the provider response — expired refresh token vs revoked app vs clock skew.
3. Never attempted: was the workflow started at all? (fire-and-forget with `.catch(console.log)`).
4. On failure the code calls `setBetweenSteps` / sets `refreshNeeded` ([post.activity.ts refreshToken L362-L386](../../../apps/orchestrator/src/activities/post.activity.ts#L362-L386)) → posts then fail fast with "Refresh channel needed" ([workflow L113-L130](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L104-L130)).
5. Check `tokenExpiration` math at read paths ([posts.service.ts L100-L103](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L96-L123)): `dayjs(undefined).isBefore(now)` when expiration is null — what does that evaluate to? Test it in a REPL; that's a cheap-hypothesis probe.
Likely root causes: provider rotates refresh tokens on use and a failure path lost the new one; two processes refreshing concurrently (backend read path *and* orchestrator both refresh — race).
Regression: single-flight refresh (mutex per integration — `async-mutex` is already a dependency).
Senior lesson: token refresh is a distributed-mutation problem; two well-meaning refreshers can invalidate each other forever.
Interview version: "I found two components refreshing the same token independently" is a strong concurrency story — tell it with the fix (single-flight + persist-before-use).

## Scenario 5: "Self-hosted user reports: every API call returns 403 right after login."

Reproduction: fresh self-host, `NOT_SECURED` unset, frontend on http://, or user whose only org membership is disabled.
First question: is the JWT absent, invalid, or is the **org resolution** failing?
Narrowing path:
1. Devtools: does the `auth` cookie exist? Secure-flag cookies don't set over http ([removeAuth/cookie options mirror the flags](../../../apps/backend/src/services/auth/auth.middleware.ts#L11-L25)) — the classic self-host misconfig (`NOT_SECURED` exists for this).
2. Cookie present → decode the JWT (it's the whole user object) — `activated: false`? ([middleware L46-L48](../../../apps/backend/src/services/auth/auth.middleware.ts#L33-L48)).
3. Activated → org path: user's memberships all `disabled` → filtered list empty → `organization[0]` undefined → TypeError → catch → 403 ([L79-L102](../../../apps/backend/src/services/auth/auth.middleware.ts#L79-L104)). The 403 hides a crash — attach a debugger or add a temporary log in the catch.
4. Also check CORS: `FRONTEND_URL` mismatch blocks credentials ([main.ts L25-L47](../../../apps/backend/src/main.ts#L22-L48)).
Likely root causes: cookie flags vs http; env URL mismatch; disabled membership edge.
Regression: unit-test the middleware with an empty org list expecting a *specific* error, not a TypeError.
Senior lesson: a catch-all `throw new HttpForbiddenException()` around 70 lines converts every bug into "auth failed" — precision of failure is a feature.
Interview version: narrate the elimination order (transport → token → claims → org state → CORS) — interviewers grade the order, not the destination.

---

Convert scenarios 1, 2, and 5 into timed rounds in [08-interview-prep/05-debugging-and-code-review-rounds.md](../08-interview-prep/05-debugging-and-code-review-rounds.md).

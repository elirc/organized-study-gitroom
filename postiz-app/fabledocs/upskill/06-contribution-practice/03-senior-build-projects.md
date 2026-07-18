# Senior Build Projects

Six projects a maintainer might actually accept. Each: problem, value, design checklist, architecture decisions, likely files, migration/test/security/perf/rollout-rollback, open questions, stretch, and the STAR story it yields.

---

## S1: Encrypt social credentials at rest (2–4 days)
Problem: `Integration.token`/`refreshToken` are plaintext ([schema L322-L325](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L314-L356)); a DB leak = full account takeover. Third-party keys are already encrypted — the pattern exists ([fixedEncryption](../../../libraries/helpers/src/auth/auth.service.ts#L49-L55)).
Value: closes the highest-impact risk (R1).
Design checklist: enumerate every read of `.token`/`.refreshToken` (providers, [refresh service](../../../libraries/nestjs-libraries/src/integrations/refresh.integration.service.ts), [analytics/missing paths in posts.service](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L96-L229)); choose crypto (reuse fixed-IV AES vs upgrade to AES-GCM random-IV — argue it); dual-read during migration.
Architecture decisions: encrypt in the [integration.repository](../../../libraries/nestjs-libraries/src/database/prisma/integrations/integration.repository.ts) write path so callers don't change; a `decrypt-on-read` helper applied uniformly.
Likely files: integration.repository.ts, integration.service.ts, all providers' token usage, a backfill command in `apps/commands`.
Migration: idempotent backfill (detect already-encrypted); dual-read (try decrypt, fall back to plaintext) until backfill completes, then remove fallback.
Test: round-trip unit; a "no plaintext token in DB after backfill" integration check.
Security: key from env, not committed; consider rotation.
Perf: encryption per read is cheap; measure anyway.
Rollout/rollback: flag `ENCRYPT_TOKENS`; rollback = keep dual-read.
Open questions: key rotation story; do we re-encrypt on JWT_SECRET change?
STAR: "encrypted crown-jewel secrets with a zero-downtime, reversible backfill across 30 provider integrations."

## S2: Transactions + integrity for post/thread writes (2–3 days)
Problem: `createOrUpdatePost` does N writes with no transaction ([L488-L631](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L488-L631)); crash → partial thread.
Value: protects the "a thread is all-or-nothing" invariant (R10).
Design checklist: wrap the upsert loop + tag rewrite + group soft-delete in `prisma.$transaction`; keep the workflow start *outside* (after commit — [service L777-L798](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L777-L798)); handle the interactive-transaction timeout for long threads.
Architecture decisions: interactive transaction vs batched; isolation level; what happens to media uploads inside (they're external — keep them out).
Test: simulate a mid-loop failure, assert no partial rows.
Rollout/rollback: additive; revert = remove the wrapper.
Open questions: transaction timeout for 100-item threads; do we need SELECT FOR UPDATE anywhere?
STAR: "added transactional integrity to the core write path; reasoned about which side effects must stay outside the transaction."

## S3: Transactional outbox for workflow scheduling (1–2 weeks)
Problem: workflow start is fire-and-forget with a swallowed error ([service L790-L798](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L790-L798)); the compensating sweep is a 2-day scan, not a guarantee.
Value: no lost scheduling without a time-bounded janitor (R7-adjacent); the textbook pattern.
Design checklist: `Outbox` table written in the same transaction as the post (builds on S2); a dispatcher (a Temporal workflow or a small poller) that drains outbox → starts workflows → marks sent; idempotency via the existing deterministic `workflowId`.
Architecture decisions: dispatcher as Temporal workflow vs cron; retention; ordering.
Likely files: schema, posts service/repository, a new dispatcher in orchestrator, retire/keep the missing-post sweep as a backstop.
Migration: run outbox + old sweep in parallel first; measure sweep-repair count drop to ~0; then reduce the sweep's role.
Test: kill the dispatcher mid-drain, assert eventual start; Temporal test env.
Rollout/rollback: dispatcher behind a flag; sweep stays as safety net during rollout.
Open questions: do we keep the sweep forever as defense-in-depth? (Probably yes.)
STAR: "replaced a best-effort scheduler handoff with a transactional outbox; the sweep's repair count going to zero was the proof."

## S4: Webhook delivery pipeline — sign, retry, log (1 week)
Problem: outgoing webhooks are unsigned, un-retried, unlogged ([activity L310-L336](../../../apps/orchestrator/src/activities/post.activity.ts#L310-L336)) (R9).
Value: receivers can verify + we can prove delivery.
Design checklist: HMAC signing (M1) + `WebhookDelivery` table (M8) + a delivery activity with Temporal retry/backoff + dead-letter + UI. Idempotency id so replays dedupe.
Architecture decisions: one delivery workflow per webhook vs a batched activity; backoff schedule; retention.
Test: delivery success/failure/dead-letter paths in Temporal test env.
Rollout/rollback: sign first (additive), then logging, then retry — three PRs.
STAR: "turned fire-and-forget webhooks into a signed, retriable, observable delivery pipeline."

## S5: Introduce the testing + CI foundation (1–2 weeks)
Problem: zero tests, broken jest wiring (R2).
Value: de-risks every future change; enables PR gating.
Design checklist: working jest configs per package (J17); Testcontainers Postgres for repo tests; `@temporalio/testing` for workflows; seed the pure cores, the IDOR fix, the refresh-retry workflow; wire CI (the [Jenkins](../../../Jenkins)/[sonar](../../../sonar-project.properties) hints suggest external CI to integrate with).
Architecture decisions: what to gate on (start advisory, then required); coverage target as a *floor on new code*, not a repo-wide mandate.
Rollout: advisory CI → fix flakes → required on changed files.
STAR: "introduced testing to a 100k-line untested codebase; chose zero-mock seams first and gated CI incrementally to avoid a wall of red."

## S6: Provider plugin architecture (2–4 weeks)
Problem: 30+ providers in one array with an ever-growing optional-member interface ([integration.manager L40-L75](../../../libraries/nestjs-libraries/src/integrations/integration.manager.ts#L40-L75), [social.integrations.interface.ts](../../../libraries/nestjs-libraries/src/integrations/social/social.integrations.interface.ts)); adding networks strains it, and the registry `!` lookup ([L170-L172](../../../libraries/nestjs-libraries/src/integrations/integration.manager.ts#L170-L172)) fails ugly.
Value: safer extensibility; conformance testing per provider; graceful unknown-provider handling.
Design checklist: a capability-declaration interface (what each provider supports) replacing runtime `!!provider.method` probes; a conformance test suite every provider must pass; a NullProvider for unknown identifiers; optional per-provider packages.
Architecture decisions: keep the registry array vs move to DI-discovered providers; how metadata (`@Plug`, `@Rules`) fits.
Migration: introduce the interface alongside; migrate providers incrementally; the conformance suite catches regressions.
Test: the conformance suite *is* the test.
Rollout/rollback: additive interface; providers opt in.
Open questions: do third-party/community providers become a real plugin API? (product decision).
STAR: "redesigned a 30-provider integration layer for safe extensibility with a conformance-test gate."

---

Pick projects by what you want to *talk about*: S1/S2/S3 are distributed-systems and data-integrity stories (strongest for backend interviews); S4 is reliability; S5 is engineering-maturity/leadership; S6 is API design. Do the design note first, always — a maintainer judges the plan before the diff.

# Architecture Critique

Written as if you're presenting to the team after two weeks in the code. Doubles as system-design interview material — cross-referenced from [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md). Confirmed issues cite exact lines; hypotheses are labeled *investigate*.

## Strongest design choices (defend these in an interview)

1. **Temporal for scheduling.** Durable timers survive restarts/deploys; retries and state are first-class; rescheduling is terminate+restart. The single best decision — it makes "a post scheduled for Friday survives Saturday's deploy" true by construction ([workflow sleep L96-L102](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L96-L102), [startWorkflow L690-L748](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L690-L748)).
2. **Provider abstraction + typed failure taxonomy.** 30+ networks behind one interface, with a three-verdict error contract (`RefreshToken`/`BadBody`/retry) that keeps platform-specific string-matching quarantined ([social.abstract.ts](../../../libraries/nestjs-libraries/src/integrations/social.abstract.ts#L5-L173)). Adding a network is a bounded task.
3. **Shared domain library.** Service+repository pairs in `libraries/` run identically in backend and orchestrator — one PostsService, two processes ([post.activity.ts L57-L66](../../../apps/orchestrator/src/activities/post.activity.ts#L54-L66)). Reuse without a network hop.
4. **Determinism boundary enforced by tooling.** The workflow/activity split is the one boundary that *can't* be violated by a careless PR, because Temporal rejects it. Tool-enforced > convention-enforced.
5. **Reconciliation mindset.** The missing-post sweep shows the team thinks in "what repairs the drift?" terms ([missing.post.workflow.ts](../../../apps/orchestrator/src/workflows/missing.post.workflow.ts)).

## Risks and trade-offs (prioritized)

| # | Risk | Evidence | Impact | Confidence |
| --- | --- | --- | --- | --- |
| R1 | **Social OAuth tokens stored plaintext** | [schema L322-L325](../../../libraries/nestjs-libraries/src/database/prisma/schema.prisma#L314-L356); no encryption in [integration.repository.ts](../../../libraries/nestjs-libraries/src/database/prisma/integrations/integration.repository.ts) (contrast third-party apiKeys which *are* encrypted) | DB dump = full control of users' social accounts | Confirmed |
| R2 | **No tests at all** | zero spec files; [jest.config.ts](../../../jest.config.ts) references missing `@nx/jest` | every change is unverified; refactors are scary | Confirmed |
| R3 | **Tenant isolation by convention** | [editTag IDOR L804-L814](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L804-L814) | cross-org data access when a filter is forgotten | Confirmed (one instance), *investigate* others |
| R4 | **No DB migrations; auto-push on deploy** | [package.json L16-L18, L33-L36](../../../package.json#L16-L36) | silent data loss, no rollback, no review | Confirmed |
| R5 | **Fail-open auth boundary** | [authenticatedController list](../../../apps/backend/src/api/api.module.ts#L48-L109) | forgetting to list a controller ships it unauthenticated | Confirmed (design) |
| R6 | **JWT never expires, carries whole user** | [signJWT L42-L44](../../../libraries/helpers/src/auth/auth.service.ts#L42-L44) | no revocation; stale privilege claims | Confirmed |
| R7 | **Non-idempotent provider posting under retry** | [workflow retry L164-L269](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L164-L269) | duplicate posts on timeout-after-success | Confirmed (logic); frequency *investigate* |
| R8 | **Unbounded recurrence expansion** | [getPosts while-loop L204-L216](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L199-L219) | slow query / memory for old daily recurring posts over a wide range | Confirmed (no cap); real-world trigger *investigate* |
| R9 | **Webhooks unsigned, no retry, swallowed errors** | [sendWebhooks L321-L335](../../../apps/orchestrator/src/activities/post.activity.ts#L310-L336) | receivers can't verify sender; silent loss; duplicates | Confirmed |
| R10 | **Missing transactions around multi-row writes** | [createOrUpdatePost L488-L631](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L488-L631) | partial threads on crash | Confirmed |
| R11 | **No transaction, and per-request counting queries in authz** | [permissions.service L83-L122](../../../apps/backend/src/services/auth/permissions/permissions.service.ts#L83-L122) | latency on every gated route at scale | Confirmed (cost), impact *investigate* |
| R12 | **Concurrent token refresh race** | read-path ([posts.service L100-L123](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.service.ts#L96-L123)) + workflow both refresh | rotating-token providers self-invalidate | *Investigate* |

## What I'd change owning this for 3 months

**Month 1 — safety floor.** (a) Encrypt social tokens at rest, reusing the existing `fixedEncryption` (R1). (b) Bootstrap tests (ticket J17) and cover the pure cores + the IDOR fix (R2/R3). (c) Fix editTag and run the full IDOR audit (R3). Migration path: encryption needs a one-time backfill script (read plaintext → write ciphertext) gated behind a flag; test strategy: round-trip unit test + a staging backfill dry-run.

**Month 2 — reliability & structure.** (d) Introduce `prisma migrate`, stop auto-push (R4); expand-contract discipline. (e) Transactional outbox for workflow starts (R7-adjacent) or at minimum wrap `createOrUpdatePost` in `$transaction` (R10). (f) Sign webhooks + add retry/delivery log (R9). Each ships behind its own PR with a rollback (feature-flag the outbox dispatcher; migrations are reversible).

**Month 3 — hardening.** (g) Structural tenant isolation: a Prisma client extension injecting `organizationId`, or Postgres RLS as a backstop (R3 permanently). (h) Short-lived JWT + refresh rotation (R6). (i) Cap recurrence expansion + add index (R8, J16). (j) Single-flight token refresh via `async-mutex` (R12).

Sequencing rationale: secrets and IDOR first (largest blast radius, smallest change), tests early (they de-risk everything after), structural fixes last (largest blast radius to *change*, so do them once the safety net exists).

## System-design framing (for interviews)

Present the above as: "Here's a real scheduler. Its bet is durable workflows — correct and defensible. Its debts are secrets-at-rest, tenant isolation by convention, and no test/migration safety net. If I owned it, I'd buy down risk in blast-radius order." That structure — *strengths, debts, prioritized paydown with migration+rollback+test per item* — is exactly what a senior system-design answer sounds like. Full walkthrough: [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md).

Drill: pick any two risks and write the RFC opening (problem, blast radius, options, recommendation) using the template in [07-career-and-collaboration/02-writing-prs-and-rfcs.md](../07-career-and-collaboration/02-writing-prs-and-rfcs.md).

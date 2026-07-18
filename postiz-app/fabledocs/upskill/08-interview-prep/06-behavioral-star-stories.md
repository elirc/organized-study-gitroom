# Behavioral STAR Stories

Ten worksheets. Each is sourced from real work you can *actually do* in this repo (module 06) or from studying it — so your stories are true, not invented. Fill Situation/Task/Action/Result in your own words; the scaffolding and evidence anchors are provided. Keep each under 2 minutes, end on impact.

Prompt map (which story answers which common question):

| Prompt | Best stories |
| --- | --- |
| "Complex codebase you learned" | 1 |
| "Bug you found/fixed" | 2, 3 |
| "Technical trade-off" | 4, 5 |
| "Disagreement/conflict" | 6 |
| "Mistake you made" | 7 |
| "Ambiguity / no clear answer" | 8 |
| "Improved quality/process" | 9 |
| "Leadership/influence without authority" | 10 |

---

## Story 1: Learning a large unfamiliar codebase
Prompts: "walk me through a complex system you learned."
Source: studying Postiz (this curriculum).
Situation: inherited/joined a 100k-line social-scheduling monorepo — NestJS API, Temporal worker, Next frontend, 30+ integrations, no tests, no docs beyond a partly-stale CLAUDE.md.
Task: become productive without breaking a multi-tenant production system.
Action: mapped the system top-down (processes → flows), traced the core flow end-to-end (schedule → publish through Temporal), built a glossary to untangle channel/integration/provider, and validated understanding by tracing failure paths.
Result: could explain the scheduling architecture and its two-sources-of-truth risk within days; identified a real IDOR while reading.
Evidence: the [system map](../01-codebase-cartography/01-system-map.md), [key flows](../01-codebase-cartography/05-key-flows.md).
Resume bullet: "Ramped on a 100k-line untested monorepo and mapped its durable-scheduling architecture, surfacing a cross-tenant data bug during onboarding."

## Story 2: Finding and fixing a security bug (IDOR)
Prompts: "a bug you're proud of finding."
Source: ticket J2.
S: while reading tag CRUD, noticed `editTag` updated by id with no org scope while its sibling `deleteTag` scoped correctly.
T: confirm and fix without breaking the intended behavior.
A: reproduced with two orgs (tag from A, edit from B), fixed the where-clause to match the sibling pattern, added the cross-org regression test, and audited siblings for the same class.
R: closed a cross-tenant write; left a test guarding it; flagged the class for structural fix.
Evidence: [editTag L804-L814](../../../libraries/nestjs-libraries/src/database/prisma/posts/posts.repository.ts#L804-L814).
Bullet: "Found and fixed an IDOR allowing cross-org writes; added regression coverage and audited the query class."

## Story 3: Debugging a hard, intermittent issue (duplicate posts)
Prompts: "a tricky bug."
Source: debugging scenario 2 / ticket M6.
S: users occasionally saw the same post published twice.
T: find why a single scheduled post duplicated.
A: used Temporal's history to see one activity with two attempts; found attempt 1 posted then timed out before recording the remote id, so the retry re-posted; traced it to non-idempotent posting.
R: proposed persisting the release id immediately + checking on retry; the canonical at-least-once fix.
Evidence: [workflow retry L164-L200](../../../apps/orchestrator/src/workflows/post-workflows/post.workflow.v1.0.5.ts#L164-L200).
Bullet: "Diagnosed duplicate publishing to a retry-after-partial-success window; designed an idempotency-key fix."

## Story 4: A technical trade-off you weighed
Prompts: "a time you chose between approaches."
Source: senior project S3 / M5.
S: scheduling had a fire-and-forget handoff repaired by a 2-day scan.
T: decide between a minimal fix and a transactional outbox.
A: weighed effort vs guarantee — the outbox eliminates lost starts but adds a dispatcher and table; recommended it but kept the sweep as a backstop and rolled out alongside, using the sweep's repair count dropping to zero as proof.
R: a durable guarantee without ripping out the safety net.
Evidence: [side effects doc](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md).
Bullet: "Chose a transactional outbox over a quick fix, de-risking rollout by running it alongside the existing reconciliation job."

## Story 5: Balancing pragmatism vs correctness
Prompts: "when you shipped something imperfect / when you pushed for quality."
Source: testing strategy (S5).
S: the repo had zero tests and broken jest wiring.
T: introduce testing without stalling delivery.
A: didn't mandate coverage; fixed the runner, tested the zero-mock pure cores first, then introduced tests via a real bug fix (red→green), then wired CI advisory-first.
R: a growing safety net the team would actually keep, without a wall of red.
Evidence: [testing strategy](../05-quality-engineering/01-testing-strategy.md).
Bullet: "Bootstrapped testing in an untested codebase with a pragmatic, risk-prioritized rollout."

## Story 6: Disagreeing in a code review
Prompts: "a disagreement with a colleague."
Source: review kata 5 (double-retry webhook PR).
S: a teammate added a manual retry loop inside an already-retried activity.
T: prevent duplicate deliveries without discouraging them.
A: separated fact (this multiplies attempts and isn't idempotent) from preference; asked about intent; offered the sequenced fix (sign + delivery id first, retry policy second); once aligned, supported the plan.
R: a better design and a good working relationship.
Evidence: [review katas](../04-code-reading-gym/04-review-katas.md).
Bullet: "Turned a review disagreement into a better webhook-delivery design through evidence and a concrete alternative."

## Story 7: A mistake and what you learned
Prompts: "a mistake you made."
Source: any schema experiment (fill with your real one) — the repo's `db push --accept-data-loss` makes this authentic.
S: ran a schema change with the repo's `db push` and lost a column's data in a dev/staging DB.
T: recover and prevent recurrence.
A: restored from backup, then proposed adopting `prisma migrate` with reviewable, reversible migrations and stopping auto-push on deploy.
R: turned a self-inflicted data loss into a process improvement.
Evidence: [data model doc](../03-architecture-and-patterns/02-data-model-and-persistence.md).
Bullet: "After a destructive `db push`, drove adoption of reviewable migrations to prevent data-loss on deploy."

## Story 8: Working under ambiguity
Prompts: "no clear right answer."
Source: ticket J9 (the soft-delete filter you can argue either way).
S: a query missing a `deletedAt` filter — but fixing it might be wrong (resolving a link from a published-then-deleted post could be intended).
T: decide with incomplete information.
A: laid out both interpretations, checked call sites and product intent, wrote the reasoning in the PR, and proposed the lower-risk option with a flag.
R: a decision defensible either way, documented for the next person.
Bullet: "Resolved an ambiguous correctness question by documenting both interpretations and choosing the reversible option."

## Story 9: Improving quality/observability
Prompts: "made something more reliable/visible."
Source: ticket J13 / project S4.
S: workflow-start failures were silently swallowed; webhook delivery was a black hole.
T: make hidden failures visible.
A: added a metric + Sentry breadcrumb on start failure; designed a webhook delivery log with success-rate metric and an alert on a *leading* indicator (sweep-repair count).
R: failures became detectable before users reported them.
Evidence: [observability doc](../05-quality-engineering/06-observability-and-operations.md).
Bullet: "Instrumented silent failure paths, adding leading-indicator alerts that surfaced issues before user reports."

## Story 10: Influence without authority
Prompts: "led without being the lead."
Source: proposing a structural fix (M2) in an OSS project.
S: recurring tenant-isolation bugs from convention-based scoping.
T: get a cross-cutting change accepted by maintainers you don't manage.
A: wrote an RFC comparing options (client extension vs RLS vs lint), led with the user-facing risk and evidence, addressed the intentionally-cross-org queries, and asked for direction before coding.
R: alignment on a structural fix over whack-a-mole.
Evidence: [maintainer communication](../07-career-and-collaboration/03-maintainer-communication.md).
Bullet: "Drove consensus on structural tenant-isolation enforcement via an evidence-led RFC in an OSS project."

---

Rehearsal check for each: under 2 minutes? one concrete detail (a line, a metric, a tool)? ends with impact, not activity? If a story is "we did X," rewrite until it's "I did X, which caused Y." Interviewers hire the person in the story, so keep yourself the subject of the verbs — while crediting the team honestly.
